---
title: "Loki PVC가 가득 찬 뒤, 용량을 비웠는데도 Loki가 뜨지 않았던 사례"
date: 2026-10-21
tags: ["Loki", "Kubernetes", "PVC", "retention", "boltdb-shipper", "트러블슈팅"]
status: done
---

Loki PVC가 가득 차서 로그 수집이 멈췄다. 오래된 chunk를 지워 용량은 확보했는데, **Loki는 그 뒤에도 계속 CrashLoopBackOff였다.** 디스크를 비웠으니 다시 뜰 거라고 생각했지만 그렇지 않았다.

이 글은 그다음에 확인한 두 가지를 기록한 것이다.

- 비운 뒤에도 왜 안 떴는지
- 애초에 왜 1Ti가 찼는지

## 환경

- Loki 2.6.1, Helm 차트 `loki-stack` 2.9.9, 단일 replica StatefulSet
- 저장 방식: `boltdb-shipper`(인덱스) + `filesystem`(chunk). 오브젝트 스토리지 없이 PVC 하나에 전부 저장
- PVC 1Ti, Dell PowerFlex CSI(`vxflexos`)
- 수집기: Grafana Alloy DaemonSet

## 1. 용량 확보

첫 단계는 무엇이 공간을 차지했는지 보는 것이다. 이때는 Pod가 떠 있어서 `exec`로 들어갈 수 있었다.

```bash
kubectl -n loki exec loki-0 -- du -sh /data/loki/*
```

`du -s`는 디렉터리별 합계만 보여주고, `-h`는 G/M 단위로 표시한다. 결과는 거의 전부 `chunks`였다. `chunks`는 실제 로그 데이터다. `wal`이 컸다면 저장소로 내보내지 못한 로그가 밀렸다는 뜻이라 원인이 달라진다.

PVC를 늘리는 게 가장 안전하지만, 급한 대로 오래된 chunk를 지웠다. 지우기 전에 대상 개수부터 확인했다.

```bash
find /data/loki/chunks -type f -mtime +<days> | wc -l
find /data/loki/chunks -type f -mtime +<days> -delete
```

이후 조회해 보니 10/10 이전 데이터는 남아 있지 않았다.

이 작업으로 사용률이 **49%(1Ti 중 469G)**까지 내려갔다. inode도 1%라 여유가 있었다.

## 2. 비웠는데도 계속 죽는다

### 로그를 볼 수가 없었다

Pod 상태를 보면 이상한 점이 있었다. **AGE가 늘 몇 초였다.** 재시작 횟수(RESTARTS)가 쌓이는 게 아니라, Pod가 아예 새로 만들어지고 있었다.

```text
Warning  CrashLoopBackOff  pod/loki-0  podmon cleaning pod [... <node> retry: 1] with delete
Normal   Scheduled         pod/loki-0  Successfully assigned loki/loki-0 to <node>
```

PowerFlex CSI에 딸린 **podmon**이 CrashLoop에 빠진 Pod를 삭제하고 있었다. podmon은 스토리지 장애에서 Pod를 다른 노드로 옮기려고 쓰는 컴포넌트다. 이 때문에 `kubectl logs --previous`로 직전 컨테이너 로그를 볼 수 없었다. 지워진 Pod에는 "직전 컨테이너"가 없기 때문이다.

그래서 컨테이너가 떠 있는 2초 남짓한 순간을 잡았다. 0.5초 간격으로 `kubectl logs`를 반복하다가, 출력이 나오는 순간 저장하는 방식이었다.

```bash
for i in $(seq 1 120); do
  out=$(kubectl -n loki logs loki-0 2>/dev/null)
  if [ -n "$out" ]; then echo "$out" > loki-crash.log; break; fi
  sleep 0.5
done
```

### 잡아낸 로그

```text
level=error caller=table.go:376 msg="failed to open file /data/loki/boltdb-shipper-active/index_20742/1792116000. Please fix or remove this file." err="recovered from panic opening boltdb file: runtime error: invalid memory address or nil pointer dereference"
level=error caller=table.go:376 msg="failed to open file /data/loki/boltdb-shipper-active/index_20742/1792118700. Please fix or remove this file." err="file size too small"
...
unexpected fault address 0x7f41e4012008
fatal error: fault
[signal SIGBUS: bus error code=0x2 addr=0x7f41e4012008 pc=0x17c3071]
goroutine 1 [running]:
go.etcd.io/bbolt.(*Cursor).first(...)
...
github.com/grafana/loki/pkg/storage/stores/shipper/index.loadBoltDBsFromDir(...)
```

