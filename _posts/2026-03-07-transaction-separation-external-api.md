---
title: "최종적 일관성과 트랜잭션 분리를 통한 외부 API 호출 격리: 주문&결제 아키텍처 개선기"
date: 2026-03-07 00:00:00 +0900
categories: [Spring, Architecture]
tags: [SpringBoot, Transaction, CAP]
---

## 개요
기프티콘 주문 시스템을 구축하며 가장 큰 고민은 **"우리 시스템의 DB 트랜잭션과 외부 API(KT 기프티쇼)의 상태를 어떻게 동기화할 것인가?"**
였습니다. <br>
네트워크 지연이나 외부 서버 장애가 우리 서비스의 전체 장애로 번지지 않도록 
아키텍처를 개선한 과정을 공유합니다.

## 문제 상황: 모든 것을 하나의 트랜잭션으로 묶었을 때의 리스크

초기 설계는 단순했습니다. 모든 과정을 하나의 `@Transactional`로 묶어 원자성(Atomicity)을 보장하려 했습니다.

### 기존 구조 (Before)
다음은 초기 구조(단일 트랜잭션 + 외부 API 호출 포함)입니다. <br> <br>
<img src="/assets/img/posts/transaction-separation-external-api/img_2.png" width="80%" alt="Before 주문 처리 흐름 다이어그램"> <br>

```java
@Transactional 
public GifticonOrderCreateResponse execute(GifticonOrderCreateRequest request, Long userSeq) {
    // 1. 주문 생성 (DB 저장)
    GifticonOrder gifticonOrder = gifticonOrderService.initOrder(request, userSeq);

    // 2. 외부 API 호출 (KT 기프티쇼 발급 요청)
    GifticonPaymentResponse response = gifticonOrderService.requestPayment(gifticonOrder);

    // 3. 결과 업데이트 (DB 반영)
    gifticonOrderService.savePaymentResult(gifticonOrder, response);

    return new GifticonOrderCreateResponse(gifticonOrder.getId(), response.isSuccess(), response.getMessage());
}
```

이 구조의 치명적인 결함
1. DB 커넥션 풀 고갈: 외부 API 응답이 5초 걸리면, DB 커넥션도 5초간 점유됩니다. 트래픽이 몰릴 때 외부 지연이 발생하면 순식간에 Connection Timeout이 발생하며 결제와 상관없는 다른 서비스(공지사항, 마이페이지 등)까지 마비됩니다.
2. 원자성(Atomicity)의 역설: 외부 API 호출은 성공했는데, 이후 DB 저장 단계에서 예외가 발생하면 주문 데이터는 롤백됩니다. 하지만 기프티콘은 이미 외부에서 발급된 상태가 되어 실질적인 데이터 불일치가 발생합니다.


## 해결 전략: 트랜잭션 분리
외부 API 호출을 트랜잭션 밖으로 밀어내어 DB 점유 시간을 최소화하고, 단계를 분리했습니다.
외부 시스템은 우리가 제어할 수 없는 영역이기 때문에 트랜잭션 경계 안에 포함시키기보다는  
**상태 기반(State-driven) 설계를 사용하는 것이 더 적절한 접근이라고 생각했습니다.**


### 개선된 구조 (After)
주문 처리 흐름은 다음과 같이 세 단계로 분리됩니다.
<img src="/assets/img/posts/transaction-separation-external-api/img_1.png" width="80%" alt="After 주문 처리 흐름 다이어그램">

```java
public GifticonOrderCreateResponse execute(GifticonOrderCreateRequest request, Long userSeq) {
    // [Phase 1] 주문 생성: 별도 트랜잭션으로 커밋 완료 (주문 상태: PENDING)
    GifticonOrder gifticonOrder = gifticonOrderService.initOrder(request, userSeq);

    // [Phase 2] 외부 API 호출: 트랜잭션 없이 순수 네트워크 통신 (DB 커넥션 미점유)
    GifticonPaymentResponse response = gifticonOrderService.requestPayment(gifticonOrder);

    // [Phase 3] 결과 저장: 새로운 트랜잭션을 열어 상태 업데이트 (주문 상태: COMPLETED/FAILED)
    gifticonOrderService.savePaymentResult(gifticonOrder, response);

    return new GifticonOrderCreateResponse(gifticonOrder.getId(), response.isSuccess(), response.getMessage());
}
```

