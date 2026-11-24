---
title: "삭제된 Pod의 재시작 횟수를 나중에 확인할 수 있을까"
date: 2026-09-23
tags: [kubernetes, prometheus, promql, kube-state-metrics, grafana]
status: done
---

## 왜 알아야 하나

Pod가 이미 delete되고 새 Pod가 올라온 뒤에 "아까 그거 몇 번이나 재시작했었지?"를 확인하려 했다.

`kubectl get pod`의 RESTARTS 컬럼은 현재 떠 있는 Pod의 값이다. 새로 올라온 Pod는 0이다.
이전 Pod의 값을 볼 방법이 `kubectl`에는 없다.

## 핵심 개념: restartCount는 Pod 오브젝트에 붙어산다

재시작 횟수는 어디 따로 누적되는 카운터가 아니라 Pod 리소스의 필드 하나다.

```
status:
  containerStatuses:
  - name: myapp
    restartCount: 4
```

그래서 etcd에서 Pod 오브젝트가 지워지면 이 값도 같이 없어진다.
새로 뜬 Pod는 UID부터 다른 별개의 오브젝트라 0에서 다시 시작한다.
Deployment나 ReplicaSet, 노드 어디에도 누적되지 않는다.

결국 **Pod 바깥으로 이 값을 내보내 저장해둔 것이 있어야** 사후 조회가 된다.
Prometheus + kube-state-metrics 조합이 그 역할을 한다.
kube-state-metrics가 `kube_pod_container_status_restarts_total`을 Pod 이름 레이블 단위로 노출하고,
Prometheus가 그걸 시계열로 저장해두기 때문이다.

## 동작 원리: range vector를 써야 하는 이유

처음엔 이렇게 조회했다.

```promql
kube_pod_container_status_restarts_total{namespace="myns"}
```

살아있는 Pod만 나온다. 이유는 이렇다.

Pod가 삭제되면 kube-state-metrics는 해당 시계열 노출을 멈춘다.
Prometheus는 수집이 끊긴 시계열에 **stale 마커**를 붙이고, 이후 instant vector 조회 결과에서 제외한다.
데이터가 지워진 게 아니라 "지금 시점의 값"으로는 안 잡히는 것이다.

range vector는 다르다. 구간 안의 샘플을 그대로 읽어오기 때문에 이미 사라진 시계열도 걸린다.

```promql
max_over_time(kube_pod_container_status_restarts_total{namespace="myns"}[7d]) > 0
```

`max_over_time(...[7d])`은 지난 7일 구간의 샘플 중 최댓값을 뽑는다.
restartCount는 Pod 생애 동안 줄지 않는 값이라, **최댓값이 곧 삭제 직전의 최종 재시작 횟수**가 된다.

뒤의 `> 0`은 PromQL 비교 연산자가 기본적으로 필터로 동작하는 성질을 쓴 것이다.
조건에 안 맞는 시계열은 결과에서 빠져서, 재시작을 겪은 컨테이너만 남는다.

## 실무에서는

Grafana Explore에서 조회했다. 설정 세 가지를 맞춰야 한다.

| 항목 | 값 | 이유 |
|------|-----|------|
| 시간 범위 | Last 7 days | `[7d]`와 맞춰야 한다. Last 1 hour인 채로 `[7d]`를 써도 1시간 구간만 평가된다 |
| Type | Instant | `max_over_time`이 이미 7일치를 값 하나로 압축한다. Range면 매 스텝마다 같은 계산을 반복한다 |
| Format | Table | Pod 하나당 한 줄로 떨어져 숫자 읽기가 편하다 |

Deployment 하나로 좁힐 때 Pod 이름 패턴에 함정이 있었다.

```promql
max_over_time(kube_pod_container_status_restarts_total{namespace="myns", pod=~"myapp-[^-]+-[^-]+"}[7d])
```

`pod=~"myapp-.*"`로 쓰면 `myapp-batch-*`처럼 접두사가 겹치는 다른 Deployment까지 딸려온다.
`-[^-]+-[^-]+`는 Deployment가 만드는 `<이름>-<ReplicaSet해시>-<Pod해시>` 형태에만 맞춘 패턴이다.
PromQL 정규식은 양끝이 자동으로 앵커링되므로 `^`, `$`는 안 붙여도 된다.

