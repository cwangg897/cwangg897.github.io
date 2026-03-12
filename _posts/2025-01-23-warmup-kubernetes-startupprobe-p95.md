---
title: "배포 직후 p95 급증 해결기: Warm-up + Kubernetes StartupProbe"
date: 2025-01-23 00:00:00 +0900
categories: [SpringBoot, Kubernetes]
tags: [SpringBoot, Kubernetes, Warmup, Performance, JVM, JIT]
---

## 문제 배경
사내 서비스 배포 직후, 초기 요청 구간에서 `p95`가 평소 대비 크게 높아지는 현상이 반복적으로 관측되었습니다.

평균 응답시간(`p50`)은 크게 나쁘지 않았지만, 실제 사용자 입장에서는 **첫 진입에서 느린 응답이나 타임아웃을 체감하는 문제**가 발생하고 있었습니다.

특히 초기 트래픽에서 다음 현상이 동반되었습니다.

- 응답시간 `p95`, `p99` 급증
- 일부 요청의 장시간 대기
- 드물지만 초기화 타이밍 이슈로 인한 오류 응답

이번 개선의 목표는 단순 평균 성능이 아니라 **배포 직후 첫 요청 품질을 안정화하는 것**이었습니다.

---

## 왜 콜드 스타트가 발생했나
원인을 점검해보니 첫 요청 구간에서 다음과 같은 초기화 비용이 집중되었습니다.

- JVM 클래스 로딩 및 JIT 워밍업
- Spring Bean 초기화 이후 실제 요청 경로에서 발생하는 지연 초기화
- DB / Redis / 외부 API 클라이언트의 커넥션 준비 비용

결국 **사용자 요청이 시스템 초기화 비용을 대신 지불하는 구조**였고, 이것이 배포 직후 `p95`를 끌어올리는 핵심 요인이었습니다.

---

## Warm-up 전략 설계 (JVM 관점)

콜드 스타트의 근본 원인을 살펴보면서 **JVM의 Tiered Compilation 특성**을 고려하게 되었습니다.

JVM은 메서드를 처음부터 최적화된 상태로 실행하지 않고 다음 단계로 점진적으로 최적화를 진행합니다.
```text
Interpreter → C1 Compiler → C2 Compiler
```


초기 요청 구간에서는 대부분의 코드가 **인터프리터 상태**로 실행됩니다.  
이 상태에서는 실행 속도가 상대적으로 느리고, 충분한 호출 횟수가 쌓인 뒤에야 **C1 → C2 최적화 컴파일**이 이루어집니다.

즉 배포 직후에는 다음 상황이 발생합니다.

- 비즈니스 로직 대부분이 아직 **JIT 최적화되지 않은 상태**
- 첫 사용자 요청이 **JVM 최적화 비용을 대신 지불**
- 결과적으로 초기 요청 latency가 급격히 증가

따라서 해결 방향은 명확했습니다.

> **사용자 요청 전에 JVM을 충분히 워밍업시키자**

이를 위해 **핵심 API를 미리 반복 호출하는 warm-up 전략**을 설계했습니다.

---

## Warm-up 반복 호출 횟수 설계

Warm-up에서 중요한 것은 **얼마나 호출해야 충분한가**입니다.

JVM은 호출 횟수 기반으로 최적화를 진행하기 때문에, 고정된 값으로 단정하기보다 **실측 기반으로 기준을 잡았습니다.**
실제로 HotSpot Tiered Compilation 기준에서 **Tier 1(C1) 컴파일은 메서드/옵션/JVM 버전에 따라 다르지만 약 1,500회 내외 호출 구간에서 관측**될 수 있습니다.

실무에서는 다음 기준으로 측정했습니다.

- 후보 API를 반복 호출하며 **p95 안정화 지점 확인**
- 컴파일 로그(`-XX:+PrintCompilation`)와 응답시간 추이를 함께 확인
- 과도한 호출로 배포 시간을 늘리지 않도록 상한 설정

테스트 결과 주요 경로는 대략 **1,500회 전후 호출 구간에서 성능이 안정화**되었습니다.

운영에서는 초기 변동성을 더 줄이기 위해 warm-up 반복 호출 기준을 **2,000회**로 보수적으로 상향해 적용했습니다.

---

## Warm-up 대상 API 선정 기준

모든 API를 warm-up 대상으로 삼으면 배포 시간이 과도하게 증가할 수 있습니다.

따라서 다음 기준으로 warm-up 대상 API를 선정했습니다.

- 배포 직후 실제 사용자 유입에서 **가장 먼저 호출되는 API**
- 트래픽 비중이 높아 `p95`에 직접 영향을 주는 API
- DB Query Path, 캐시 로딩, 외부 연동 등 초기화 비용이 큰 API

이 기준을 통해 **실제 사용자 트래픽 경로와 유사한 워밍업을 수행**하도록 설계했습니다.

### 선정 시나리오 (실사용 흐름 기준)

우리 서비스에서 가장 많이 발생하는 초기 사용자 흐름을 기준으로 warm-up 순서를 정했습니다.

1. 앱/웹 진입 직후: 홈 화면 데이터 조회
2. 목록 탐색: 업무목록/카테고리/리스트 조회
3. 상세 진입: 프리랜서 리스트 조회 (업무 요청 직전에 자주 사용)
4. 행동 직전: 사용자 상태/권한/포인트 등 부가 정보 조회

