---
title: "Docker로 띄운 GitLab의 로그를 영구 보관하도록 바꾼 기록"
date: 2026-10-27
tags: ["GitLab", "Docker Compose", "logrotate", "svlogd", "로그"]
status: done
---

GitLab 서비스 로그를 영구 보관해야 하는 요구가 생겼다. 그동안 로그 설정은 한 번도 건드린 적이 없어서, 지금 상태부터 확인했다.

## 환경

- GitLab CE 18.x, Docker Compose 기반 단일 컨테이너 구성
- `gitlab.rb`를 직접 수정하지 않고 compose의 `GITLAB_OMNIBUS_CONFIG` 환경변수로 설정을 관리

## 기본 설정으로는 30개까지만 남는다

컨테이너 안의 `/etc/gitlab/gitlab.rb`에서 logging 부분을 보면 전부 주석 처리된 기본값이다.

```ruby
# logging['svlogd_size'] = 200 * 1024 * 1024 # rotate after 200 MB of log data
# logging['svlogd_num'] = 30 # keep 30 rotated log files
# logging['svlogd_timeout'] = 24 * 60 * 60 # rotate after 24 hours
# logging['svlogd_filter'] = "gzip" # compress logs with gzip
...
# logging['logrotate_frequency'] = "daily" # rotate logs daily
# logging['logrotate_rotate'] = 30 # keep 30 rotated logs
# logging['logrotate_compress'] = "compress" # see 'man logrotate'
# logging['logrotate_method'] = "copytruncate" # see 'man logrotate'
# logging['logrotate_dateformat'] = nil # use date extensions for rotated files rather than numbers ...
```

`svlogd_*`와 `logrotate_*`가 따로 있다. GitLab Omnibus는 로그를 **두 가지 방식으로 나눠서 정리**하기 때문이다.

| 방식 | 대상 | 기본 보관 |
|---|---|---|
| svlogd (runit) | `/var/log/gitlab/<서비스>/current`. gitaly, sidekiq, puma, workhorse, postgresql, redis 등 | 30개 |
| logrotate | `gitlab-rails/*.log`(production_json, api_json, audit_json 등), nginx, gitlab-shell 등 | 30개 |

둘 다 하루에 한 번 로테이트하므로 기본값이면 약 한 달치만 남는다. 한쪽만 바꾸면 나머지 로그는 계속 지워지니 **양쪽 다** 손봐야 했다.

## 바꾼 설정

```ruby
logging['svlogd_num'] = 0
logging['logrotate_rotate'] = -1
logging['logrotate_dateformat'] = "-%Y-%m-%d"
```

로테이트 주기(하루), gzip 압축, `copytruncate` 방식은 기본값 그대로 두었다.

### svlogd: 0이 "지우지 않음"

svlogd의 `n` 값은 남겨둘 옛 파일 개수다. 0으로 두면 옛 파일을 지우지 않는다. 로테이트된 파일 이름은 `@4000000069...s` 같은 TAI64N 타임스탬프라 기존 파일 이름이 바뀌지 않는다. 그래서 파일이 계속 쌓여도 로테이트할 때 부담은 없다.

### logrotate: 0이 아니라 -1

여기가 헷갈리는 지점이다. logrotate의 `rotate`는 **0이면 옛 로그를 보관하지 않고 바로 지운다.** svlogd와 같은 생각으로 0을 넣으면 정반대가 된다. 지우지 않으려면 `-1`을 넣어야 한다.

`-1`은 logrotate 버전에 따라 지원하지 않을 수 있다. GitLab은 자체 logrotate를 번들하고 있으니 그 버전을 확인했다.

```bash
docker exec gitlab /opt/gitlab/embedded/sbin/logrotate --version
```

확인한 번들 logrotate 버전은 `-1` 설정을 지원했다.

### dateformat: 번호 대신 날짜

`logrotate_dateformat`은 기본값이 `nil`이다. 이 상태에서는 로테이트된 파일에 번호가 붙는다.

```
production_json.log.1.gz   ← 어제
production_json.log.2.gz   ← 그저께
```

번호 방식에서는 로테이트할 때마다 **기존 파일 전부의 번호가 하나씩 밀린다.** 30개일 때는 문제가 안 되지만, 영구 보관이면 파일이 몇 천 개가 되고 매일 그만큼 rename이 일어난다. 파일 이름만 보고 날짜를 알 수도 없다.

값을 넣으면 logrotate의 `dateext`가 켜지고, 지정한 형식이 파일 이름 뒤에 붙는다.

```
production_json.log-YYYY-MM-DD.gz
```

한 번 만들어진 파일은 다시 이름이 바뀌지 않는다. 다만 날짜 단위라서 같은 날 두 번 로테이트되면 이름이 겹친다. 크기 기준 로테이트(`logrotate_maxsize`, `logrotate_size`)는 기본값 `nil` 그대로라 하루 한 번만 돌기 때문에 이 형식으로 충분했다.

## 하는 김에 주석 정리

기존 `GITLAB_OMNIBUS_CONFIG`는 설정만 나열돼 있어서, 왜 이 값을 넣었는지 남아 있지 않았다. 로그 설정을 추가하면서 서비스별로 묶고 주석을 달았다.

블록 스칼라(`|`) 안의 내용은 그대로 Ruby(`gitlab.rb`)로 넘어가므로, 주석은 YAML 주석이 아니라 Ruby 주석 `#`로 쓴다. 모든 줄의 들여쓰기는 첫 줄과 같은 깊이로 맞춰야 한다. 덜 들여쓴 줄이 나오면 블록이 그 앞에서 끝나 버린다.

주석에는 설정의 정의보다 **왜 이 값인지**를 적었다.