Loki는 기동할 때 아직 업로드하지 않은 인덱스(`boltdb-shipper-active`)를 연다. 처음 몇 개 파일은 `file size too small`, `recovered from panic`으로 에러를 남기고 넘어갔다. 그런데 그다음 파일에서 bbolt가 **SIGBUS**로 프로세스를 통째로 죽였다.

SIGBUS는 메모리 매핑한 파일에서 실제로는 없는 영역을 읽을 때 나는 신호다. bbolt는 파일을 메모리에 매핑해서 읽는다. 그래서 파일 헤더에 적힌 크기와 실제 크기가 맞지 않으면 이렇게 죽는 것으로 보인다.

### 깨진 파일들

해당 디렉터리를 열어 보니 이런 상태였다.

```text
-rw-rw-r--  8192   10-16 02:07  1792116000
-rw-rw-r--  4096   10-16 02:46  1792118700
-rw-rw-r--  4096   10-16 03:31  1792121400
-rw-rw-r--  524288 10-16 03:59  1792122300
-rw-rw-r--  262144 10-16 04:14  1792123200
...                                          (256KB 파일 7개)
-rw-rw-r--  4096   10-16 05:31  1792128600
-rw-rw-r--  8192   10-16 05:45  1792129500
-rw-rw-r--  12288  10-16 06:02  1792130400
-rw-rw-r--  0      10-16 06:00  1792130418224351870.temp
-rw-rw-r--  0      10-16 06:01  1792130478223697958.temp
```

시각은 모두 **10/16 02:00~06:02(UTC)로, 디스크가 차 가던 시간대**다. 정상 파일은 256~512KB인데 사이사이에 4~12KB짜리가 끼어 있었다. 쓰다가 공간이 없어 도중에 멈춘 파일로 보인다.

정리하면, 디스크를 비운 것은 맞지만 **디스크가 찼을 때 망가진 파일은 그대로 남아 있었다.** 공간을 만든다고 깨진 파일이 고쳐지지는 않는다.

### 조치

```bash
kubectl -n loki scale sts loki --replicas=0
kubectl -n loki wait --for=delete pod/loki-0 --timeout=120s
```

볼륨이 RWO라 기존 Pod가 완전히 사라져야 다른 Pod가 붙일 수 있다. 그래서 `wait --for=delete`로 기다렸다.

임시 Pod는 Loki와 같은 UID(10001)로 띄우고, PVC를 같은 경로(`/data`)에 마운트했다. 이미지는 노드에 이미 있는 Loki 이미지를 썼다. 그래서 이미지를 새로 받을 필요가 없었고, 안에 busybox 쉘도 있었다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: loki-pvc-fix
  namespace: loki
