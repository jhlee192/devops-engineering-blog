---
title: "Nginx limit_except로 컨테이너 레지스트리 push를 특정 IP만 허용한 기록"
date: 2026-11-02
tags: ["Nginx", "limit_except", "Container Registry", "skopeo", "cri-o"]
status: done
---

컨테이너 이미지 레지스트리 앞단 Nginx에 **GET을 제외한 모든 메서드(POST, PUT, PATCH 등)를 차단**하는 설정을 넣었다. pull은 어디서든 되어야 하고, push는 지정한 서버에서만 되어야 한다.

## 환경

- 레지스트리는 Nginx 리버스 프록시 뒤에 있다. Nginx가 TLS를 종료하고 `proxy_pass`로 레지스트리에 넘긴다.
- push 클라이언트는 `skopeo`, pull 클라이언트는 쿠버네티스 노드의 `cri-o`다.

## push/pull 때 들어오는 메서드

설정을 넣기 전에 push와 pull을 각각 해 보고 Nginx access 로그에 어떤 메서드가 찍히는지 확인했다.

**push (skopeo)**

```text
198.51.100.10 - - [02/Nov/2026:11:31:09 +0900] "GET /v2/ HTTP/1.1" 401 87 "" "skopeo/1.22.2" -
198.51.100.10 - - [02/Nov/2026:11:31:09 +0900] "HEAD /v2/team/alpine/blobs/sha256:fd2758d7... HTTP/1.1" 404 0 "" "skopeo/1.22.2" -
198.51.100.10 - - [02/Nov/2026:11:31:09 +0900] "HEAD /v2/team/alpine/blobs/sha256:cd17896e... HTTP/1.1" 404 0 "" "skopeo/1.22.2" -
198.51.100.10 - - [02/Nov/2026:11:31:09 +0900] "HEAD /v2/team/alpine/blobs/sha256:24a707ea... HTTP/1.1" 200 0 "" "skopeo/1.22.2" -
198.51.100.10 - - [02/Nov/2026:11:31:09 +0900] "HEAD /v2/team/alpine/blobs/sha256:fe07684b... HTTP/1.1" 200 0 "" "skopeo/1.22.2" -
198.51.100.10 - - [02/Nov/2026:11:31:09 +0900] "HEAD /v2/team/alpine/blobs/sha256:bea4bd83... HTTP/1.1" 200 0 "" "skopeo/1.22.2" -
198.51.100.10 - - [02/Nov/2026:11:31:09 +0900] "PUT /v2/team/alpine/manifests/cron HTTP/1.1" 201 0 "" "skopeo/1.22.2" -
```

**pull (cri-o)**

```text
198.51.100.10 - - [02/Nov/2026:11:31:41 +0900] "GET /v2/ HTTP/1.1" 401 87 "" "cri-o/1.31.13 go/go1.22.5 os/linux arch/amd64" -
198.51.100.10 - - [02/Nov/2026:11:31:41 +0900] "GET /v2/team/alpine/manifests/3.19.4 HTTP/1.1" 200 739 "" "cri-o/1.31.13 go/go1.22.5 os/linux arch/amd64" -
198.51.100.10 - - [02/Nov/2026:11:31:41 +0900] "GET /v2/team/alpine/blobs/sha256:dc580219... HTTP/1.1" 200 1442 "" "cri-o/1.31.13 go/go1.22.5 os/linux arch/amd64" -
```

pull은 GET만 쓴다. push는 HEAD로 blob이 이미 있는지 확인하고 PUT으로 manifest(태그)를 올린다.

이 예시의 push는 레이어가 이미 레지스트리에 있어서 업로드(POST/PATCH) 없이 **HEAD + PUT만으로 태그가 바뀌었다.** POST만 막으면 새 레이어 업로드는 막히지만, 이미 있는 레이어로 기존 태그를 덮어쓰는 건 막을 수 없다는 뜻이다. 그래서 메서드를 하나씩 고르지 않고 GET 외의 메서드를 전부 막기로 했다.

## limit_except

Nginx의 `limit_except`는 **지정한 메서드를 제외한 나머지 메서드에만** 블록 안의 접근 제어를 적용한다.

```nginx
limit_except GET {
    ...
}
```

- `GET`을 지정하면 **HEAD도 함께 제외**된다. 위 로그처럼 HEAD는 push에서도 쓰이지만 조회만 하는 메서드라서, GET과 같이 허용돼도 상관없다.
- 블록 안의 `allow`/`deny`는 **위에서부터 비교해서 처음 일치한 규칙 하나만 적용**된다. 그래서 좁은 규칙(특정 IP)을 위에, `all`을 맨 아래에 둔다.
- `allow`/`deny`는 한 줄에 주소 하나만 받는다. IP가 여러 개면 줄을 나눠 쓰고, 대역은 CIDR(`192.0.2.0/24`)로 쓸 수 있다.

설정은 레지스트리로 `proxy_pass`하는 `location` 블록 안에 넣는다.

## 첫 테스트: 막았는데 push가 됐다

처음에는 push를 하던 서버의 IP 하나만 막아 보려고 이렇게 넣었다.

```nginx
# image modulation deny
limit_except GET {
    allow all;
    deny 198.51.100.10;
}
```

