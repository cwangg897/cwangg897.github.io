---
title: "Cloudflare Turnstile과 Redis RateLimiter로 회원가입 어뷰징 0건 만들기"
date: 2025-03-01 00:00:00 +0900
categories: [Security, Architecture]
tags: [Java, SpringBoot, Redis, Cloudflare, RateLimiter, AbuseDetection]
---

## 개요

신규 가입 보상(재화)을 노린 매크로 기반 자동 가입이 하루 1만 건 이상 발생했습니다.  
문제는 단순 가입 수치가 아니라, 생성된 계정이 특정 계정으로 보상을 집중시키면서 서비스 경제를 훼손한다는 점이었습니다.

이번 글에서는 다음 2가지를 결합해 가입 어뷰징을 원천 차단한 과정을 정리합니다.

- 네트워크 단: Cloudflare Turnstile 기반 봇/자동화 요청 선차단
- 애플리케이션 단: Redis TTL을 활용한 Sliding Window RateLimiter

핵심은 **단일 IP 차단이 아니라 복합 조건(IP + 닉네임 패턴)을 기준으로 악성 요청만 정밀 제어**한 것입니다.

---

## 문제 상황

가입 보상 정책이 고정되어 있던 시기에 매크로가 집중 유입되면서 다음 현상이 반복됐습니다.

- 특정 시간대에 가입 API 트래픽 급증
- 유사 닉네임 패턴 연속 생성
- 공용 네트워크와 유사한 IP 구간에서 정상/비정상 요청이 혼재

당시 운영에서 가장 어려웠던 점은 다음이었습니다.

1. IP만으로는 정상 사용자까지 함께 막게 됨
2. 전통적인 CAPTCHA만으로는 매크로의 우회 시도를 지속적으로 막기 어려움
3. 앱 레벨 제한만으로는 엣지(Edge) 단 부하를 충분히 줄이기 어려움

즉, 한 레이어만으로는 방어가 지속되지 않았습니다.

---

## 기존 방식의 한계: 왜 단일 IP 차단은 실패하는가

실제 공격은 프록시/분산 구간을 섞어 들어오기 때문에 "IP당 N회" 같은 단순 제한은 금방 우회됩니다.  
반대로 회사/카페 같은 공용 네트워크에서는 정상 가입도 같은 NAT IP로 보이기 때문에 오탐이 쉽게 발생합니다.

결론적으로 필요한 건 다음이었습니다.

- 차단 강도를 높여도 정상 사용자 경험은 유지
- 우회 비용을 올려 공격 효율을 급격히 낮춤
- 네트워크 단과 앱 단에서 서로 다른 신호를 결합

---

## 해결 전략: 다층 방어선

### Before / After

정리하면 구조 변화는 아래와 같습니다.

**Before**

- 회원가입 요청이 들어오면 앱에서 바로 처리
- 단순 IP 차단이나 개별 룰만으로 대응
- CAPTCHA만으로는 우회 시도가 계속됨
- 정상 사용자와 어뷰징 요청을 정밀하게 구분하기 어려움

**After**

- 브라우저에서 Turnstile 토큰 발급 후 서버 초반에서 유효성 검증
- 통과한 요청만 Redis Sliding Window RateLimiter로 2차 검사
- `nicknamePattern` 기준으로 반복 생성 패턴을 묶어 제한
- 필요한 경우에만 IP를 보조 기준으로 사용

즉, 단일 차단 규칙에 의존하던 구조를  
**Turnstile + Redis Sliding Window + 닉네임 패턴 기반 탐지**로 바꿔, 정상 사용자는 최대한 통과시키고 어뷰징 요청만 더 정밀하게 제어하도록 설계했습니다.

<img src="/assets/img/posts/ratelimiter/img_4.png" width="150%" alt="Cloudflare Turnstile 검증과 Redis RateLimiter 적용 흐름 다이어그램">

### 1) 1차 방어 - Cloudflare Turnstile

회원가입 요청은 애플리케이션에 도달한 뒤에도 바로 비즈니스 로직으로 들어가지 않도록 했습니다.  
서버 초반에서 Turnstile 토큰 유효성을 먼저 검증하고, 실패한 요청은 즉시 종료했습니다.

