---
title: "git을 2.x으로 올렸는데도 러너가 'Couldn't find remote ref refs/pipelines'로 실패한 사례"
date: 2026-10-07
tags: [gitlab, gitlab-runner, git, ci, troubleshooting]
status: done
---

## 상황

- GitLab CE 18.x (Docker 기반 구성)
- GitLab Runner 16.x, **shell executor**, 구형 RHEL 계열 Linux 환경
- 러너 호스트의 기본 Git은 OS 기본 패키지의 구버전

어느 날 특정 프로젝트의 파이프라인이 소스를 받는 단계(`get_sources`)에서 실패하기 시작했다.

## 증상

처음 에러는 이랬다.

```text
Fetching changes with git depth set to 20...
Reinitialized existing Git repository in <runner-build-dir>/.../.git/
fatal: git fetch-pack: expected shallow list
fatal: The remote end hung up unexpectedly
ERROR: Job failed: exit status 1
```

git을 업그레이드한 뒤에는 에러가 이렇게 바뀌었다.

```text
Fetching changes with git depth set to 20...
Initialized empty Git repository in <runner-build-dir>/.../.git/
Created fresh repository.
fatal: Couldn't find remote ref refs/pipelines/<pipeline-id>
ERROR: Job failed: exit status 1
```

새로 만든 파이프라인도 같은 에러로 실패했다. 얼마 뒤에는 같은 러너 호스트를 사용하는 다른 프로젝트들도 같은 에러로 실패하기 시작했다.

## 가설과 검증

### 가설 1: git 구버전 Git이 너무 오래돼서 shallow fetch를 못 한다 (반만 맞음)

`expected shallow list`는 오래된 git 클라이언트가 최신 GitLab과 shallow fetch(`--depth`) 협상을 할 때 나오는 에러로 알려져 있다. 러너 서버의 `git --version`은 구버전 Git.1이었다.

해당 OS 계열에서 사용할 수 있는 패키지 저장소를 통해 Git을 2.x 계열로 업데이트했다.

```bash
sudo yum update -y git
```

gitlab-runner RPM은 `git` 패키지에 의존한다. 그래서 `yum remove git` 후 재설치하면 러너까지 같이 지워진다. `update`로 교체해야 한다.

업그레이드 후 `git --version`은 2.x.0으로 나왔다. 그런데 에러가 `Couldn't find remote ref`로 바뀌었다.

### 가설 2: 끝난 파이프라인의 잡을 Retry 해서 ref가 없다 (이번엔 아님)

러너는 브랜치 이름 대신 `refs/pipelines/<ID>`라는 GitLab 내부 ref로 fetch한다. GitLab은 파이프라인이 끝나면 이 ref를 정리한다. 그래서 끝난 파이프라인의 잡을 Retry 하면 이 에러가 날 수 있다.

하지만 새 파이프라인을 만들어도 똑같이 실패했으므로 이번 원인은 아니었다.

### 가설 3: GitLab 서버가 pipeline ref를 만들지 못한다 (틀림)

다른 프로젝트까지 실패하는 걸 보고 서버 쪽을 의심했다. `gitlab-rails runner`로 최근 파이프라인의 ref 존재 여부를 조회했다.

```ruby
p.project.repository.ref_exists?("refs/pipelines/#{p.id}")
```

- 실행 중(running/pending)인 파이프라인은 모두 ref가 있었다.
- 끝난 파이프라인은 ref가 없었다. 위에서 말한 정상적인 정리 동작이다.

Gitaly 로그의 시간 순서도 확인했다.

```text
02:41:38  WriteRef    OK   ← 파이프라인 ref 생성
02:41:46  InfoRefsUploadPack (러너)   ← 러너의 첫 요청
          (러너의 다음 요청 없음)
02:42:01  DeleteRefs  OK   ← 잡 실패 → 파이프라인 종료 → ref 정리
```

러너가 fetch하던 시점에는 ref가 분명히 있었다. 서버는 정상이었다.

### 단서: 러너가 첫 요청 뒤에 멈춘다

nginx 로그에서 러너 IP의 요청을 업그레이드 전후로 비교했다.

| 시점 | info/refs 응답 크기 | 그 다음 |
|---|---|---|
| 업그레이드 전 (git 구버전 Git) | 363 bytes | `POST git-upload-pack` 진행 |
| 업그레이드 후 | 166 bytes | **POST 없이 종료** |

git 2.x는 기본으로 프로토콜 v2를 쓴다. v2에서는 첫 GET(`info/refs`)에 서버가 capability 목록만 돌려주고, 클라이언트가 이어서 `ls-refs` POST로 ref 목록을 요청한다. 166바이트는 v2 capability 응답이다. 그런데 클라이언트가 그 다음 POST를 보내지 않았다.

러너 서버에서 같은 계정, 같은 refspec으로 공개 저장소를 fetch해 보니 정상이었다. GET 다음에 `ls-refs` POST까지 나갔다.

### 가설 4: credential.helper=store가 인증 후 흐름을 깨뜨린다 (틀림)

내 재현과 실제 잡의 차이는 인증(401 → 토큰으로 재시도) 하나뿐이라고 생각했다. 마침 러너 실행 계정의 `~/.gitconfig`에 `credential.helper=store`가 있었다.

서버 설정을 건드리지 않으려고, 파이프라인 변수로 이 잡에서만 설정을 덮어쓰고 추적 옵션을 켰다.

