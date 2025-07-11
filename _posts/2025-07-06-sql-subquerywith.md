---
title: "[Study/MySQL] 서브쿼리와 WITH 절의 쓰임 비교"
author: eun
date: 2025-07-06 10:00:00 +0900
categories: [Study, MySQL]
tags: [Intern, MySQL]
render_with_liquid: true
image:
  path: https://miro.medium.com/v2/resize:fit:1100/format:webp/1*akaZCEWu-YERVMty5Jd1Fw.jpeg
  # lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt: Subquery vs WITH
---

실습을 하다가 문득, 둘의 차이점에 대해 물음표가 생기게 되었습니다. 오늘은 서브쿼리의 개념과 with의 개념, 그리고 둘의 차이와 간단한 예제에 대해 공부해보겠습니다.

<br>

> **꼭 알아야 하는 개념**
> 
> 
> **※ 해당 개념에서의 “동적”:** 쿼리를 실행할 때마다 필요한 값을 그때그때 계산해서 쿼리 결과에 반영하는 개념.
> 
> **※ CTE: Common Table Expression의 약자로,** 단일 쿼리 내부에서 임시로 결과를 저장해 놓고, 해당 쿼리 내에서 반복적으로 사용 가능한 임시 결과 집합(테이블)을 의미한다. 대표적으로 with 문법이 존재함.
> 

<br>

## 0. 서브쿼리 vs WITH 차이 요약

---

| 구분 | 서브쿼리 | WITH 절(CTE) |
| --- | --- | --- |
| 특징 | 쿼리 안에 사용되는 **작은** **쿼리** | 쿼리 맨 위에 미리 이름을 붙인 **임시 테이블** |
| 쓰임 | 간단한 조건이나 계산에 적합 | 복잡한 로직, 여러번 재사용할 때 편리 |
| 반복 계산 여부 | O (같은 서브쿼리면 반복 계산) | X (한 번 계산 후 재사용) |
|  |  |  |

※ WITH 절은 서브쿼리보다 나중에 도입된 개념으로, 서브쿼리의 단점(복잡성, 재사용성)을 개선하기 위해 발전된 문법이다.

<br>

## 1. 서브쿼리(Subquery)의 개념

---

서브쿼리란 **쿼리 안에 포함된 또 다른 쿼리**를 말한다.

메인 쿼리의 조건이나 결과를 **동적으로 만들기 위해** 사용한다. 괄호로 감싸서 작성하며, 위치에 따라 역할과 실행 순서가 달라진다.

```sql
SELECT
    emp_no,
    salary
FROM
    employees
WHERE
    ***salary > (
        SELECT AVG(salary) FROM employees
    );***
```

`SELECT AVG(salary) FROM employees`는 평균 급여를 미리 계산해 테이블에 고정된 값으로 넣는 게 아니라, **쿼리가 실행될 때마다 최신 직원 급여 데이터를 기준으로 평균을 다시 계산해서 비교**한다는 의미다.

즉, 서브쿼리를 통해 결과를 **실시간으로 계산해서 쿼리에 반영한다**는 뜻을 의미한다.

<br>

## 2. 서브쿼리의 종류와 예제

---

서브쿼리는 대표적으로 SELECT, FROM, WHERE 절에 사용된다.

<br>

### 2.1) SELECT절에서의 서브쿼리

SELECT절 안에 들어가는 서브쿼리는 **각 행마다 계산된 값을 컬럼으로 추가**한다.

```sql
-- 같은 테이블 사용 예제: 직원 급여와 전체 평균 급여 조회
SELECT
    emp_no,
    salary,
    (SELECT AVG(salary) FROM employees) AS avg_salary
FROM
    employees;
```

**동작 순서**

1. `employees`의 모든 행을 읽음
2. 각 행마다 `SELECT AVG(salary) FROM employees` 서브쿼리를 실행해 평균 급여를 구함
3. 결과에 평균 급여 컬럼 추가

```sql
-- 다른 테이블 사용 예제: 주문 정보와 고객 국가 표시
SELECT
    o.order_id,
    o.amount,
    (SELECT country FROM customers c WHERE c.customer_id = o.customer_id) AS customer_country
FROM
    orders o;

```

**동작 순서**

1. `orders`의 모든 주문 조회
2. 각 주문마다 고객 테이블에서 국가를 찾아서 컬럼으로 추가

<br>

### 2.2) FROM절에서의 서브쿼리 (=Inline View)

FROM절에 들어가는 서브쿼리는 **임시 테이블처럼 사용**된다. 이를 ‘인라인 뷰(Inline View)’라고 부른다.

```sql
-- 같은 테이블 사용 예제: 부서별 평균 급여 구하고 5000 이상 부서 조회
SELECT
    dept_no,
    avg_salary
FROM
    **(SELECT dept_no, AVG(salary) AS avg_salary FROM employees GROUP BY dept_no) AS dept_avg**
WHERE
    avg_salary >= 5000;
```