- 자동화 트래픽의 1차 제거
- 회원가입 비즈니스 로직 진입 전 조기 차단
- 자동화 요청이 실제 가입 처리까지 이어지는 비율 감소

요청 흐름은 단순합니다.

1. 브라우저에서 Cloudflare Turnstile을 통해 토큰을 발급받습니다.
2. 클라이언트는 회원가입 요청과 함께 이 토큰을 애플리케이션으로 전달합니다.
3. 애플리케이션은 회원가입 로직에 들어가기 전에 Turnstile 토큰 유효성을 먼저 검증합니다.
4. 이 1차 검증을 통과한 요청에 대해서 Redis Sliding Window RateLimiter로 2차 검사를 수행합니다.

즉, Turnstile은 입구에서 자동화 요청을 1차로 거르는 역할이고,  
RateLimiter는 통과한 요청 중에서도 짧은 시간 동안 반복되는 비정상 가입 패턴을 2차로 제어하는 역할입니다.

### 왜 다른 CAPTCHA가 아니라 Cloudflare Turnstile 이었는가

당시 저희는 이미 Cloudflare를 CDN/WAF 레이어로 사용하고 있었습니다.  
그래서 별도 CAPTCHA 솔루션을 새로 붙이는 것보다, 기존 인프라 안에서 바로 연동 가능한 Turnstile이 더 적합했습니다.

선택 이유는 단순했습니다.

- 이미 사용 중인 Cloudflare 환경과 자연스럽게 연결됨
- 전통적인 CAPTCHA처럼 기존 사용자까지 반복 인증시키는 불편이 상대적으로 적음
- 회원가입 로직 앞단에서 비정상 요청을 먼저 걸러 운영하기 쉬움

즉, Turnstile을 선택한 이유는 "가장 강력한 CAPTCHA"라서가 아니라,  
**기존 Cloudflare 기반 인프라와 가장 잘 맞고 사용자 경험 저하가 적었기 때문**입니다.

### 2) 2차 방어 - Redis Sliding Window RateLimiter

Turnstile 통과 요청에 대해 애플리케이션 레벨에서 Sliding Window 제한을 적용했습니다.

- 고정 윈도우의 경계 버스트 문제 완화
- Redis TTL로 윈도우 수명 자동 관리
- 요청 키를 복합 조건으로 구성해 정밀 차단

### 왜 다른 Rate Limiting 알고리즘이 아니라 Sliding Window 였는가

Rate Limiting 알고리즘은 대표적으로 `Fixed Window`, `Sliding Window`, `Token Bucket`, `Leaky Bucket` 같은 방식이 있습니다.  
이번 가입 어뷰징 방지 시나리오에서는 그중 `Sliding Window`가 가장 현실적인 선택이었습니다.

#### Fixed Window를 선택하지 않은 이유

`Fixed Window`는 구현이 가장 단순하지만, 윈도우 경계에서 요청이 몰리면 짧은 순간에 제한치를 우회하는 버스트가 발생할 수 있습니다.

예를 들어 "1분에 5회" 제한이라면

- `12:00:59`에 5회
- `12:01:00`에 다시 5회

처럼 거의 동시에 10회가 통과할 수 있습니다.

가입 어뷰징은 이런 경계 타이밍을 노리는 경우가 많아서, 단순 고정 윈도우는 방어력이 부족했습니다.

#### Token Bucket / Leaky Bucket을 최우선으로 두지 않은 이유

`Token Bucket`은 정상적인 순간 버스트를 어느 정도 허용하면서 전체 평균 속도를 제어하는 데 강점이 있습니다.  
`Leaky Bucket`은 일정한 속도로 요청을 흘려보내는 데 유리합니다.

하지만 이번 문제는 "짧은 시간 동안 비정상적인 가입 시도 자체를 얼마나 정밀하게 잘라낼 것인가"가 더 중요했습니다.  
즉, 트래픽을 부드럽게 평탄화하는 것보다 **최근 N초 동안 실제로 몇 번 시도했는지**를 더 정확하게 보는 편이 맞았습니다.

