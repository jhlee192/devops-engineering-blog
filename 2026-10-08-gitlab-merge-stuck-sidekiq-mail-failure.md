---
title: "GitLab 머지가 'Merging! Drum roll' 화면에서 몇 분씩 멈춰 있던 사례"
date: 2026-10-08
tags: [gitlab, sidekiq, docker, troubleshooting]
status: done
---

## 상황

- GitLab CE 18.11 (공식 Omnibus 이미지를 Docker Compose로 단독 운영)
- 단일 서버 구성(세부 사양 비공개)
- 설정은 컨테이너 안 `gitlab.rb`가 아니라 compose의 `GITLAB_OMNIBUS_CONFIG` 환경변수로 관리
- Sidekiq: 프로세스 1개, `sidekiq['concurrency'] = 15`

개발자들이 MR을 머지하면 한참 걸린다고 했다.

## 증상

머지 버튼을 누르면 이 화면에서 몇 분씩 멈춰 있었다.

```text
Merging! Drum roll, please... 🥁
```

GitLab에서 머지는 웹 요청이 바로 처리하지 않는다. 처리 순서는 이렇다.

1. Rails가 Sidekiq에 `MergeWorker` 잡을 넣는다. 이때 MR은 `locked` 상태가 된다.
2. Sidekiq가 Gitaly에 실제 merge를 요청한다.
3. 후처리가 끝나면 MR이 `merged` 상태로 바뀐다.

"Drum roll" 화면은 1번과 3번 사이를 기다리는 화면이다. 그래서 Sidekiq부터 봤다.

**Admin → Monitoring → Background jobs** 화면의 상태는 이랬다.

| 프로세스 | 스레드 | 작동 | Utilization |
|---|---|---|---|
| 1 | 설정된 동시성 한도 | 한도 도달 | 100% |

## 가설과 검증

### 가설 1: 오래 걸리는 잡 몇 개가 스레드를 붙잡고 있다 (아니었음)

대형 import나 미러 동기화처럼 몇 분씩 걸리는 잡이 스레드를 차지하면 이렇게 될 수 있다. `gitlab-rails runner`로 실행 중인 잡 목록과 큐 상태를 조회했다.

```text
== processes
conc=<configured-limit> busy=<configured-limit>
== queues (size>0)
default size=<대기 잡 수> latency=<수 분>
mailers size=<대기 잡 수> latency=<수 초>
== busy
18s  NewMergeRequestWorker
11s  Repositories::PostReceiveWorker
10s  MergeWorker
 9s  ActionMailer::MailDeliveryJob   (x8)
 7s  Ci::PipelineCreationMetricsWorker
 ...
```

- 오래 붙잡고 있는 잡은 없었다. 대부분 수 초 안에 끝나는 잡이었다.
- 대신 `default` 큐에 수백 건이 쌓여 있었고, 가장 오래 기다린 잡의 대기 시간(latency)이 수 분 수준이었다.
- 머지가 느린 게 아니라 **머지 잡이 4분 가까이 순서를 기다리고 있었다.**

처리량이 모자란 상태였다. 다만 실행 중인 15개 중 **8개가 메일 발송 잡**이라는 게 이상했다.

### 가설 2: 스레드가 모자라니 늘리면 된다 (보류)

처음 떠올린 해결책은 스레드 증설이었다. 하지만 서버 상태가 좋지 않았다.

```text
load average: <높은 부하 상태>
%Cpu(s): 71.5 us, 18.4 sy, ... 7.0 id
  PID  %CPU  COMMAND
 1544 100.0  dockerd
```

CPU 유휴율이 한 자릿수 수준이었다. 이 상태에서 스레드나 프로세스를 늘리면 같은 CPU를 나눠 쓰게 될 뿐이라, 늘리기 전에 무엇이 처리 시간을 쓰고 있는지부터 보기로 했다.

### 가설 3: 메일 잡이 처리 시간을 잡아먹고 있다 (맞음)

Sidekiq 로그(JSON)에서 최근 10분 동안 끝난 잡을 클래스별로 묶어 `duration_s`를 더했다.

