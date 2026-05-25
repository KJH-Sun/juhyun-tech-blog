---
layout: post
title: "gRPC + Service Mesh 환경에서 만난 세 가지 Long-lived Connection 문제"
description: "무중단 배포, connection skew, fan-out 병목을 하나의 long-lived connection 관점으로 다시 정리한 운영 기록"
date: 2026-06-08
tags:
  - grpc
  - istio
  - envoy
  - kubernetes
  - performance
---

gRPC와 service mesh를 함께 쓰는 시스템을 운영하다 보면, 얼핏 서로 관계없어 보이는 문제들이 비슷한 시기에 튀어나올 때가 있다.

내가 경험한 것도 그랬다.

- rollout 중 간헐적인 `UNAVAILABLE` 에러
- 일부 Pod에만 트래픽이 몰리는 connection skew
- fan-out 요청에서 비정상적으로 높아지는 지연 시간

처음에는 각각 따로 해결해야 하는 문제처럼 보였다. 하나는 배포 문제였고, 하나는 load balancing 문제였고, 또 하나는 성능 문제였기 때문이다.

그런데 실험을 진행할수록 세 문제를 하나로 묶는 공통 배경이 보이기 시작했다. 핵심은 **long-lived connection**이었다. gRPC의 HTTP/2 연결은 오래 살아 있고, service mesh는 그 연결 위에서 endpoint를 선택하고 유지한다. 이 특성이 rollout, 분산, 성능에 연쇄적으로 영향을 줬다.

이번 글은 그 세 문제를 각각의 사건으로 기록하기보다, 하나의 운영 패턴으로 묶어 정리한 내용이다.

## 문제 1: rollout은 성공했는데, 요청은 흔들렸다

첫 번째 문제는 무중단 배포가 기대만큼 "무중단"이지 않다는 점이었다.

애플리케이션은 graceful shutdown을 하도록 설정했는데도, rollout 시점에 gRPC `UNAVAILABLE`가 간헐적으로 보였다. 애플리케이션 로그만 보면 종료가 천천히 진행되는 것처럼 보였지만, 실제 네트워크 경로에서는 더 빨리 무언가가 사라지고 있었다.

이 문제를 이해하려면 애플리케이션만 보면 안 됐다. service mesh sidecar까지 포함한 종료 순서를 같이 봐야 했다.

### 실제로 문제가 되었던 것

- app보다 sidecar가 먼저 연결 정리를 끝내는 경우
- endpoint 제거가 전파되기 전에 종료 중인 Pod로 트래픽이 한 번 더 들어오는 경우
- 에러가 난 endpoint를 mesh가 바로 격리하지 못하는 경우

즉, 애플리케이션 하나의 graceful shutdown만으로는 충분하지 않았다. **Kubernetes lifecycle, service registry 반영 타이밍, sidecar drain, client retry**가 함께 맞물려야 했다.

### 여기서 얻은 첫 번째 규칙

종료 관련 설정은 독립 값이 아니라 시간 관계로 봐야 한다.

- preStop은 endpoint 제거가 전파될 시간을 벌어야 한다.
- sidecar drain 시간은 preStop보다 짧으면 안 된다.
- application shutdown timeout은 drain보다 충분히 길어야 한다.
- Kubernetes 강제 종료 시간은 이 모든 것의 마지막 안전망이어야 한다.

이걸 식으로 쓰면 대략 이런 느낌이다.

```text
preStop < sidecar drain < app graceful shutdown < pod termination grace period
```

이 순서가 어긋나면, rollout은 성공해도 요청 단에서는 미세한 흔들림이 발생하기 쉽다.

## 문제 2: 연결을 끊어도 트래픽이 다시 같은 Pod로 갔다

두 번째 문제는 더 이상했다. 요청 분포를 보니 일부 상황에서 트래픽이 특정 Pod에 거의 고정됐다. 처음에는 gRPC의 장기 연결 특성 때문이라고 생각했다. 연결이 하나 오래 유지되면 그 연결을 잡은 Pod가 계속 요청을 받는 것은 자연스러워 보였기 때문이다.