핵심은 "아무 API나 많이 호출"이 아니라, **실제 사용자 여정에서 초반에 자주 호출되는 API를 우선 워밍업**하는 것입니다.

---

## Spring 생명주기 기준으로 warm-up 실행 시점 확정

Warm-up을 언제 실행할지 결정하는 것도 중요한 문제였습니다.

Spring 생명주기를 기준으로 다음 옵션을 검토했습니다.

### `@PostConstruct`

- Bean 단위 초기화
- 전체 컨텍스트 준비 상태 보장 불가

### `CommandLineRunner`

- 대부분 초기화 이후 실행
- 하지만 운영 컴포넌트 준비 상태를 세밀하게 보장하기 어려움

### `ApplicationReadyEvent`

- 애플리케이션이 **요청 처리 준비 완료 상태**

최종적으로 **ApplicationReadyEvent**에서 warm-up을 시작하도록 선택했습니다.

```java
@Component
@RequiredArgsConstructor
public class WarmupRunner {

    private final WarmupService warmupService;

    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        warmupService.run();
    }
}
```

## Kubernetes Probe 기반 트래픽 차단 구조 설계

Warm-up을 수행하더라도 동시에 트래픽이 들어오면 의미가 없습니다.

따라서 Kubernetes Probe를 이용해 warm-up 완료 전에는 트래픽을 차단하는 구조를 설계했습니다.
핵심 요구사항은 다음과 같았습니다.

> `warm-up이 끝나기 전에는 외부 트래픽을 받지 않는다.`

이를 위해 `startupProbe`를 사용했습니다.

`startupProbe` 설계:
- `/warmup` endpoint 연결
- warm-up 완료 전까지 `503` 반환
- warm-up 완료 시 `200 OK` 반환

```java
@RestController
@RequiredArgsConstructor
public class WarmupController {

    private final WarmupState warmupState;

    @GetMapping("/warmup")
    public ResponseEntity<String> warmup() {
        if (!warmupState.isCompleted()) {
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body("warming up");
        }
        return ResponseEntity.ok("ok");
    }
}
```

`startupProbe`는 `/warmup`을 주기적으로 호출하고, warm-up 완료 후 `200 OK`가 반환되면 정상 기동 완료로 판단하도록 구성했습니다.

이렇게 하면 Kubernetes는 다음과 같이 동작합니다.

```text
Pod 시작
↓
startupProbe 실행
↓
warm-up 미완료 → 트래픽 차단
↓
warm-up 완료 → 서비스 트래픽 허용
```

이 구조 덕분에 사용자가 초기화 비용을 부담하지 않도록 만들 수 있었습니다.

---

## Before / After 구조

### Before (문제 구조)

배포 직후 Pod가 뜨자마자 트래픽이 유입되고, 첫 사용자 요청이 JVM/JIT/커넥션 초기화 비용을 직접 부담했습니다.

```text
Pod 시작
↓
즉시 트래픽 유입
↓
첫 요청에서 초기화 비용 발생
↓
p95/p99 급증 + 일부 오류/대기
```

### After (Warm-up 구조)

`ApplicationReadyEvent` 시점에 warm-up을 먼저 수행하고, `startupProbe`가 `/warmup` 200 OK를 확인한 뒤에만 트래픽을 허용하도록 변경했습니다.

```text
Pod 시작
↓
ApplicationReadyEvent
↓
핵심 API warm-up 반복 호출
↓
startupProbe(/warmup) 성공
↓
서비스 트래픽 유입
```

이 변경으로 초기화 비용은 사용자 요청이 아니라 배포 프로세스에서 선반영되도록 전환되었습니다.

---

## 운영 검증 및 효과

Warm-up 적용 전후를 동일 트래픽 구간으로 비교한 결과 다음과 같은 변화가 있었습니다.

- 배포 직후 초기 요청 구간의 `p95`가 유의미하게 감소
- 초기 요청에서 발생하던 장시간 대기 현상 해소
- 초기화 미완료 상태에서 유입되던 오류성 CS 사실상 제거

내부 측정 기준으로 초기 구간 평균 응답속도도 약 60% 이상 개선되었습니다.

특히 중요한 변화는 평균값보다 **배포 직후 초기 구간의 p95, p99가 안정화**되었다는 점이었습니다.

---

## 마치며

warm-up은 효과가 확실했지만, 배포 시간이 늘어나는 trade-off가 발생했습니다.

다음 단계에서는 아래를 추가로 검토할 예정입니다.

- `-XX:CompileThreshold`를 포함한 JVM 옵션 튜닝
- `-XX:+PrintCompilation`, JITWatch 기반 핫 메서드 분석
- 엔드포인트별 warm-up 횟수 차등화

많은 성능 최적화가 평균 응답시간을 기준으로 논의되지만,
실제 사용자 경험은 배포 직후 **초기 구간의 p95, p99**에 크게 좌우됩니다.

특히 배포 직후 구간은 관측되지 않기 쉬운 성능 공백 구간이며,
이 구간을 제어하는 것이 운영 안정성에서 중요한 요소라고 생각합니다.
