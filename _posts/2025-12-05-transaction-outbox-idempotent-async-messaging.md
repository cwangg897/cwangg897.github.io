---
title: "외부 API 장애를 격리하는 Transaction Outbox + 멱등 비동기 파이프라인 구축기"
date: 2025-12-5 00:00:00 +0900
categories: [Spring, Architecture]
tags: [Java, SpringBoot, TransactionOutbox, Async, Idempotency]
---

## 개요

업무 매칭 완료, 업무 제안 같은 주요 비즈니스 이벤트가 발생하면 알림톡 API를 호출해야 했습니다.  
초기에는 이벤트를 받아 비동기로 외부 알림 API를 호출하는 구조였지만, 장애 상황에서 전송 실패를 안정적으로 복구하지 못하는 문제가 있었습니다.

이번 글에서는 **Transaction Outbox 패턴**으로 비즈니스 트랜잭션과 알림 발송을 분리하고,  
워커 기반 재시도와 DB 제약조건을 이용해 **멱등성이 보장된 비동기 메시징 파이프라인**을 구축한 과정을 정리합니다.

---

## 문제 상황: 비동기였지만 신뢰성은 부족했던 구조

기존 구조는 다음과 같았습니다.
<img src="/assets/img/posts/transaction-outbox-idempotent-async-messaging/img.png" width="70%" alt="기존 비동기 알림 API 호출 구조 다이어그램">

정상 시에는 동작했지만, 외부 API 장애나 네트워크 문제에서 취약했습니다.

- 외부 알림 API 장애/타임아웃 발생 시 실패 이벤트 유실 가능
- 실패 감지 및 재처리 경로가 명확하지 않음
- 운영에서 "어떤 메시지가 왜 실패했는지" 추적 어려움

결과적으로 알림 누락이 월평균 20건 이상 발생했습니다.

---

## 해결 전략: Transaction Outbox + Worker 재시도 + 멱등성
<img src="/assets/img/posts/transaction-outbox-idempotent-async-messaging/img_1.png" width="70%" alt="Transaction Outbox 기반 비동기 메시징 파이프라인 구조 다이어그램">

### 핵심 원칙

- 비즈니스 상태 변경과 메시지 기록은 하나의 DB 트랜잭션에서 처리
- 외부 API 호출은 트랜잭션 밖의 별도 워커에서 처리
- 같은 알림이 Outbox에 중복 저장되지 않도록 DB 제약조건을 적용하고, 재시도 과정의 중복 실행 가능성은 ShedLock과 상태 관리로 완화

### 1단계: Transaction Outbox 도입

비즈니스 트랜잭션 커밋 시점에 Outbox 테이블에 이벤트를 함께 기록합니다.

```java
@Transactional
public void completeMatch(Long matchId) {
    matchService.complete(matchId); // 비즈니스 상태 변경

    outboxRepository.save(OutboxEvent.of(
            "MATCH_COMPLETED",
            "match:" + matchId,
            payloadJson
    ));
}
```

이렇게 하면 "비즈니스는 성공했는데 알림 이벤트는 사라지는" 문제가 크게 줄어듭니다.  
적어도 전송 대상 이벤트가 DB에 남기 때문에 복구가 가능합니다.

### 2단계: 워커 기반 비동기 발송 + 재시도

별도 워커가 Outbox에서 `PENDING` 상태를 배치로 읽어 외부 알림 API를 호출합니다.

```java
public void processOutboxEvent(OutboxEvent event) {
    try {
        alimtalkClient.send(event.getPayload());
        outboxRepository.markSent(event.getId());
    } catch (Exception ex) {
        outboxRepository.markRetry(event.getId(), ex.getMessage(), nextRetryAt(event));
    }
}
```

재시도 정책은 지수 백오프 기반으로 운영해 장애 시 외부 API를 과도하게 압박하지 않도록 했습니다.
재시도는 **최대 3회**로 제한했습니다.  
너무 많은 재호출은 외부 시스템 관점에서 비정상 트래픽으로 판단되어 IP 차단이나 공격성 요청으로 오인될 수 있기 때문입니다.

### 3단계: DB 제약조건 기반 멱등성 보장

같은 비즈니스 이벤트가 중복 유입되더라도 한 번만 전송되도록,  
`(업무제안 ID, 수신자 ID, 알림 타입)` 기준의 복합 UNIQUE 제약 조건을 적용했습니다.

```sql
ALTER TABLE outbox_event
ADD CONSTRAINT uk_outbox_notification UNIQUE (proposal_id, receiver_id, notification_type);
```