가입 API는 결제 API처럼 짧은 버스트를 적극 허용해야 하는 성격도 아니었기 때문에, 버스트 허용성이 강한 알고리즘보다 더 보수적인 제어가 필요했습니다.

#### Sliding Window를 선택한 이유

`Sliding Window`는 현재 시점을 기준으로 최근 요청 수를 계산하기 때문에, 고정 윈도우보다 경계 버스트에 강합니다.

이번 시나리오에서 특히 잘 맞았던 이유는 다음과 같습니다.

- 매크로가 짧은 시간 동안 반복 호출하는 패턴을 더 자연스럽게 포착할 수 있음
- 정상 사용자는 거의 영향 없이, 비정상적인 연속 가입 시도만 정밀하게 제한 가능
- `IP`, 닉네임 패턴 같은 여러 신호별 정책을 분리해 적용하기 쉬움

즉, 저희가 원했던 것은 "트래픽을 예쁘게 분산시키는 것"이 아니라  
**최근 시점 기준의 비정상 가입 시도를 민감하게 감지하고 바로 제어하는 것**이었고, 그 목적에 가장 잘 맞는 방식이 Sliding Window였습니다.

아래처럼 생각하면 이해가 쉽습니다.

<img src="/assets/img/posts/ratelimiter/img_3.png" width="50%" alt="Sliding Window 기반 요청 처리 흐름 다이어그램">

### 복합 조건은 하나의 긴 key보다 정책별 key로 나누는 편이 더 현실적이었다

처음에는 아래처럼 모든 조건을 하나의 key로 묶는 방식도 생각할 수 있습니다.

```text
signup:rl:{ip}:{nicknamePattern}
```

하지만 실제 구현은 이 방식보다, 요청 특성에 따라 대표 key를 하나 정해 Sliding Window를 적용하는 편이 더 현실적이었습니다.

예를 들면 닉네임 패턴이 명확하게 보이는 경우에는 다음과 같은 key를 사용할 수 있습니다.

```text
signup:rl:nickname:abc{NUM}
```

반대로 닉네임 패턴으로 묶기 어려운 경우에는 IP 기반 key로 fallback 할 수 있습니다.

```text
signup:rl:ip:1.2.3.4
```

즉, 하나의 요청에 대해 여러 key를 동시에 평가했다기보다,  
요청에서 가장 의미 있는 신호를 뽑아 대표 key를 만들고 그 기준으로 제한을 적용하는 방식에 가까웠습니다.

다만 IP 기반 key는 공용 네트워크 환경에서 오탐 가능성이 높기 때문에, 주요 차단 기준으로 사용하기보다 닉네임 패턴으로 묶기 어려운 요청에서만 보조적으로 사용했습니다.  
또한 IP 기준은 더 완화된 임계치를 적용해 정상 사용자가 함께 차단되지 않도록 조정했습니다.

`nicknamePattern`은 실제 공격 요청에서 반복적으로 관찰된 닉네임 규칙을 기준으로 분류한 값입니다.  
당시 어뷰징 계정은 `abc1`, `abc2`, `abc3`처럼 동일한 접두어 뒤에 숫자만 바뀌는 형태가 많았기 때문에, 숫자 구간을 정규화해 같은 패턴의 요청으로 묶었습니다.

예를 들면 다음과 같이 처리할 수 있습니다.

```text
abc1   -> abc{NUM}
abc2   -> abc{NUM}
abc999 -> abc{NUM}
```

---

## 구현 예시 (Spring Boot + Redis)

참고한 구현처럼, Sliding Window는 `INCR + EXPIRE`가 아니라 Redis `Sorted Set + Lua Script`로 처리하는 편이 정확합니다.

### 왜 Lua Script를 사용했는가

핵심은 `ZREMRANGEBYSCORE -> ZCARD -> ZADD -> EXPIRE`를 한 번에 처리해야 한다는 점이었습니다.  
이 과정을 애플리케이션에서 여러 Redis 명령으로 나누면 동시성 문제가 생길 수 있고, `multi/exec`를 사용하더라도 중간 결과를 바탕으로 계산하고 결과를 바로 반환하는 로직은 Lua Script가 더 자연스럽습니다.