### 트레이드오프: 강력한 일관성 vs 최종 일관성
트랜잭션을 분리하면 시스템은 강한 일관성(Strong Consistency)을 포기하는 대신
**최종적 일관성(Eventual Consistency)** 을 선택하게 됩니다.

이는 주문 저장과 외부 API 결과 반영이 하나의 트랜잭션으로 묶여 있지 않기 때문에
더 이상 **All-or-Nothing** 보장이 이루어지지 않기 때문입니다.

`API 호출은 성공했는데 결과 반영 트랜잭션에서 에러가 난다면?`
이 과정에서 DB 상태와 외부 시스템 상태가 일시적으로 서로 다른 **불일치 구간(Inconsistency Window)** 이 발생할 수 있습니다.

| 구분 | 강한 정합성 (Before) | 최종적 일관성 (After) |
|-----|-------------------|--------------------|
| 정합성 | DB와 외부 상태가 단일 트랜잭션으로 항상 일치 (Atomic) | 일시적으로 DB는 PENDING, 외부는 발급완료인 상태 존재 |
| 가용성 | 외부 장애 시 내 트랜잭션도 실패 (낮은 가용성) | 외부 장애 시에도 내 주문 데이터는 안전하게 저장 (높은 가용성) |
| 지연 시간 | API 응답 시간만큼 사용자가 대기함 | 사용자는 빠르게 응답받고 결과는 나중에 확인 |


### 최종적 일관성을 위한 후속 조치: 스케줄러 재처리
트랜잭션을 분리했을 때 가장 위험한 시나리오는 **"기프티콘 발급은 성공했으나, 우리 DB에는 결과가 반영되지 않은 경우"** 입니다. 
이를 해결하기 위해 주기적인 배치 스케줄러를 도입했습니다.

재처리 로직의 흐름
1. 대상 조회: 생성된 지 일정 시간(예: 5분)이 지났음에도 여전히 PENDING 상태인 주문을 조회합니다.
2. 외부 상태 확인: 외부 API(KT 기프티쇼)의 주문 단건 조회 API를 호출하여 실제 발급 여부를 확인합니다.
3. 상태 동기화:
   - 외부에선 성공했다면? 우리 DB 상태를 COMPLETED로 변경합니다.
   - 외부에 기록이 없다면? 우리 DB 상태를 CANCEL로 변경하여 사용자에게 알립니다.

즉, 시스템이 일시적으로 불일치 상태에 빠질 수는 있지만 시간이 지나면 자동으로 정상 상태로 수렴하도록 설계했습니다.


### 마치며: 외부 API 연동에서 얻은 깨달음
이번 기프티콘 결제 연동은 단순히 기능을 구현하는 것을 넘어,
외부 시스템과 연동할 때 어떤 아키텍처를 선택해야 하는지 고민해 볼 수 있는 경험이었습니다.<br>

특히 다음 세 가지 관점을 깊게 고민해 볼 수 있었습니다.<br>
- 외부 시스템은 우리가 제어할 수 없는 영역이기 때문에, 트랜잭션 경계 안에 포함시키지 않는 설계가 왜 중요한지 체감했습니다.
- 트랜잭션을 분리하는 과정에서 강한 일관성 대신 **최종적 일관성(Eventual Consistency)** 을 선택해야 하는 상황과 그에 따른 트레이드오프를 직접 경험했습니다.
- 외부 API 호출 결과와 내부 데이터 상태가 일시적으로 어긋날 수 있기 때문에, 이를 복구하기 위한 **재처리 스케줄러(Self-Healing 구조)** 가 필요하다는 점을 이해할 수 있었습니다.

정상적인 흐름뿐 아니라 **실패 상황을 어떻게 처리할 것인지 설계하는 것이
백엔드 시스템의 안정성을 높이는 데 중요한 요소라는 점을 다시 한번 느낄 수 있었습니다.**