그래서 가장 먼저 시도한 것은 연결 rotation이었다.

- 요청 수 기준으로 연결을 주기적으로 끊기
- 서버에서 일정 시간이 지나면 GOAWAY를 보내 재연결 유도하기

이 접근은 일부 상황에서는 효과가 있었다. 하지만 Pod 수를 늘리거나 zone 구성이 달라지면 다시 skew가 나타났다. 여기서 중요한 단서가 나왔다.

**문제는 연결이 오래 살아서만이 아니라, 재연결할 때도 같은 locality 집합으로 돌아간다는 점**이었다.

즉, 연결을 끊는 것 자체는 재분배를 보장하지 않았다. service mesh의 locality-aware load balancing 정책이 특정 endpoint 집합을 우선하게 만들면, 새 연결도 다시 그 집합 안에서 맺어질 수 있다. 장기 연결은 그 편중을 더 오래 고착시키는 역할을 했다.

### 여기서 얻은 두 번째 규칙

long-lived connection 문제를 볼 때는 "연결을 얼마나 자주 끊을까"만 보면 부족하다. 먼저 봐야 할 것은 **재연결 시 endpoint selection이 실제로 분산되는가**다.

내 경우에는 locality-aware distribution을 명시적으로 조정한 뒤에야 skew가 크게 줄었다. 그 다음에 connection rotation은 보조 수단으로 의미가 생겼다.

순서를 바꾸면 안 된다.

1. endpoint selection 정책 확인
2. locality 또는 priority 편향 확인
3. 그 다음에 connection rotation이나 max connection age 조정

이 순서가 맞아야 한다.

## 문제 3: 오류는 없는데도 fan-out 요청이 느렸다

세 번째 문제는 성능이었다. 특정 API는 내부적으로 downstream gRPC를 매우 많이 fan-out하고 있었다. 기능적으로는 맞았고, 오류도 없었다. 그런데 지연 시간이 예상보다 높았다.

이 문제도 처음에는 단순히 "호출 수가 많아서 느리다" 정도로 생각하기 쉬웠다. 하지만 실제로는 조금 더 미묘했다.

장기 연결 하나 위에 많은 stream이 몰리면 packet density가 높아지고, service mesh proxy의 event loop가 그 부하를 직렬화된 형태로 받는 순간이 생긴다. 특히 fan-out이 크면 응답 자체보다 네트워크 경로와 중간 프록시의 처리 방식이 더 빨리 병목이 된다.

이때 연결 수를 조금 조정하는 것만으로는 충분하지 않았다. 더 효과적이었던 것은 요청 형태를 바꾸는 일이었다.

- 1000번 호출하는 패턴을 유지할 것인가
- 배치 API로 묶어 1번 호출로 바꿀 것인가

결과적으로 후자가 훨씬 컸다. end-to-end latency가 크게 줄었고, 문제를 단순 성능 튜닝이 아니라 **호출 형태 재설계**로 보는 것이 맞다는 점을 확인했다.

### 여기서 얻은 세 번째 규칙

mesh 환경에서 fan-out 병목을 보면 connection pool 값만 먼저 만지기 쉽다. 하지만 fan-out 자체가 과하면, 가장 큰 이득은 대개 **더 많은 튜닝이 아니라 호출 수를 줄이는 방향**에서 나온다.

## 세 문제를 하나로 묶는 공통 원인

세 문제는 각각 다른 층위에서 발생했지만, 공통 배경은 분명했다.

### 1. 연결은 생각보다 오래 산다

HTTP/1.1 기반 시스템에서는 요청마다 재분배 기회가 상대적으로 많다. 하지만 gRPC의 HTTP/2 연결은 한 번 맺으면 오래 유지되고, 많은 요청이 같은 연결을 타고 간다.