spec:
  securityContext:
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
  containers:
  - name: sh
    image: <registry>/grafana/loki:2.6.1
    command: ["sleep", "3600"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: storage-loki-0
```

깨진 파일은 지우지 않고 Loki가 읽지 않는 경로로 옮겼다. 다른 파일이 문제였다면 되돌려야 하기 때문이다. 정상 크기로 보이는 256~512KB 파일 7개는 그대로 두었다.

```bash
cd /data/loki/boltdb-shipper-active/index_20742
mkdir -p /data/loki-corrupt-backup/index_20742
mv 1792116000 1792118700 1792121400 1792128600 1792129500 1792130400 *.temp \
   /data/loki-corrupt-backup/index_20742/
```

임시 Pod를 지우고 replica를 1로 되돌리자 이번에는 떴다.

```text
msg="recovered WAL checkpoint recovery finished" elapsed=302.766068ms errors=false
msg="WAL segment recovery finished" elapsed=27.182427454s errors=false
msg="Loki started"
```

WAL 8.7G를 복구하는 데 27초가 걸렸고, Ready까지는 약 1분 40초가 걸렸다. Alloy가 재시도로 밀린 로그를 다시 보내면서 수집도 정상으로 돌아왔다. 옮긴 파일에 해당하는 10/16 02:00~06:00(UTC) 구간은 일부 조회가 안 될 수 있다. 다만 디스크가 차던 구간이라 원래 온전하지 않았을 가능성이 크다. 정상 기동을 확인한 뒤 백업 디렉터리는 지웠다.

## 3. 애초에 왜 1Ti가 찼나

Loki가 뜬 뒤 설정을 다시 봤다. retention은 켜져 있었다.

```yaml
compactor:
  retention_enabled: true
  ...
table_manager:
  retention_deletes_enabled: true
  retention_period: 120h
```

`retention_period: 120h`가 있으니 5일치만 남아야 한다. 그런데 실제로 적용된 값은 메트릭에 그대로 드러나 있었다.

```bash
kubectl -n loki exec loki-0 -- wget -qO- localhost:3100/metrics \
  | grep '^loki_distributor_bytes_received_total'
```

```text
loki_distributor_bytes_received_total{retention_hours="744",tenant="fake"} 1.085136571e+09
```

`retention_hours="744"`, 즉 **31일**이다. 120h는 적용되지 않고 있었다.

원인은 설정 위치였다. `120h`는 예전 방식인 **`table_manager`** 아래에 들어 있었다. 그런데 실제로 삭제를 수행하는 건 compactor이고, compactor가 보는 보존 기간은 **`limits_config.retention_period`**다. `limits_config`에는 이 값이 없었다.

```yaml
limits_config:
    enforce_metric_name: false
    reject_old_samples: true
    reject_old_samples_max_age: 168h
    max_entries_limit_per_query: 5000
    ingestion_rate_mb: 30
    ingestion_burst_size_mb: 40
    per_stream_rate_limit: 8mb
    per_stream_rate_limit_burst: 20mb
    # retention_period 없음 → 기본값 744h
```

compactor 방식의 retention 설정은 [Loki 공식 문서의 Retention 항목](https://grafana.com/docs/loki/latest/operations/storage/retention/)에 정리돼 있다. 이번에 실수한 지점은 retention 관련 설정이 compactor, limits_config, table_manager 세 곳에 흩어져 있다는 것이다. 겉으로 보기에 비슷한 이름의 값이 있으니, 설정 파일만 봐서는 적용됐다고 착각하기 쉬웠다.

## 4. 며칠로 할지 정하기

`limits_config`에 넣기로 하고, 기간은 실제로 쌓이는 양을 보고 정했다. Prometheus에서 Loki가 하루 동안 디스크에 내보낸 chunk 양(압축 후)과 받은 원본 양을 뽑았다.

```promql
sum(increase(loki_ingester_chunk_stored_bytes_total[1d]))
sum(increase(loki_distributor_bytes_received_total[1d]))
```

| 날짜 | 저장량(압축 후) | 원본 수신량 |
|---|---|---|
| 10/10 (토) | 1.1 GB | 7.7 GB |
| 10/11 (일) | 3.8 GB | 27.8 GB |
| 10/12~15 (월~목) | 63~68 GB/일 | 374~424 GB/일 |

처음에는 10/12부터 17배로 뛴 걸 보고 무언가 잘못 배포된 줄 알았다. 그런데 **10/10, 10/11은 주말**이었다. 장애가 해결된 날 오후의 유입량(시간당 원본 약 34GB)도 지난주 같은 요일·시간대(약 39GB)와 비슷했다. 그래서 이상 급증이 아니라 평일과 주말의 차이로 판단했다. 평일에는 특정 게이트웨이 서비스 4개 Pod가 각각 시간당 약 5.5GB를 남기고 있었다.

다만 메트릭 값과 실제 디스크 사용량이 맞지 않았다. 메트릭으로 더한 양보다 디스크에 쌓인 양이 약 1.5배 많았다. 작은 chunk 파일이 50만 개 정도라서 파일시스템 블록 낭비가 있는 것으로 보인다. 그래서 디스크 기준으로 **평일 하루 약 100GB, 1주일 약 500GB**로 잡았다.

| 보관 기간 | 최악의 경우 사용량 | 1Ti 대비 |
|---|---|---|
| 5일 | 약 500 GB | 50% |
| **7일** | **약 500~600 GB** | **50~60%** |
| 10일 | 약 800 GB 이상 | 80% 이상 |
| 14일 | 약 1 TB | 100% |

**7일(`168h`)**로 정했다. 이유는 세 가지다.

- 실제 삭제는 하루 단위 인덱스 테이블로 이뤄지고, `retention_delete_delay`(기본 2h)도 있다. 그래서 디스크에는 실제로 8일치 가까이 남는 것으로 보고 계산했다.
- 지금 설정의 `reject_old_samples_max_age: 168h`와 기간이 맞는다.
- 설정에 `max_look_back_period: 240h`가 있다. 보관 기간을 그 이상으로 늘려도 쿼리로는 10일까지만 조회된다.

```yaml
limits_config:
    ...
    retention_period: 168h
```

측정 기간이 1주일뿐이라 월말·월초 같은 시기는 반영되지 않았다. 적용한 뒤 사용률을 지켜볼 예정이다.

## 5. 재배포하려다 발견한 것

Helm으로 설치한 릴리스라 values만 바꿔서 `helm upgrade`를 하면 될 줄 알았다. 그런데 사용자 지정 values를 보니 이상했다.

```bash
helm get values -n loki loki
```

이미지 경로, PVC 크기, 스토리지 클래스, 앞에서 본 `limits_config`까지, **실제 적용된 설정 대부분이 여기에 없었다.** 처음 설치할 때 **차트의 `values.yaml` 자체를 고쳐서** 설치한 것으로 보였다. 릴리스에 남은 차트 파일 목록에 `values.yaml_backup`이 있는 것도 그 흔적으로 보인다.

이 상태에서 순정 차트에 `--reuse-values`로 올리면, 고쳐 둔 기본값들이 순정 값으로 돌아간다. 예를 들어 이미지가 사내 레지스트리에서 Docker Hub로 바뀐다. 서버에 남아 있던 차트 디렉터리도 확인했는데 버전이 2.0.2였다. 렌더링해 보니 지금 클러스터에서는 없어진 `PodSecurityPolicy(policy/v1beta1)`가 생기고, ring 저장소 설정도 바뀌어서 쓸 수 없었다.

그래서 이렇게 진행하기로 했다.

```bash
# 차트 기본값까지 합친 최종 적용 값 전부
helm get values -n loki loki -a -o yaml > loki-values-full.yaml
# limits_config.retention_period: 168h 추가 후

helm pull loki-stack --repo https://grafana.github.io/helm-charts --version 2.9.9 --untar

helm get manifest -n loki loki > deployed.yaml
helm template loki ./loki-stack -n loki -f loki-values-full.yaml > rendered.yaml
diff deployed.yaml rendered.yaml
```

diff에 Secret의 `loki.yaml`과 Pod 템플릿의 `checksum/config`만 나오면 `-f loki-values-full.yaml`로 업그레이드한다.

재배포는 아직 하지 않았다. 재배포 후 확인할 항목은 아래 '남은 것'에 적어 두었다.

## 남은 것

- **retention 적용 확인.** 재배포 후 `retention_hours`가 `168`로 바뀌는지, 사용률이 실제로 내려가는지 확인할 것
- **PVC 사용률 알람.** 이번에도 로그 수집이 멈추고 나서야 알았다. `kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes`로 80% 알람을 걸 예정
- **로그 양.** 게이트웨이 서비스 하나가 유입량의 절반 이상을 차지한다. 로그 레벨을 줄이거나 수집 단계에서 버리면 같은 디스크로 더 오래 보관할 수 있다
- **values 파일 보관.** 이번처럼 원본 차트를 찾지 못하는 일이 없도록 values 파일과 차트를 정해진 위치에 남겨둘 것

## 정리

디스크가 차서 멈춘 Loki는 공간을 비운다고 바로 살아나지 않을 수 있다. 이번에는 디스크가 찰 때 쓰다 만 boltdb 인덱스 파일이 남아 있었고, Loki가 기동하면서 그 파일을 읽다가 SIGBUS로 죽었다. 로그에 `Please fix or remove this file`이 찍혀 있다면 그 파일부터 보면 된다.

디스크가 찬 이유는 retention이 꺼져서가 아니라 **엉뚱한 섹션에 들어가 있어서**였다. 설정 파일에 값이 보인다고 적용된 것은 아니었다. 메트릭의 `retention_hours` 라벨로 실제 값을 확인할 수 있다.
