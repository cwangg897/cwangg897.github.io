---
title: "Redisson 세마포어로 선착순 매칭 병목 해소하기"
date: 2025-10-31 00:00:00 +0900
categories: [Spring, Architecture]
tags: [Java, SpringBoot, Redis, Redisson, MySQL, Concurrency]
---

## 개요

프리랜서 매칭 서비스에서 고객의 업무 요청이 올라오면, 여러 프리랜서가 같은 건을 선착순으로 수락합니다.  
이때 트래픽이 순간적으로 몰리면 동일한 업무 요청 행(Row)에 락 경합이 집중되면서 응답 지연이 급격히 증가했습니다.

이번 글에서는 **DB 락에만 의존하던 구조를 Redis 기반 분산 세마포어(Redisson `RPermitExpirableSemaphore`)로 보강**해  
트랜잭션 진입 전 부하를 차단하고, DB에서는 최종 정합성만 검증하도록 바꾼 과정을 정리합니다.

---

## 문제 상황: 단일 Row 락 경합으로 인한 병목

기존 구조는 아래와 같았습니다.
<img src="/assets/img/posts/semaphore/img_2.png" width="50%" alt="기존 DB 락 기반 선착순 수락 처리 구조 다이어그램">

선착순 시나리오 특성상 수백 명이 같은 업무 요청에 동시에 접근하면  
대부분의 요청이 DB 락 대기열에서 대기하게 됩니다.

결과적으로 다음 문제가 발생했습니다.

- 트랜잭션 대기 시간 증가
- DB 커넥션 점유 시간 증가
- 사용자 체감 실패율 증가(타임아웃, 지연으로 인한 중복 시도)

핵심 원인은 **"모든 경쟁을 DB 내부에서만 해결하려고 한 점"** 이었습니다.

---

## 해결 전략: Redis 세마포어 + DB 최종 검증(2단계)

개선 구조는 아래와 같습니다.
<img src="/assets/img/posts/semaphore/img_3.png" width="80%" alt="Redis 세마포어 기반 2단계 검증 처리 구조 다이어그램">


### 설계 목표

- 트랜잭션 진입 전 과도한 요청을 애플리케이션 레벨에서 차단
- DB는 최종 정합성 보장 역할에 집중
- 분산 환경에서도 안전하게 동작

### 1단계: Redis 분산 세마포어로 선제적 진입 제어

요청마다 먼저 `RPermitExpirableSemaphore.tryAcquire()`를 수행해 "진입권"을 얻은 요청만 DB 트랜잭션으로 보냅니다.

예시코드 - 회사코드를 완벽하게 공개하진않았습니다
```java
public MatchAcceptResult accept(Long requestId, Long freelancerId) {
    String key = "match:accept:semaphore:" + requestId;
    RPermitExpirableSemaphore semaphore = redissonClient.getPermitExpirableSemaphore(key);
    
    String permitId = semaphore.tryAcquire(100, 300, TimeUnit.SECONDS);
    if (permitId == null) {
        return MatchAcceptResult.rejected("동시 요청이 많아 잠시 후 다시 시도해주세요.");
    }

    try {
        return acceptInTransaction(requestId, freelancerId); // DB 최종 검증
    } finally {
        semaphore.release(permitId);
    }
}
```

포인트는 **락을 DB까지 끌고 가지 않고, 먼저 트래픽을 거르는 것**입니다.

### 왜 `RLock`이 아니라 세마포어를 선택했는가
`RLock`은 사실상 단일 임계구역을 순차 처리하는 모델에 가깝습니다.
<img src="/assets/img/posts/semaphore/img.png" width="100%" alt="RLock 기반 단일 임계구역 순차 처리 개념 다이어그램">