| 잡 | 건수 | 평균 | 합계 |
|---|---|---|---|
| **ActionMailer::MailDeliveryJob** | 수백 건(전부 fail) | 수 초 | 수천 초 |
| NewMergeRequestWorker | 20 | 66.8초 | 1,336초 |
| ProcessCommitWorker | 134 | 4.2초 | 559초 |
| Repositories::PostReceiveWorker | 39 | 13.6초 | 530초 |
| MergeWorker | 11 | 28.8초 | 317초 |

짧은 관찰 구간 동안 사용 가능한 스레드 시간의 대부분이 소진되고 있었다. 그중 상당 부분이 실패하는 메일 발송 작업에 쓰이고 있었다.

실패 원인은 Dead 목록에서 바로 나왔다.

```text
<대부분> ["ActiveJob::QueueAdapters::SidekiqAdapter::JobWrapper", "Errno::ENOENT"]
   1 ["NewMergeRequestWorker", "GRPC::DeadlineExceeded"]
```

- Dead 목록은 설정된 보관 상한에 도달해 있었고, 최근 항목 대부분이 메일 잡의 `Errno::ENOENT`였다.
- 가장 오래된 항목이 6일 전이었다. 그러니 적어도 그때부터 GitLab 알림 메일이 한 통도 나가지 않고 있었던 것으로 보인다.
- 메일 잡은 실패하면 3번 더 재시도한다. 그래서 메일 하나가 스레드를 최대 4번 차지한다.

## 원인

### 메일: SMTP를 설정하지 않아 sendmail로 보내고 있었다

`gitlab-ctl show-config`로 적용된 설정을 보면 메일 관련 값은 보내는 사람 주소(`gitlab_email_from`)뿐이었다. 그런데 설정 값이 없다는 게 "꺼져 있다"는 뜻은 아니었다. 기본값을 따라가 보니 이랬다.

| 단계 | 위치 | 값 |
|---|---|---|
| Omnibus 기본값 | `cookbooks/gitlab/attributes/default.rb` | `smtp_enable = false` |
| reconfigure로 만들어진 설정 파일 | `gitlab-rails/etc/gitlab.yml` | `email_enabled:` (빈 값) |
| Rails 초기화 | `config/initializers/1_settings.rb` | 빈 값이면 `email_enabled = true` |
| Rails 발송 방식 | `config/environments/production.rb` | `delivery_method = :sendmail` |

- 메일 기능은 켜져 있고(`email_enabled = true`), SMTP는 꺼져 있다.
- 그래서 Rails 기본값인 sendmail로 발송한다.
- 그런데 컨테이너 안에 `/usr/sbin/sendmail`이 없다. 그래서 `ENOENT`(파일 없음)가 난다.

Rails에서 실제 값을 확인하니 그대로였다.

```bash
gitlab-rails runner 'puts Gitlab.config.gitlab.email_enabled; puts ActionMailer::Base.delivery_method'
# true
# sendmail
```

`show-config`는 사용자가 지정한 값과 일부 계산된 값만 보여 준다. 이번처럼 Rails가 실행될 때 채우는 기본값은 나오지 않는다.

### CPU: dockerd가 코어 하나를 다 쓰고 있었다

컨테이너의 Docker 로그 설정을 확인했다.

```bash
$ docker inspect gitlab --format '{{.HostConfig.LogConfig}}'
{json-file map[]}

$ ls -lh <컨테이너 로그 경로>
-rw-r----- 1 root root 17G ... <container-id>-json.log
```

로테이션 설정이 없어서 컨테이너 stdout 로그가 대용량으로 누적돼 있었다. Omnibus 이미지는 내부 로그를 stdout으로도 내보낸다. dockerd의 100% CPU는 이 로그 처리 부하로 보인다. 다만 dockerd를 프로파일링해서 확인하지는 않았다.

## 해결

### 1차 시도: `smtp_enable = false` (효과 없음)

메일을 끄려고 compose에 `gitlab_rails['smtp_enable'] = false`를 먼저 넣었다. 하지만 위 표에서 보듯 이 값은 원래 기본값이 false다. 이미 적용되어 있는 상태를 다시 적은 것이라 아무것도 바뀌지 않는다.

`smtp_enable`은 **어떻게** 보낼지를 정하고, `gitlab_email_enabled`는 **보낼지 말지**를 정한다.

### 실제 적용

이 GitLab에서는 메일을 쓰지 않기로 하고 메일 기능 자체를 껐다.