**동작 순서**

1. 서브쿼리에서 부서별 평균 급여 계산
2. 결과 임시 테이블 `dept_avg` 생성
3. 메인 쿼리에서 5000 이상인 부서만 필터링

```sql
-- 다른 테이블 사용 예제: 상품별 총 판매수량과 상품명 조회
SELECT
    p.product_id,
    p.product_name,
    s.total_quantity
FROM
    products p
JOIN
    (SELECT product_id, SUM(quantity) AS total_quantity FROM order_items GROUP BY product_id) AS s
ON
    p.product_id = s.product_id;
```

**동작 순서**

1. `order_items`에서 상품별 판매수량 집계
2. 임시 테이블 `s` 생성
3. `products`와 조인해 상품명과 수량 조회

<br>

### 2.3) WHERE절에서의 서브쿼리

`WHERE`절 서브쿼리는 **조건을 동적으로 생성**하는 데 쓰인다.

```sql
-- 같은 테이블 사용 예제: 급여가 전체 평균 이상인 직원 조회
SELECT
    emp_no,
    salary
FROM
    employees
WHERE
    salary > (SELECT AVG(salary) FROM employees);
```

**동작 순서**

1. 서브쿼리에서 전체 평균 급여 계산
2. 메인 쿼리에서 평균 이상인 직원 필터링

```sql
-- 다른 테이블 사용 예제: 주문 금액이 상품 최고가 이상인 주문 조회
SELECT
    order_id,
    amount
FROM
    orders
WHERE
    amount >= (SELECT MAX(price) FROM products);
```

**동작 순서**

1. 서브쿼리에서 `products` 최고가 계산
2. `orders`에서 금액이 최고가 이상인 주문만 조회

## 3. WITH 절의 개념

---

CTE=Common Table Expression)를 선언하기 위한 문법이다. 

WITH 절은 쿼리의 맨 앞에서 **임시 테이블을 이름과 함께 미리 정의**하는 기능이다. 복잡한 쿼리를 여러 단계로 나누어 작성할 때 가독성과 재사용성을 높인다.

이 임시 테이블은 쿼리 실행 동안에만 유효하며, 여러 쿼리에 걸쳐 지속되지는 않는다. 다시 말해, WITH 절은 뒤에 오는 단일 SQL 문 전체에서만 유효하다.

<br>

## 4. WITH 절 예제

---

```sql
**WITH dept_avg AS (
    SELECT dept_no, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_no
)**
SELECT
    e.emp_no,
    e.salary,
    e.dept_no
FROM
    employees e
JOIN
    dept_avg d ON e.dept_no = d.dept_no
WHERE
    e.salary > d.avg_salary;
```

**동작 순서**

1. WITH절에서 부서별 평균 급여를 미리 계산하여 `dept_avg` 임시 테이블 생성
2. `employees` 테이블과 `dept_avg`를 조인
3. 급여가 부서 평균 이상인 직원만 조회

<br>

## 5. 서브쿼리와 WITH 절의 차이

---

| 구분 | 서브쿼리 | WITH 절 |
| --- | --- | --- |
| 선언 위치 | 쿼리 내부 원하는 위치에 작성 | 쿼리 시작 부분에 작성 |
| 재사용성 | 재사용 불가, 반복 작성 필요 | 한번 선언해 여러 곳에서 재사용 가능 |
| 가독성 | 복잡해지면 읽기 어려움 | 단계별 분리로 가독성 우수 |
| 성능 | 단순 쿼리에 적합 | 복잡 쿼리 최적화에 유리 (MySQL 8.0 이상) |
| 쓰임 | 간단한 조건이나 계산에 적합 | 복잡 로직 분리 및 재사용에 적합 |

※ 서브쿼리는 작고 간단한 조건 처리에 좋고, WITH절은 복잡한 로직을 단계별로 나누어 관리할 때 매우 유용하다.

<br><br>

예전에 SQLD 준비할 때 개념적으로만 암기했던 개념인데, 확실히 실무에서 사용해보니 이해하기에는 훨씬 쉬웠습니다. 아무래도 쿼리문이 길어질수록 서브쿼리보다는 with 절로 사용하는게 깔끔하고 코드 리뷰할 때도 편하더라고요. 각 쓰임 용도에 맞게 쓰면 좋을 듯합니다.

궁금한 점이나 추가 예제가 필요하면 댓글로 알려주세요 ╰(*°▽°*)╯


<br><br><br><br>

### 🔗 참고 링크
- [https://g-dhasade16.medium.com/sql-subquery-vs-ctes-b312a64614f](https://g-dhasade16.medium.com/sql-subquery-vs-ctes-b312a64614f)