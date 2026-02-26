---
title: "HttpStatusCodeException 하나로 4xx/5xx 모두 처리하기"
date: 2026-02-25
categories: [spring]
tags: [spring, resttemplate, exception]
layout: single
toc: false
classes: wide
---

외부 API를 호출할 때 가장 자주 마주치는 고민 중 하나는 **예외 처리 전략**이었다.
특히 `RestTemplate` 기반으로 PG, 인증 서버, 내부 마이크로서비스 등을 호출하다 보면 4xx와 5xx 예외가 다양한 형태로 발생한다.

초기에는 다음처럼 예외를 모두 개별적으로 나눠 처리했었다.

```java
catch (HttpClientErrorException e) { ... }
catch (HttpServerErrorException e) { ... }
catch (RestClientException e) { ... }
```

하지만 프로젝트 규모가 커지고 외부 API 호출이 늘어나면서,
이 방식은 코드 중복과 가독성 저하라는 문제를 만들었다.

그래서 `HttpStatusCodeException` 하나로 예외 처리를 통합하는 패턴을 사용했다.

---

## HttpStatusCodeException이란 무엇인가

```java
catch (HttpStatusCodeException e)
```

`HttpStatusCodeException`은 Spring의 `RestClientResponseException` 계열 예외이며,
HTTP 상태 코드를 포함한 응답 정보를 그대로 담고 있는 것이 특징이다.

이 예외 하나만 catch하면 다음을 모두 처리할 수 있다.

* `HttpClientErrorException` (4xx)
* `HttpServerErrorException` (5xx)

즉, 세부 예외를 각각 나누지 않아도 된다.

계층 구조를 보면 이해가 쉽다.

```
RestClientException
 └── RestClientResponseException
      └── HttpStatusCodeException
           ├── HttpClientErrorException (4xx)
           └── HttpServerErrorException (5xx)
```

결국 대부분의 HTTP 응답 기반 에러는 이 하나로 커버된다.

---

## 왜 HttpStatusCodeException 하나로 처리했는가

### 1) 예외 클래스 분기가 줄어든다

외부 API 호출이 많은 서비스에서는 try-catch 블록이 반복적으로 등장한다.

개별 예외를 모두 나누면 이런 코드가 쉽게 만들어진다.

```java
catch (HttpClientErrorException e) {
    ...
}
catch (HttpServerErrorException e) {
    ...
}
```

문제는 로직 자체는 거의 동일한데 catch만 늘어난다는 점이다.

그래서 이렇게 단순화했다.

```java
catch (HttpStatusCodeException e) {
    ...
}
```

코드 라인이 줄어드는 것보다 중요한 건
**예외 처리의 의도가 더 명확해진다는 것**이다.

---

### 2) 상태 코드 기반 분기가 더 현실적이다

실제 운영에서는 예외 타입보다 상태 코드가 더 중요한 경우가 많다.

예를 들어:

* 4xx → 요청 데이터 문제, 인증 실패, 파라미터 오류
* 5xx → 외부 시스템 장애, 재시도 대상

그래서 대부분의 분기는 다음처럼 상태 코드 기준으로 처리하게 된다.

```java
if (e.getStatusCode().is4xxClientError()) {
    // 요청 문제
} else {
    // 서버 오류
}
```

예외 클래스를 나누는 것보다 훨씬 직관적이다.

---

### 3) 응답 Body까지 그대로 확인할 수 있다

`HttpStatusCodeException`의 큰 장점은 응답 정보를 그대로 보존한다는 점이다.

```java
e.getResponseBodyAsString()
e.getResponseHeaders()
e.getStatusCode()
```

특히 다음이 중요하다.

* PG 응답 에러 코드 파싱
* 내부 API errorCode 확인
* 장애 로그 전문 기록

예를 들어:

```java
log.error("external api error: status={}, body={}",
    e.getStatusCode(),
    e.getResponseBodyAsString()
);
```

별도 예외 변환 없이 바로 로그를 남길 수 있다.

---

## 자주 사용하는 처리 패턴

### 1) 상태 코드별 도메인 예외 변환

외부 예외를 그대로 던지기보다
서비스 내부 예외로 변환하는 경우가 많다.

```java
catch (HttpStatusCodeException e) {

    if (e.getStatusCode().is4xxClientError()) {
        throw new InvalidRequestException(e.getResponseBodyAsString());
    }

    throw new ExternalApiServerException(e);
}
```

이렇게 하면 컨트롤러 레이어에서는 HTTP 클라이언트 구현체에 의존하지 않는다.

---

### 2) 재시도 정책 분리

4xx와 5xx는 재시도 전략이 완전히 다르다.

* 4xx → 대부분 재시도 의미 없음
* 5xx → retry 대상

그래서 retry 로직을 붙일 때도 상태 코드 기반으로 판단한다.

```java
if (e.getStatusCode().is5xxServerError()) {
    retry();
}
```

---

### 3) 로그는 반드시 별도 남긴다

외부 API 장애는 원인 추적이 중요하다.

그래서 예외를 변환하더라도
응답 body는 최대한 그대로 남기는 것이 좋다.

단, 결제/개인정보 API는 반드시 마스킹이 필요하다.

---

## HttpStatusCodeException을 사용할 때 주의점

### 1) 네트워크 예외는 잡지 못한다

타임아웃이나 DNS 오류 같은 경우는
`ResourceAccessException` 등 다른 예외가 발생한다.

```java
catch (ResourceAccessException e) {
    // timeout, connection error
}
```

그래서 실제 운영 코드에서는 보통 이렇게 구성했다.

```java
catch (HttpStatusCodeException e) {
    ...
}
catch (ResourceAccessException e) {
    ...
}
```

---

### 2) 모든 5xx가 재시도 대상은 아니다

흔히 하는 실수가 “5xx니까 무조건 retry”다.

하지만 PG나 내부 API는

* 500이지만 이미 처리 완료된 케이스
* 타임아웃 후 성공한 케이스

도 존재한다.

그래서 retry는 반드시 **멱등 키**와 함께 설계해야 한다.

---

## 결론

외부 API 예외 처리는 예외 클래스를 얼마나 많이 아느냐보다
얼마나 단순한 구조로 유지하느냐가 더 중요했다.

`HttpStatusCodeException` 하나로 묶으면

* 코드가 단순해지고
* 상태 코드 기반 분기가 명확해지고
* 응답 전문까지 쉽게 다룰 수 있다.

특히 결제/인증/외부 연동이 많은 서비스에서는
예외 타입을 세분화하기보다

> “어떤 상태 코드인가”
> “재시도 대상인가”
> “도메인 예외로 어떻게 변환할 것인가”

에 집중하는 것이 운영 관점에서 훨씬 안정적이었다.