### 2. endpoint 선택은 연결 생성 시점의 영향을 크게 받는다

연결이 오래 살수록 "처음 누구를 골랐는가"의 영향이 커진다. locality-aware policy, priority, health 상태 같은 요소가 이 시점에 큰 의미를 가진다.

### 3. proxy는 단순 전달자가 아니다

service mesh proxy는 트래픽을 그냥 중계만 하지 않는다. drain, outlier detection, locality-aware balancing, connection pool 동작이 모두 실질적인 시스템 행태를 바꾼다.

그래서 rollout 문제, skew 문제, fan-out 성능 문제를 각자 따로 보지 않고, **long-lived connection 위에서 proxy가 어떤 선택을 하는가**라는 하나의 질문으로 묶어 보는 편이 더 잘 풀렸다.

## 실무에서 유효했던 대응 패턴

이 경험을 다시 정리하면, 대응은 세 갈래였다.

### 1. Shutdown timing을 설계한다

- preStop, readiness removal, sidecar drain, graceful shutdown의 순서를 함께 본다.
- drain은 app shutdown보다 먼저 끝나면 안 된다.
- retry는 안전망일 뿐, 종료 경로의 설계 실패를 대신해 주지 않는다.

### 2. Reconnection보다 endpoint selection을 먼저 본다

- skew가 보이면 먼저 locality와 priority를 의심한다.
- connection rotation은 분산 정책이 맞게 작동할 때만 보조적으로 유효하다.

### 3. Fan-out은 설정이 아니라 구조 문제로 본다

- downstream 호출 수를 줄일 수 있는지 먼저 검토한다.
- batch API, aggregation, cache 재설계가 pool tuning보다 큰 효과를 낼 수 있다.

## 운영 체크리스트

비슷한 환경을 운영한다면 아래 체크리스트부터 보는 편이 좋다.

### Rollout

- app graceful shutdown만 보고 있지 않은가
- sidecar drain 시간이 preStop보다 충분히 긴가
- readiness 제거와 endpoint 전파 시간을 고려했는가
- retry가 문제를 숨기고만 있지는 않은가

### Load Distribution

- skew가 connection lifetime 문제인지, selection policy 문제인지 분리했는가
- locality-aware policy가 특정 집합에 과도한 우선순위를 주고 있지 않은가
- reconnect가 실제 분산으로 이어지는지 메트릭으로 확인했는가

### Performance

- high fan-out 요청이 proxy에서 density 문제를 만들고 있지 않은가
- connection pool 튜닝 전에 호출 수 자체를 줄일 수 있는가
- batch API로 바꾸면 병목을 더 크게 줄일 수 있지 않은가

## 마무리

운영 문제는 자주 증상 단위로 들어온다. 배포 에러, skew, 성능 저하처럼 각각 다른 이름으로 올라온다. 하지만 그 밑바닥을 따라가 보면, 같은 구조적 특성이 여러 형태로 나타난 경우가 있다.

gRPC + service mesh 환경에서 내게 그 공통점은 long-lived connection이었다.

이 관점을 얻고 나니 해결 순서도 달라졌다. rollout은 종료 타이밍의 문제로, skew는 endpoint selection의 문제로, fan-out 지연은 호출 구조의 문제로 다시 분리해서 볼 수 있었다. 그리고 각각의 설정값보다 먼저, 시스템이 연결을 어떻게 만들고 얼마나 오래 유지하며 어디로 보내는지를 이해하는 쪽이 훨씬 중요하다는 점을 배웠다.

비슷한 현상을 겪고 있다면, 문제를 세 개로 나누기 전에 먼저 하나의 질문을 던져 보면 좋다.

`이 시스템의 long-lived connection은 지금 어디에서, 어떤 방식으로, 얼마나 오래 영향을 미치고 있는가?`

대개 해답의 절반은 그 질문 안에 들어 있다.