이렇게 하면 워커 재시도나 중복 소비 상황에서 같은 알림이 중복 저장되는 문제를 줄일 수 있습니다.  
다만 중복 전송 자체를 완전히 차단하는 수준까지 과하게 가져가지는 않았습니다. 알림 특성상 동일 메시지가 2번 전송되더라도 치명적인 데이터 오류로 이어질 가능성은 낮다고 판단했기 때문입니다.

---

## 설계하면서 고민했던 포인트

### 1. 중복 저장 방지와 재처리 통제

Outbox의 핵심은 메시지가 유실되지 않고, 실패 시 다시 처리할 수 있어야 한다는 점입니다.

- Outbox: `(업무제안 ID, 수신자 ID, 알림 타입)` 복합 UNIQUE 제약으로 중복 이벤트 생성 차단
- ShedLock: 하나의 인스턴스만 Outbox를 처리하게 해 중복 실행 가능성 완화
- 상태 관리: `PENDING`, `RETRY`, `SENT`, `FAILED` 상태를 기준으로 재처리 대상을 통제

### 2. Polling 부하와 지연 시간

Outbox 릴레이 방식은 Polling과 CDC 중에서 선택해야 했습니다.

- Polling: 구현이 단순하지만 주기에 따라 지연/DB 조회 부하가 발생
- CDC(Debezium 등): 실시간성과 효율은 좋지만 인프라 복잡도와 운영 난이도 증가

이번 프로젝트에서는 **회사 리소스와 운영 복잡도**를 고려해 스케줄링 Polling을 먼저 선택했습니다.  
또한 `ShedLock`을 적용해 여러 서버가 떠 있더라도 하나의 인스턴스만 Outbox 스케줄러를 수행하도록 했습니다.  
이 방식으로 중복 실행을 줄이면서도, 배치 크기와 인덱스, 실행 주기를 조정해 DB 부하를 통제했습니다.

### 3. Outbox 테이블 비대화와 정리 정책

Outbox는 영구 저장소가 아니라 전달 보장을 위한 임시 저장소에 가깝습니다.  
처리 완료 데이터가 계속 쌓이면 조회 성능과 용량 모두 악화됩니다.

그래서 별도 배치 잡으로 `SENT`/`FAILED(보관기간 경과)` 데이터를 주기적으로 정리하도록 했습니다.

- 단기: 주기 삭제(cleanup)
- 필요 시: 장기 보관 대상은 별도 스토리지로 이관

### 4. 메시지 순서 보장(Ordering)

주문 생성 -> 주문 취소처럼 순서가 중요한 이벤트는 발행 순서가 깨지면 문제가 됩니다.  
여러 서버가 동시에 스케줄러를 수행하면 같은 이벤트를 중복으로 읽거나, 처리 순서가 꼬일 가능성이 있습니다.

이를 줄이기 위해 다음 원칙을 적용했습니다.

- Outbox 조회 시 `created_at`, `id` 기준으로 정렬해 먼저 생성된 이벤트가 먼저 처리되도록 했습니다.
- `ShedLock`으로 하나의 인스턴스만 Outbox를 처리하게 해 중복 실행과 순서 꼬임 가능성을 줄였습니다.

---

## 운영 관점에서 달라진 점

- 실패 이벤트가 DB에 남아 조회/재처리 가능
- 이벤트별 상태(`PENDING`, `SENT`, `RETRY`, `FAILED`) 추적 가능
- 실패 사유/재시도 횟수/마지막 시도 시각을 기준으로 운영 대응 가능

즉, 비동기 구조를 유지하면서도 운영 가시성과 복구 가능성을 확보했습니다.

---

## 성과

- 외부 시스템 장애 시 내부 비즈니스 트랜잭션 보호
- 알림 파이프라인 결합도 감소 (도메인 로직 vs 외부 전송 분리)
- 알림톡 누락 문제: **월평균 20건 이상 -> 0건 달성**

---

## 마치며

비동기라는 이유만으로 시스템이 안정적인 것은 아니었습니다.  
실제로 중요한 것은 **실패를 기록하고, 재처리 가능하며, 중복 없이 복구할 수 있는 구조**였습니다.

Transaction Outbox 패턴과 멱등성 설계를 통해  
외부 API 장애를 내부에서 흡수하고, 결국 사용자에게는 "알림이 누락되지 않는 경험"을 제공할 수 있었습니다.

다만 중복 전송 방지까지 과하게 가져가지는 않았습니다.  
알림 도메인에서는 동일 메시지가 2번 전송되는 것이 치명적인 데이터 오류로 이어질 가능성은 상대적으로 낮다고 판단했고,
이번 설계에서는 무엇보다 **At Least Once 방식으로 누락 없이 전달되고, 재처리 가능한 구조**를 우선했습니다.