```yaml
# docker-compose.yml
environment:
  GITLAB_OMNIBUS_CONFIG: |
    ...
    gitlab_rails['gitlab_email_enabled'] = false
```

Docker 로그 로테이션은 데몬 기본값으로 걸었다.

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "500m",
    "max-file": "20"
  }
}
```

적용 순서는 이렇다.

```bash
sudo python3 -m json.tool /etc/docker/daemon.json   # 문법 오류가 있으면 dockerd가 안 뜬다
sudo systemctl restart docker                       # daemon.json은 데몬 재시작 때 읽힌다
sudo docker compose down                            # 기존 컨테이너 삭제
sudo docker compose up -d                           # 새 설정으로 다시 생성
```

`log-opts`는 컨테이너를 만들 때 고정된다. 그래서 기존 컨테이너를 `docker start`로 다시 켜기만 하면 이전 설정(로테이션 없음)이 그대로 남는다. 반드시 컨테이너를 다시 만들어야 한다. 기존 컨테이너를 지우면 누적된 로그 파일도 같이 지워진다.

volumes가 `$GITLAB_HOME/config` 같은 변수 형태였다. 그래서 재생성하기 전에 `docker compose config`로 변수가 풀린 경로를 확인해서, 기존 컨테이너의 마운트 경로와 같은지 맞춰 봐야 한다. 변수가 비어 있으면 빈 디렉터리로 새 GitLab이 초기화되기 때문이다.

마지막으로 Dead 목록이 설정된 보관 상한에 도달한 상태(대부분 메일 실패)은 Background jobs 화면에서 비웠다. 그대로 두면 앞으로 다른 잡이 실패해도 묻혀서 보이지 않는다.

### 결과

| 항목 | 조치 전 (평일 17시) | 조치 후 |
|---|---|---|
| load average | 높은 부하 | 부하 완화 |
| CPU 유휴 | 한 자릿수 | 약 1/3 수준 |
| dockerd CPU | 코어 1개 수준 | 낮은 수준 |
| Sidekiq 사용 중 스레드 | 15/15 | 0~4 (10초 간격 12회 측정) |
| `default` 큐 대기 건수 / latency | 수백 건 / 수 분 | 0 / 0초 |
| 스레드 사용률 | 90% 이상 | 낮은 수준 |
| 실패 잡 | 관찰 구간 동안 다수 발생 | 0건 |

메일 잡이 사라진 것 외에, 잡 하나하나의 실행 시간도 줄었다. 예를 들어 `FlushCounterIncrementsWorker`는 평균 3.6초에서 0.55초가 됐다. CPU 경합이 줄어든 효과로 보인다.

스레드 증설은 하지 않았다.

## 남은 것

- **조치 후 수치는 재기동 직후, 오전에 잰 값이다.** 조치 전 측정은 오후 5시였다. 부하가 많은 시간대에 다시 재서 비교해야 한다.
- 웹 UI에서 스레드가 잠깐씩 15/15로 차는 것은 아직 보인다. 원격 미러 동기화 잡(`RepositoryUpdateRemoteMirrorWorker`)이 평균 104초씩 걸린다. 여기에 push나 파이프라인 잡이 몰리면 그렇게 되는 것으로 보인다. 스레드가 순간적으로 차는지보다 **latency가 금방 0으로 돌아오는지**를 기준으로 보기로 했다.
- 사용량이 많은 시간에도 latency가 계속 높게 유지되면 Sidekiq를 증설한다. Sidekiq 프로세스 하나는 CPU 코어 1개까지만 쓸 수 있어서, 늘린다면 스레드보다 `queue_groups`로 프로세스를 늘리는 쪽을 검토하고 있다. DB 커넥션은 설정된 풀 한도 대비 사용량에 여유가 있었다.
- 메일 기능을 껐으니 알림 메일은 나가지 않는다. 다만 조치 전에도 이미 모두 실패하고 있었다. 메일이 필요해지면 조직에서 제공하는 SMTP 릴레이로 `smtp_enable`을 설정해야 한다.
- 조치 전에 `NewMergeRequestWorker`가 평균 67초 걸렸고 Gitaly 타임아웃(`DeadlineExceeded`)도 1건 있었다. 이번에는 따로 확인하지 않았다.