그래서 Redis `pipeline`이나 `multi/exec`보다,  
**삭제 -> 개수 계산 -> 현재 요청 추가 -> TTL 갱신 -> 결과 반환까지를 한 번에 처리할 수 있는 Lua Script**가 더 적합했습니다.  
추가로 Redis 왕복 횟수도 줄일 수 있어 응답 지연과 구현 복잡도를 함께 낮출 수 있었습니다.

```java
@Service
@RequiredArgsConstructor
public class SignupRateLimiter {

    private final RedisTemplate<String, String> redisTemplate;
    private final DefaultRedisScript<Long> slidingWindowScript;
    private static final long LIMIT = 5;
    private static final long WINDOW_SECONDS = 60;

    public boolean allow(String policyKey, String nickname) {
        long nowMillis = System.currentTimeMillis();
        long windowMillis = WINDOW_SECONDS * 1000L;

        Long requestCount = redisTemplate.execute(
            slidingWindowScript,
            List.of("signup:rl:" + policyKey),
            String.valueOf(windowMillis),
            String.valueOf(nowMillis),
            nickname
        );

        return requestCount != null && requestCount <= LIMIT;
    }
}
```

```java
@Bean
public DefaultRedisScript<Long> slidingWindowScript() {
    String script = """
        local key      = KEYS[1]
        local window   = tonumber(ARGV[1])
        local now      = tonumber(ARGV[2])
        local nickname = ARGV[3]
        local boundary = now - window

        redis.call('ZREMRANGEBYSCORE', key, '-inf', boundary)

        local count = redis.call('ZCARD', key)
        redis.call('ZADD', key, now, nickname)

        local ttlSeconds = math.ceil(window / 1000)
        redis.call('EXPIRE', key, ttlSeconds)

        return count + 1
        """;

    DefaultRedisScript<Long> lua = new DefaultRedisScript<>();
    lua.setScriptText(script);
    lua.setResultType(Long.class);
    return lua;
}
```

여기서 핵심은 간단합니다.  
`policyKey`는 어떤 기준으로 요청을 묶을지 정하는 값이고, `nickname`은 ZSET에 저장되는 실제 요청 값입니다.

즉, `a1`, `a2`, `a3`처럼 서로 다른 닉네임이 들어와도 모두 `abc{NUM}` 같은 동일한 `policyKey`로 정규화되면 같은 제한을 적용받습니다.

운영에서는 하나의 기준만 보지 않고, 시나리오별로 정책을 나눠 적용했습니다.

- `nicknamePattern`이 명확하면 닉네임 패턴 기준으로 제한
- 닉네임 패턴으로 묶기 어렵다면 IP 기준으로 fallback
- 가입 성공 이후 보상 지급 API에도 별도 제한을 걸어 후속 어뷰징 차단

이렇게 하면 공용 네트워크의 정상 사용자는 최대한 통과시키고,
공격 패턴이 뚜렷한 요청은 닉네임 패턴 기준으로 더 정밀하게 제어할 수 있습니다.

---

## 결과

다층 방어 적용 후 다음 결과를 확인했습니다.

- 매크로/어뷰징 계정 생성: **일 1만 건 이상 -> 약 0건으로 수렴**
- 정상 가입 전환율: 유지

핵심은 "강하게 막는 것"이 아니라 **정상 흐름을 해치지 않으면서 악성 자동화만 정확히 비용 증가시키는 구조**를 만든 것입니다.

---

## 마치며

가입 어뷰징은 보통 하나의 기술로 끝나지 않습니다.  
엣지 단(Cloudflare)과 앱 단(Redis RateLimiter)을 분리해 방어 책임을 나누고, 단일 신호가 아닌 복합 신호로 판단해야 운영에서 오래 버틸 수 있습니다.

이번 작업을 통해 백엔드 개발에서도 기능 구현만이 아니라, 서비스 악용을 어떻게 탐지하고 제어할지까지 함께 설계해야 한다는 점을 분명히 배웠습니다.