```yaml
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.example.com'

        ## Rails: 웹 화면이랑 API를 담당하는 GitLab 본체
        # clone 주소에 표시할 SSH 포트. 실제 포트 매핑과 일치시킨다.
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
        # Sidekiq 스레드가 15개라 DB 커넥션이 모자라지 않게 여유 있게 잡음
        gitlab_rails['db_pool'] = 20
        # 메일은 안 쓰니까 꺼둠
        gitlab_rails['gitlab_email_enabled'] = false

        ## Sidekiq: 웹훅, CI, 미러링 같은 뒷단 작업을 처리
        # push 후에 MR이나 파이프라인 갱신이 늦어지면 여기부터 의심해볼 것
        sidekiq['concurrency'] = 15

        ## Nginx: 앞단 웹서버. 인증서는 /etc/gitlab/ssl에 마운트해서 사용
        nginx['redirect_http_to_https'] = true
        nginx['ssl_certificate']     = "/etc/gitlab/ssl/<certificate-file>"
        nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/<private-key-file>"

        ## 로그: 영구 보관해야 해서 자동 삭제를 전부 꺼둠
        # gitaly, sidekiq 같은 서비스 로그는 원래 30개만 남기고 지워지는데, 0으로 두면 안 지움
        logging['svlogd_num'] = 0
        # rails, nginx 로그도 마찬가지. 여기선 0이 오히려 바로 지우는 값이라 -1로 둬야 함
        logging['logrotate_rotate'] = -1
        # 파일 이름에 번호 대신 날짜가 붙어서 나중에 찾기 편함 (production.log-YYYY-MM-DD.gz)
        logging['logrotate_dateformat'] = "-%Y-%m-%d"
```

### 주석을 달다가 발견한 SSH 포트 불일치

`gitlab_shell_ssh_port`에 주석을 달려고 compose의 포트 매핑을 다시 확인했다. 그런데 두 값이 달랐다.

- compose 포트 매핑: 호스트의 사용자 지정 포트 → 컨테이너 SSH 포트
- `gitlab_shell_ssh_port`: 실제 매핑과 일치하지 않는 포트

`gitlab_shell_ssh_port`는 포트를 여는 설정이 아니다. 웹 UI의 **Clone with SSH** 주소에 **표시할** 포트일 뿐이다. 컨테이너 안의 sshd는 여전히 22번에서 받고, 바깥에서 들어오는 실제 통로는 compose 매핑이 정한다. GitLab은 컨테이너 밖에서 몇 번 포트로 들어오는지 알 수 없어서 사람이 이 값으로 알려줘야 한다.

그래서 이 상태에서는 UI에 `ssh://git@gitlab.example.com:<wrong-port>/...`가 찍히고, 앞단에서 잘못 지정한 포트를 전달하는 장비가 없다면 그대로 복사한 사용자는 접속에 실패한다. HTTPS clone은 이 설정과 상관없다.

어느 쪽인지는 외부에서 두 포트로 직접 붙어보면 구분된다.

```bash
ssh -T -p <configured-port> git@gitlab.example.com
ssh -T -p <published-port> git@gitlab.example.com
```

`-T`는 TTY를 할당하지 않는 옵션이다. `Welcome to GitLab`이나 `Permission denied (publickey)`가 나오면 그 포트까지는 닿은 것이고, `Connection refused`나 타임아웃이면 그 포트로는 길이 없는 것이다.

이번 구성은 앞단에서 잘못 지정한 포트를 전달하는 장비가 없었다. 그래서 실제 외부 포트 매핑에 맞춰 `gitlab_shell_ssh_port`를 수정했다. 위 compose 예시는 고친 뒤의 값이다. 적용한 뒤에 웹 UI에서 프로젝트의 **Code → Clone with SSH** 주소에 실제 외부 SSH 포트가 표시되는지 보면 된다.

## 적용

`GITLAB_OMNIBUS_CONFIG`는 환경변수라 컨테이너를 만들 때 고정된다. `docker restart`로는 반영되지 않고, 컨테이너를 다시 만들어야 한다.

```bash
docker compose config   # YAML 문법 확인. 컨테이너는 건드리지 않음
docker compose up -d    # 설정이 달라진 컨테이너만 recreate
```

`up -d`는 compose 파일과 실행 중인 컨테이너의 설정을 비교해서, 다르면 중지 → 삭제 → 생성 → 시작한다. 새 컨테이너가 뜰 때 `gitlab-ctl reconfigure`가 자동으로 돈다.

## 반영 확인

설정 키가 실제 설정 파일로 렌더링됐는지 양쪽 모두 확인한다.

svlogd 쪽은 로그 디렉터리마다 `config` 파일이 생긴다.

```bash
docker exec gitlab cat /var/log/gitlab/gitaly/config
```

`s`(크기), `n`(개수), `t`(시간), `!gzip`(압축 필터) 중 `n0`이 보이면 반영된 것이다.

logrotate 쪽은 서비스별 설정 파일이 만들어진다.

```bash
docker exec gitlab cat /var/opt/gitlab/logrotate/logrotate.d/gitlab-rails
```

`rotate -1`, `dateext`, `dateformat -%Y-%m-%d`가 들어 있으면 된다.

실제 파일 이름은 다음 로테이트가 돈 뒤에 확인할 수 있다. 하루가 지나면 날짜 접미사가 붙은 파일이 생겨야 한다.

```bash
docker exec gitlab ls /var/log/gitlab/gitlab-rails/
```

## 남은 것

- 이미 기본값(30개)으로 지워진 로그는 되살릴 수 없다. 설정을 적용한 시점부터 쌓인다.
- 이제 로그 디렉터리는 계속 커지기만 한다. 디스크 사용률은 따로 지켜봐야 한다.
