+++
title = "SQL Injection"
date = "2026-08-30T12:49:00.000+09:00"
draft = false
summary = "SQL Injection에 대해 알아보자"
tags = [ "웹보안", "SQL", "SQL Injection" ]
series = [ "키워드 노트" ]
+++

이번에는 SQL Injection에 대해 알아보려 합니다. 제가 느끼기에는 정보 보안, 특히 웹 보안을 공부할 때 가장 처음 접하는 용어같습니다. 그만큼 기초적이고 또 중요한 개념입니다.

## 1. 개념 정의

### SQL Injection이란

SQL Injection(SQLi)은 애플리케이션이 DB로 보내는 쿼리를 공격자가 조작하는 취약점입니다. SQLi를 통해 공격자는 정상적으로는 접근할 수 없는 다른 사용자의 개인 정보, 비밀번호 등의 데이터를 조회할 수 있습니다. 더 나아가 데이터를 수정하거나 삭제해 애플리케이션의 컨텐츠를 영구적으로 수정할 수도 있습니다.

### 원인

SQLi가 발생하는 근본 원인은 하나로 요약됩니다. 코드가 사용자 입력과 분리되지 않은 채 하나의 문자열로 합쳐진다는 점입니다.

```java
String query = "SELECT * FROM users WHERE username = '" + input + "'";
```

위의 취약한 쿼리문에 공격자가 악의적인 `input`을 삽입한 후 DB로 보냈을 때 DB는 어디까지가 값이고 어디까지가 명령인지 구분할 방법이 없습니다. 그래서 DB는 `input`안에 따옴표, 주석 기호 등의 SQL 문법 요소가 섞여있어도 이를 그대로 쿼리 문법의 일부로 해석합니다.

이런 취약점은 특정 언어나 프레임 워크의 문제가 아니라 동적 쿼리를 문자열 결합으로 생성하는 모든 코드에서 동일하게 발생합니다. 그래서 SQLi는 오래전부터 알려졌음에도 불구하고 여전히 널리 발생하는 취약점으로 남아있습니다.

## 2. 공격 시나리오

### 2-1. 인증 우회

취약한 로그인 로직:

```php
$query = "SELECT * FROM users WHERE username = '$username' AND password = '$password'";
```

공격 페이로드:

```plain
username: administrator' --
password: (아무 값)
```

실행되는 쿼리:

```sql
SELECT * FROM users WHERE username = 'administrator' --' AND password = ''
```

`--`가 이후 구문을 주석 처리하면서 비밀번호 검증 조건 자체가 사라집니다. 결과적으로 비밀번호 없이 `administrator` 계정으로 로그인됩니다.

### 2-2. 숨겨진 데이터 조회

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

페이로드: `Gifts' OR 1=1--`

```sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1
```

`1=1`은 항상 참이므로 `released`조건과 무관하게 전체 상품이 노출됩니다.

### 2-3. UNION 공격으로 다른 테이블 데이터 추출

sql
SELECT name, description FROM products WHERE category = 'Gifts'

페이로드: `' UNION SELECT username, password FROM users--`

UNION 쿼리가 성립하려면 두 쿼리의 컬럼 수가 같고, 데이터 타입이 호환되어야 합니다. 이 조건을 맞추기 위해 실전에서는 `ORDER BY` 인덱스를 늘려가거나 `UNION SELECT NULL,NULL,...` 형태로 컬럼 수를 먼저 탐색합니다.

## 3. 탐지/실습 (DVWA)

로컬 환경에 구축한 뒤 보안 레벨을 바꿔가며 같은 페이로드가 어떻게 다르게 처리되는지 비교했습니다.

### 3-1. Low 레벨

1. Baseline 확인
`User ID`에 `1`을 입력하면 `ID`가 `1`인 사용자의 정보만 정상 출력됩니다.
2. Payload 입력
`User ID` `1' OR '1'='1` 입력
3. 결과
전체 사용자 목록이 반환됩니다.![](/images/sqli-low.png)
User ID 하나만 조회되어야 정상인데, 등록된 사용자 5명 전원의 정보가 노출되었습니다. 인증이나 권한 확인 없이 조건절이 우회됨을 확인했습니다.
4. 원인

