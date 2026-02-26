---
title: "LEAD 함수 사용"
date: 2026-02-23
categories: [sql-tuning]
tags: [oracle, sql, window-function]
layout: single
toc: false
classes: wide
---

게시판 이전글/다음글을 구현하면서 LEAD 함수를 많이 사용했다.

```sql
LEAD(ID) OVER (ORDER BY ID DESC)
```

초기에는 함수 내부에 조건을 넣으려고 했지만
실제로는 서브쿼리에서 데이터를 먼저 필터링하는 구조가 훨씬 안정적이었다.

이유는 두 가지였다.

* 실행 계획이 단순해짐
* 인덱스 활용률 증가

패턴은 다음과 같았다.

```sql
SELECT *
FROM (
    SELECT ID,
           LEAD(ID) OVER (ORDER BY ID DESC) AS NEXT_ID
    FROM BOARD
    WHERE USE_YN = 'Y'
)
```

윈도우 함수는 계산 단계이기 때문에
데이터를 먼저 줄이는 것이 핵심이다.