이번 케이스는 "한 명만 통과"가 아니라 "정해진 수만큼 동시 진입 허용"이 필요한 문제였기 때문에
슬롯 기반 제어가 가능한 세마포어가 더 적합했습니다.
<img src="/assets/img/posts/semaphore/img_1.png" width="100%" alt="세마포어 슬롯 기반 동시 진입 제어 개념 다이어그램">

### 왜 `RPermitExpirableSemaphore`를 최종 선택했는가

비정상 종료(OOM, 강제 종료, 네트워크 단절) 상황에서는 `release`를 호출하지 못해 permit 누수가 발생할 수 있습니다.  
`RPermitExpirableSemaphore`는 permit에 `leaseTime`을 부여할 수 있어, 이런 경우에도 시간이 지나면 자동 만료되어 회복됩니다.

### 2단계: DB에서 최종 정합성 검증

세마포어를 통과했더라도, 최종 승인 여부는 DB에서 확인합니다.

- 이미 마감된 요청인지
- 이미 다른 프리랜서가 수락했는지
- 상태 전이가 유효한지

즉, Redis는 **부하 제어**, MySQL은 **정합성의 단일 소스(Single Source of Truth)** 역할을 담당합니다.

---

## 분산 환경 안정성: 누수/데드락 방지

누수 방지는 설계에서 가장 중요했습니다. 다음을 적용했습니다.

1. `tryAcquire` 성공 후 DB 트랜잭션에서 예외가 발생해도 `finally`에서 `release(permitId)`를 보장했습니다.
2. 요청 타임아웃/서버 장애 같은 비정상 종료 케이스를 고려했습니다.  
   permit에 `leaseTime`을 두어, `release` 누락 시에도 자동 만료되도록 설계했습니다.
3. 운영에서는 `availablePermits`/`usedPermits` 지표를 메트릭으로 노출해 누수 의심 상태를 탐지할 수 있게 했습니다.

### permit을 "남은 좌석"과 1:1로 맞추지 않은 이유

처음에는 permit 개수를 남은 좌석과 항상 동일하게 맞추는 방안을 검토했습니다.  
하지만 `remaining(최종 상태)`는 DB 트랜잭션 커밋 시점에만 확정됩니다.

이때 permit을 실시간으로 남은 좌석과 동기화하려고 하면 커밋 전/중 상태와 경쟁(race)하게 되어, 다음 문제가 발생할 수 있습니다.

- 초과 허용(oversell)
- 과도 차단(under-utilization)

그래서 Redis permit은 "남은 좌석"이 아니라 **동시 진입 제한 수(게이트)** 로만 사용하고,  
최종 좌석 확정은 DB 트랜잭션에서만 수행하도록 역할을 분리했습니다.

---

## 부하 테스트 결과 (K6, 동시 500명)

순간 트래픽 급증 구간을 가정해 K6로 동시 요청 500명 시나리오를 검증했습니다.

- p95 응답 시간: **30ms 이내 유지**
- 순간 트래픽 급증 시 지연 현상: **해소**

특히 효과가 컸던 지점은  
**DB 부하를 애플리케이션 바깥(Redis 레이어)으로 격리해 락 대기열 자체를 줄인 것**이었습니다.

## 운영 결과

운영 환경에서는 "매칭 수락 실패" 관련 CS가 **0건**이었고,  
마감/선착순 구간의 체감 지연 이슈도 크게 줄었습니다.

---

## 마치며
선착순/마감형 트래픽에서는  
DB를 마지막 검증 지점으로 남기고, 앞단에서 진입량을 제어하는 구조가 훨씬 안정적이라는 것을 확인할 수 있었습니다.

또한 이번 작업을 통해 graceful shutdown이 아닌 종료(OOM, 프로세스 강제 종료, 네트워크 단절) 상황까지 고려해야 한다는 점을 분명히 배웠습니다.  
세마포어는 획득만큼 `release` 설계가 중요하며, `finally` 해제와 permit 만료 전략을 함께 가져가야 실제 운영에서 안정적으로 동작한다는 점을 확인했습니다.