```php
$query = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
```

`$id`가 검증없이 따옴표 안에 그대로 들어갑니다. 공격자가 `1' OR '1'='1`을 삽입하면 다음과 같이 조합됩니다.

```sql
SELECT first_name, last_name FROM users WHERE user_id = '1' OR '1'='1';
```

`'1'='1'`은 항상 참이므로 `WHERE`조건이 항상 참이 되어 전체 사용자 목록이 반환된 것입니다.

### 3-2. Medium 레벨

보안 레벨을 Medium으로 변경하면 입력 방식이 텍스트창에서 드롭다운으로 바뀝니다. 그래서 Burp Suite로 요청을 가로채 id 파라미터를 직접 조작해야 합니다.

일단 `User ID`를 1로 선택한 후 요청을 보내고 Burp Suite로 요청을 가로챈 뒤 `id=1&Submit=Submit`을 `id=1 OR 1=1&Submit=Submit`로 변경 후 Forward해 전체 사용자 목록을 받았습니다.

![](/images/sqli-medium.png)

### 3-3. High 레벨

`View Source`로 High 레벨 코드를 보면 아래와 같이 되어 있습니다.

```php
$id = $_SESSION['id'];

$getid = "SELECT first_name, last_name FROM users WHERE user_id = '$id' LIMIT 1;";
```

보면 LIMIT 1으로 출력 개수를 제한해뒀을 뿐 입력 검증을 강화한 형태가 아닙니다. 

```sql
SELECT first_name, last_name FROM users WHERE user_id = '' UNION SELECT user, password FROM users#' LIMIT 1;
```

위와 같이 쿼리문이 되면`#` 뒤의 `' LIMIT 1;`는 전부 주석 처리되어 데이터베이스가 아예 읽지 않습니다. `LIMIT 1`이 문자열로만 남을 뿐 적용되지 않으므로, UNION 쿼리는 제한 없이 `users` 테이블 전체 행을 반환합니다.

전체 사용자 계정과 비밀번호 해시가 여러 행에 걸쳐 그대로 출력됩니다.

## 4. 방어 기법
 
### 4-1. 근본적 해결책
 
**파라미터화된 쿼리(Prepared Statement)** 사용이 유일하게 확실한 근본 대책입니다. 쿼리 구조와 데이터를 데이터베이스 드라이버에서 분리하기 때문에 입력값에 어떤 SQL 문법이 들어와도 값으로만 처리됩니다.
 
```java
PreparedStatement stmt = connection.prepareStatement(
    "SELECT * FROM users WHERE username = ? AND password = ?");
stmt.setString(1, username);
stmt.setString(2, password);
```
 
같은 원리로 ORM(Object-Relational Mapper)을 사용해 쿼리를 자동 생성하는 방식도 근본 대책에 포함됩니다.
 
### 4-2. 보조적 방어책
 
근본 대책을 대책은 아니지만 함께 적용하면 좋은 것들은 다음과 같습니다.
 
- **입력 검증**: 예상되는 형식(숫자, 이메일 등)을 벗어난 입력을 조기에 거부
- **최소 권한 원칙**: 애플리케이션이 사용하는 DB 계정에 필요한 권한만 부여 (DROP, 시스템 테이블 접근 등 차단)
- **에러 메시지 노출 최소화**: 상세 DB 에러를 사용자에게 그대로 반환하지 않음 (에러 기반 SQLi 방지)
- **WAF(웹 방화벽)**: 알려진 패턴 기반 탐지로 1차 방어선 역할. 우회 가능성이 있으므로 근본 대책의 보조 수단으로만 사용
- **정기적인 코드 리뷰 및 스캐닝**: Burp Scanner, sqlmap 등을 이용한 주기적 점검