```yaml
variables:
  GIT_TRACE: "1"
  GIT_TRANSPORT_HELPER_DEBUG: "1"   # fetch 본체 ↔ git-remote-https 대화 출력
  GIT_CONFIG_COUNT: "1"             # 이 잡에서만 git 설정 덮어쓰기
  GIT_CONFIG_KEY_0: "credential.helper"
  GIT_CONFIG_VALUE_0: ""
```

credential helper를 꺼도 똑같이 실패했다. 대신 디버그 출력에서 결정적인 단서가 나왔다.

```text
Debug: Remote helper: <- stateless-connect
Debug: Got cap stateless-connect      ← 도우미는 v2(stateless-connect)를 지원한다고 알림
...
Debug: Read ref listing.              ← fetch는 v2를 쓰지 않고 옛 방식 list로 요청
fatal: Couldn't find remote ref refs/pipelines/<pipeline-id>
```

내 재현에서는 `-> stateless-connect git-upload-pack`이 나갔다. 실제 잡의 fetch는 v2를 시도조차 하지 않았다.

### 가설 5: 잡에서 도는 git이 2.x이 아니다 (맞음)

결정적인 단서는 에러 메시지의 대소문자였다.

- 실제 잡: `fatal: Couldn't find remote ref` (대문자 C)
- 내 재현(Git 2.x): `fatal: couldn't find remote ref` (소문자 c)

git은 이후 버전에서 이 메시지를 소문자로 바꿨다. 잡의 fetch는 옛날 git이 실행하고 있다고 보고 확인에 들어갔다.

러너 프로세스의 실제 PATH를 확인했다.

```bash
$ sudo cat /proc/<gitlab-runner PID>/environ | tr '\0' '\n' | grep ^PATH
PATH=<service-path>

$ ls -la <custom-git-path>
-rwxr-xr-x. 1 root root 1527752  5월 29  2020 <custom-git-path>

$ <custom-git-path> --version
git version <legacy-version>
```

2020년에 누군가 수동으로 설치한 git 구버전 Git이 `/usr/local/bin`에 있었다. 이 파일은 어느 패키지에도 속하지 않는다. `yum update git`은 `/usr/bin/git`만 바꿨다.

**내가 잘못 판단한 지점**: 처음 확인할 때 `sudo -u gitlab-runner -i ...`로 `which git`을 봤고, `/usr/bin/git` 2.x이 나와서 "러너는 2.x을 쓴다"고 결론 내렸다. 하지만 `sudo`는 `secure_path`로 PATH를 바꾼다. 실제 잡 셸은 러너가 `su -s /bin/bash gitlab-runner -c 'bash -l'`로 띄우고, 서비스 프로세스의 PATH를 그대로 물려받는다. 잡과 같은 조건으로 확인하려면 이렇게 해야 했다.

```bash
sudo env -i PATH=<service-path> \
  su -s /bin/bash gitlab-runner -c 'bash -l -c "which git; git --version"'
# <custom-git-path>
# git version <legacy-version>
```

## 원인

업그레이드 후에는 git 두 버전이 섞여서 동작하고 있었다.

1. 잡은 PATH 순서 때문에 `<custom-git-path>`(구버전 Git)으로 `git fetch`를 실행한다.
2. 구버전 Git 본체는 HTTPS 통신을 exec-path(`<git-exec-path>`)의 `git-remote-https`에 맡긴다. 이 파일은 패키지 업그레이드로 **2.x 버전으로 바뀌어 있었다.**
3. 2.x 도우미는 `Git-Protocol: version=2` 헤더를 붙여 요청하고, 서버는 v2 capability로 응답한다.
4. 구버전 Git 본체는 v2를 모른다. 그래서 옛 방식(`list`)으로 ref 목록을 받는데, v2 응답에는 ref가 없으니 빈 목록이 된다.
5. 빈 목록에서 `refs/pipelines/<ID>`를 찾지 못해 `Couldn't find remote ref`로 끝난다.

정리하면 업그레이드 전의 `expected shallow list`도, 업그레이드 후의 `Couldn't find remote ref`도 원인은 잡이 구버전 Git으로 돌고 있었다는 것이었다. 업그레이드는 도우미만 바꿔서 에러의 모양만 바뀌었던 것으로 보인다.

## 해결

수동 설치된 옛 git 파일을 삭제하지 않고 백업 디렉터리로 옮겼다.

```bash
sudo mkdir -p <backup-dir>/git-old-version
sudo mv <custom-git-path> <custom-git-path>-receive-pack <custom-git-path>-shell \
        <custom-git-path>-upload-archive <custom-git-path>-upload-pack <backup-dir>/git-old-version/
```

러너는 재시작하지 않았다. 잡마다 셸이 새로 뜨면서 PATH를 다시 검색하기 때문이다.

조치 후 영향을 받던 여러 프로젝트의 build 잡이 모두 `Checking out <sha> as detached HEAD`까지 진행했고, 파이프라인이 성공했다.

## 남은 것

- 백업해 둔 옛 git은 한동안 문제가 없으면 지울 예정이다. 이 git을 절대경로로 부르는 스크립트가 있는지는 끝까지 확인하지 못했다.
- 러너(16.3)와 GitLab(18.11)의 버전 차이가 크다. 이번 원인은 아니었지만 러너 업그레이드를 검토할 필요가 있다.
- 이번에 시간이 가장 많이 든 부분은 "어떤 git이 실행되는가"를 잡과 다른 조건(`sudo -u`)으로 확인한 것이었다. shell executor에서 실행 파일 버전을 볼 때는 러너 프로세스의 PATH(`/proc/<pid>/environ`)부터 확인했으면 빨랐을 것 같다.
