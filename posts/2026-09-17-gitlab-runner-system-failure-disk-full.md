---
title: "GitLab \"There has been a runner system failure\" — 원인이 디스크 부족이었던 사례"
date: 2026-09-17
tags: ["GitLab", "GitLab Runner", "CI/CD", "트러블슈팅", "systemd"]
status: done
---

파이프라인이 실패했다는 접수를 받았다. GitLab 화면에 찍힌 메시지는 이것뿐이었다.

```
There has been a runner system failure, please try again
```

이 메시지의 문제는 **아무것도 알려주지 않는다는 것**이다. 러너 쪽에 문제가 생겼다는 것까지는 알겠는데, 어떤 문제인지는 한 글자도 없다. 게다가 "다시 시도해보라"는 안내 때문에 일시적인 문제로 넘기기 쉽다.

러너 호스트 로그를 확인해보니 이번 건의 원인은 **디스크 부족**이었다. 이 문구가 뜰 수 있는 경우 중 하나로 기록해둔다.

## 환경

- GitLab Runner, `shell` executor
- 러너는 systemd 서비스(`gitlab-runner`)로 등록
- `max_builds=10`, 장애 시점 동시 빌드 5개

## 러너 호스트의 로그 확인

GitLab UI에는 job 로그조차 비어 있었다. 그래서 러너가 돌고 있는 호스트로 들어가 systemd 저널을 직접 확인했다.

실패 시각을 알고 있으니 범위를 좁혀서 뽑는다.

```bash
journalctl -u gitlab-runner \
  --since '2026-09-16 19:10' \
  --until '2026-09-16 19:30' \
  --no-pager > runner.log
```

`--no-pager`를 붙여야 파일로 리다이렉트할 때 페이저에 걸리지 않는다. 러너 한 대가 여러 프로젝트를 물고 있으면 로그가 섞이므로, 추출한 뒤 **프로젝트명과 러너 토큰이 문제의 job과 일치하는지** 먼저 확인해야 한다. 엉뚱한 job의 로그를 보고 헤매기 쉽다.

## 찾은 로그

WARNING 두 줄이 연속으로 찍혀 있었다.

```
WARNING: Submitting job to coordinator... job failed
  bytesize=0 checksum= code=200 job=<job-id> job-status=failed
  runner=<runner-token> update-interval=0s

WARNING: Failed to process runner
  builds=<concurrent-build-count> error=open /tmp/trace<id>: no space left on device
  executor=shell max_builds=10 runner=<runner-token>
```

`no space left on device`. 디스크가 찼다.

## 두 로그가 어떻게 이어지는가

이 두 줄을 따로 보면 안 된다. 순서대로 읽으면 사건이 그대로 재구성된다.

**아래 줄이 먼저 벌어진 일이다.** 러너는 job의 실행 로그를 `/tmp/trace...` 파일에 버퍼링한 뒤 GitLab 서버로 전송한다. 그런데 디스크가 차서 이 임시 파일을 **열지도 못했다**.

**위 줄이 그 결과다.** `bytesize=0 checksum=` — 서버로 올린 job 로그가 0바이트다. 쓸 파일이 없었으니 당연하다. GitLab UI의 job 로그가 텅 비어 있었던 것도 이 때문이다.

여기서 눈여겨볼 건 **`code=200`** 이다. GitLab 서버와의 통신 자체는 성공(HTTP 200)했다. 러너 프로세스도 죽지 않았고 네트워크도 멀쩡했다. 그래서 러너 상태나 네트워크부터 의심하면 헛다리를 짚기 쉽다. 실패한 건 통신이 아니라 **로컬 파일 쓰기**였다.

`There has been a runner system failure`라는 모호한 메시지도 여기서 나온 것으로 보인다. GitLab 서버 입장에서는 "러너가 job을 받아갔는데 빈 결과를 보내왔다"는 것밖에 알 수 없으니, 원인을 특정하지 못하고 뭉뚱그린 메시지를 내보내는 셈이다.

## 이후

**아무 조치도 하지 않았는데 용량이 해소됐다.**

디스크를 정리한 적이 없으니 무언가가 알아서 공간을 반납했다는 뜻이다. 실행 중이던 job이 끝나면서 빌드 디렉터리와 임시 파일이 정리됐을 가능성이 크다. 무엇이 정확히 공간을 차지했다 풀었는지는 추적하지 않았다. 이미 해소된 건을 되짚는 비용 대비 실익이 크지 않다고 판단했다.

다만 이건 문제가 해결된 게 아니라 사라진 것에 가깝다.

## 남은 것

- **디스크 사용률 모니터링과 임계치 알람.** 이게 없어서 파이프라인 실패로 접수될 때까지 몰랐다. 우선순위 1순위다.
- **빌드 디렉터리 정리 주기 검토.** `shell` executor는 컨테이너 executor와 달리 빌드 디렉터리가 호스트에 그대로 쌓이고 자동으로 정리되지 않는다.

## 정리

`There has been a runner system failure`는 원인이 아니라 증상이다. 이 문구만으로는 아무것도 알 수 없으니 러너 호스트의 `journalctl -u gitlab-runner`를 열어보는 게 빠르다.

디스크 부족은 그 문구가 뜨는 여러 원인 중 하나일 뿐이지만, job 로그가 비어 있다면(`bytesize=0`) 확인해볼 만한 후보다. 같은 메시지를 만난 분에게 하나의 사례로 도움이 되면 좋겠다.