결과는 4가 나왔다. 해당 컨테이너가 지난 7일간 4번 재시작했다는 뜻이고,
Pod 자체가 4번 교체된 게 아니라 같은 Pod 안에서 컨테이너 프로세스만 4번 죽고 다시 떴다는 의미다.

### Pod의 생존 구간

기동 시각은 별도 메트릭에 있다.

```promql
max_over_time(kube_pod_start_time{namespace="myns", pod="myapp-xxxxxxxxx-xxxxx"}[7d])
```

값이 Unix epoch(초)라 그대로는 안 읽힌다. `date -r <epoch>`로 변환하거나
Grafana에서 Unit을 `Time > Datetime (from seconds)`로 바꾸면 된다.

더 편한 방법은 이 쿼리를 Range + Time series로 **그려보는 것**이다.
`kube_pod_start_time`은 값이 상수라 수평선으로 그려지는데,
선이 시작하는 지점과 끊기는 지점이 곧 Pod의 생존 구간이다. X축에 날짜가 바로 찍힌다.

숫자로 정확히 받아야 하면 subquery를 쓴다.

```promql
max_over_time(timestamp(kube_pod_info{namespace="myns", pod="myapp-xxxxxxxxx-xxxxx"})[7d:1m])
```

`[7d:1m]`은 "지난 7일을 1분 간격으로 되짚으며 안쪽 표현식을 평가하라"는 뜻이다.
`timestamp()`는 instant vector용이라 삭제된 Pod에 직접 못 쓰는데, 이 형태로 우회한다.
결과는 메트릭이 마지막으로 수집된 시각, 곧 Pod가 사라진 시각이다.
정밀도는 스크랩 주기 수준이다.

이 값이 현재 시각과 거의 같게 나오면 그 Pod는 아직 살아있는 것이다.

## 흔한 오해: 값이 1이면 한 번이 아니다

재시작 사유를 보려고 이 쿼리를 던졌다가 한 번 헷갈렸다.

```promql
max_over_time(kube_pod_container_status_last_terminated_reason{namespace="myns"}[7d]) > 0
```

결과 테이블의 Value가 1로 나왔다. "한 번 재시작했구나" 하고 읽었는데 틀렸다.

`kube_pod_container_status_last_terminated_reason`은 **값이 항상 1로 고정된 info 계열 메트릭**이다.
정보는 값이 아니라 `reason` 레이블(`OOMKilled` / `Error` / `Completed`)에 담긴다.
kube-state-metrics에는 이런 형태가 여럿 있다. `kube_pod_info`도 마찬가지로 값은 1이고 메타정보가 전부 레이블에 있다.

Table 뷰에서 Value 컬럼만 보면 숫자처럼 읽히니 주의할 지점이다.
횟수는 `restarts_total` 쪽에서 따로 봐야 한다.

이름이 `last_`인 것도 그대로 읽어야 한다. **마지막 종료 사유만** 남기 때문에,
여러 번 재시작하면서 사유가 바뀌었다면 중간 이력은 이 메트릭에 안 남는다.
처음엔 OOMKilled였다가 나중에 Error로 죽었다면 Error만 보인다.

## 남은 것

이 방법의 한계는 결국 Prometheus retention이다.
`--storage.tsdb.retention.time` 기본값이 15d라, 그 범위 밖은 이 방법으로도 못 본다.

아래는 이번에 직접 확인한 내용은 아니고, 알고 있던 내용이다.

Prometheus가 없는 클러스터라면 남는 건 이벤트 정도인데,
kube-apiserver의 `--event-ttl` 기본값이 1시간이라 그 안에서만 유효하다.
게다가 이벤트는 같은 사유가 반복되면 개별 레코드 대신 `count` 필드만 올라가서 횟수 추정용으로는 부정확하다.
노드의 `/var/log/pods/` 아래 회차별 로그 파일도 Pod가 삭제되면 kubelet이 정리한다.

## 더 볼 것

- [kube-state-metrics — Pod Metrics](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/pod-metrics.md)
- [Prometheus — Staleness](https://prometheus.io/docs/prometheus/latest/querying/basics/#staleness)
- [Prometheus — Subquery](https://prometheus.io/docs/prometheus/latest/querying/basics/#subquery)
