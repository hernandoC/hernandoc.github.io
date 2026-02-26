---
title: "HttpStatusCodeException 하나로 4xx/5xx 모두 처리하기"
date: 2026-02-25
categories: [spring]
tags: [spring, resttemplate, exception]
---

외부 API 호출 시 예외 처리를 단순화하기 위해 HttpStatusCodeException을 자주 사용했다.

```java
catch (HttpStatusCodeException e)
```

이 예외는 다음을 모두 포함한다.

* HttpClientErrorException (4xx)
* HttpServerErrorException (5xx)

따라서 세부 예외를 각각 catch하지 않아도 된다.

실무에서는 상태 코드만 분기해서 처리하는 경우가 많았다.

```java
if(e.getStatusCode().is4xxClientError()){
    // 요청 문제
}else{
    // 서버 오류
}
```

예외 클래스를 줄이면 코드 가독성이 훨씬 좋아진다.