설정을 반영하고 그 서버에서 push와 pull을 다 해 봤는데, 둘 다 정상이었다. 로그에도 PUT이 그대로 201로 찍혔다.

```text
198.51.100.10 - - [02/Nov/2026:11:35:45 +0900] "PUT /v2/team/alpine/manifests/cron HTTP/1.1" 201 0 "" "skopeo/1.22.2" -
```

원인은 순서였다. `allow`/`deny`는 위에서부터 비교하다가 **처음 일치한 규칙 하나만 적용**한다. `allow all`이 맨 위에 있으니 모든 IP가 첫 줄에서 허용으로 끝났고, 그 아래 `deny`까지는 내려가지 않았다. 특정 IP만 막으려면 `deny IP`를 위에, `allow all`을 아래에 써야 한다.

## 테스트 설정: 검수 서버만 차단

순서를 바로잡아서 **테스트용 클라이언트 한 대만 쓰기 메서드를 막는** 설정으로 다시 테스트했다. 운영 설정을 바로 넣으면 막히는지 확인하기 어려워서 이 단계를 먼저 거쳤다.

```nginx
location / {
    # image modulation deny (test)
    limit_except GET {
        deny 192.0.2.103;    # 테스트 클라이언트
        allow all;
    }

    proxy_pass http://localhost:5000;
    ...
}
```

검수 서버에서 push를 하면 막히고, pull은 그대로 되는지 본다.

## 운영 설정: 지정한 서버만 push 허용

테스트가 끝나면 반대로, **push를 허용할 서버만 `allow`하고 나머지는 전부 막는다.**

```nginx
location / {
    # image modulation deny
    limit_except GET {
        allow 192.0.2.11;
        allow 192.0.2.12;
        allow 192.0.2.13;
        deny  all;
    }

    proxy_pass http://localhost:5000;
    ...
}
```

| 메서드 | 허용 IP | 그 외 |
|---|---|---|
| GET, HEAD (pull) | 허용 | 허용 |
| POST, PUT, PATCH, DELETE 등 (push·삭제) | 허용 | 403 |

`allow`에 넣는 IP는 **Nginx access 로그 첫 칸에 찍히는 IP**여야 한다. 서버가 내부망으로 붙는지, NAT를 거쳐 공인 IP로 붙는지에 따라 찍히는 IP가 달라서, 적용 전에 각 서버에서 요청을 보내 로그로 확인했다.

## 설정 체크와 적용

```bash
nginx -t -c /etc/nginx/nginx.conf
```

- `-t`: 설정을 적용하지 않고 문법만 검사한다.
- `-c`: 검사할 메인 설정 파일을 지정한다. 생략하면 Nginx 빌드 시 지정된 기본 경로를 검사하므로, 기본 경로가 아닌 설정을 쓰고 있다면 꼭 붙여야 한다. 출력에 나온 경로가 실제 쓰는 설정 파일인지 확인한다.

```bash
nginx -s reload
```

- `-s reload`: 마스터 프로세스에 설정을 다시 읽으라는 신호를 보낸다. 기존 연결은 끊지 않고 마무리한 뒤 새 설정으로 바뀐다. 재기동이 아니라서 push/pull 중인 연결이 있어도 끊기지 않는다.
- systemd로 관리한다면 `systemctl reload nginx`도 같은 동작이다.

## 참고: GitLab 번들 Nginx인 경우

GitLab Omnibus에 포함된 Nginx를 사용하는 경우에도 같은 원리로 적용할 수 있다. 레지스트리 server 블록은 GitLab이 자동으로 만든 아래 파일에 있고, 그 안의 `location /`에 위 설정을 넣었다.

```text
/var/opt/gitlab/nginx/conf/gitlab-registry.conf
```

설정 체크는 번들 Nginx 바이너리로 하는데, `-c`를 빼먹으면 엉뚱한 파일을 검사한다.

```bash
/opt/gitlab/embedded/sbin/nginx -t -c /var/opt/gitlab/nginx/conf/nginx.conf
```

- `-c` 없이 실행했더니 `/opt/gitlab/embedded/conf/nginx.conf`를 검사하고 `syntax is ok`가 나왔다. 이 파일은 패키지 기본 설정이라 실제로는 쓰이지 않는다. GitLab이 쓰는 설정은 `/var/opt/gitlab/nginx/conf/nginx.conf`이고, 출력에 이 경로가 나와야 제대로 검사한 것이다.

적용은 재기동 없이 HUP 신호로 했다.

```bash
gitlab-ctl hup nginx
```

- `hup`: Nginx에 HUP 신호를 보내 설정만 다시 읽게 한다. `nginx -s reload`와 같은 동작이다.

이 파일은 GitLab이 생성하는 파일이라 **`gitlab-ctl reconfigure`나 GitLab 업그레이드 때 다시 만들어지면서 직접 넣은 설정이 사라진다.** 작업 후 아래 명령으로 설정이 남아 있는지 확인한다.

```bash
grep -c limit_except /var/opt/gitlab/nginx/conf/gitlab-registry.conf
```

- `-c`: 일치하는 줄 수를 센다. `0`이면 설정이 사라진 것이므로 다시 넣는다.
