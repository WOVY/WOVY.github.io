+++
title = "JWT 개념과 구조 그리고 보안 취약점"
date = "2026-08-20T08:54:00.000+09:00"
draft = true
summary = ""
tags = [ "JWT" ]
series = [ "키워드 노트" ]
+++

백엔드 프로젝트에서 JWT를 관행적으로 써 본 경험은 있지만 정확한 개념과 구조는 잘 몰랐다. 이 글에서는 JWT의 개념과 구조를 정리하고 관련 취약점과 보안 위협도 함께 정리한다.

## JWT란?

- JWT는 JSON Web Token의 약자다.
- RFC 7519로 표준화된 토큰 형식이다.
- 두 개체 사이에서 정보를 JSON 객체로 안전하게 전달하기 위해 만들어졌다.
- 서명에는 HMAC(비밀키 공유 방식) 또는 RSA·ECDSA(공개키·개인키 방식)를 사용한다.

## JWT는 언제 사용하는가

- **인가(Authorization)**: 로그인 후 발급받은 JWT를 요청에 담아 리소스 접근 권한을 증명한다. SSO에서 널리 쓰인다.
- **정보 교환**: 서명된 JWT는 발신자 신원과 내용 위변조 여부를 함께 검증한다. 그래서 당사자 간 신뢰 기반 정보 전달 수단으로도 쓰인다.

## JWT 구성 요소

JWT는 header, payload, signature 세 부분으로 이루어진다. 각 부분은 Base64Url로 인코딩하고, 점(`.`)으로 구분한다.

```plain
(header).(payload).(signature)
```

### 1. header

header는 토큰 타입(`typ`)과 서명 알고리즘(`alg`)을 담는다.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### 2. payload

payload는 클레임(claim)을 담는다. 클레임은 이름-값 쌍으로 정보를 표현한 것이다. 클레임은 크게 세 종류로 나뉜다.

- **등록된 클레임(registered claim)**: `iss`(발급자), `exp`(만료 시간), `sub`(주체), `aud`(대상자) 등 표준 예약 필드
- **공개 클레임(public claim)**: 자유롭게 정의하되 충돌을 막기 위해 IANA 레지스트리에 등록하거나 URI 형태로 정의한다.
- **비공개 클레임(private claim)**: 당사자 간 합의로 만드는 커스텀 필드

payload는 서명되어 있지만 암호화되어 있지 않다. 그래서 누구나 내용을 읽을 수 있다. 비밀 정보는 담지 않는다.

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

### 3. signature

signature는 인코딩된 header, 인코딩된 payload, 비밀키를 header에 명시된 알고리즘으로 결합해 생성한다.

```plain
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

signature는 메시지 변조 여부를 검증한다. 개인키로 서명한 경우에는 signature는 발신자 신원도 함께 검증한다.

## 동작 방식

클라이언트는 보호된 리소스에 접근할 때마다 Authorization 헤더에 Bearer 방식으로 JWT를 담아 전송한다.

```plain
Authorization: Bearer <token>
```

1. 클라이언트가 인증 서버에 인증을 요청한다.
2. 인증 서버는 권한이 확인되면 클라이언트에게 액세스 토큰을 발급한다.
3. 클라이언트는 액세스 토큰으로 보호된 리소스에 접근한다.

![](/images/client-credentials-grant.webp "출처 : jwt.io")

> 참고: https://www.jwt.io/introduction

## 주요 보안 취약점

**1. alg: none 공격**

공격자는 header의 `alg` 값을 `none`으로 바꾸고 signature를 제거한다. 검증 로직이 부실하면 서버는 이 조작된 토큰을 그대로 수락한다.

관련 CVE: CVE-2021-22160(Apache Pulsar), CVE-2022-23540(jsonwebtoken), CVE-2025-61152(python-jose).

최신 표준 라이브러리는 이 공격을 기본으로 차단한다. 다만 커스텀 구현, 레거시 버전, 설정 실수에서는 지금도 재현 사례가 보고된다.

**2. 알고리즘 컨퓨전(RS256 → HS256)**

서버가 검증 알고리즘을 토큰 header의 `alg` 값에서 그대로 가져오면 취약해진다. 공격자는 RS256의 공개키를 HMAC 비밀키로 재사용해 위조 토큰을 만들 수 있다.

관련 CVE: CVE-2015-9235(jsonwebtoken), CVE-2023-48238(json-web-token), CVE-2024-54150(cjwt).

**3. 약한 시크릿 브루트포스**

HS256처럼 비밀키를 공유하는 방식에서 키가 단순하면, 공격자는 브루트포스로 키를 탈취할 수 있다.

**4. jwk / jku 헤더 인젝션**

- `jwk` 공격: 공격자는 자신이 만든 키 쌍의 공개키를 header에 직접 삽입한다. 서버가 이 키의 신뢰 여부를 확인하지 않으면 검증에 그대로 사용한다.
- `jku` 공격: 서버가 키 목록을 가져올 URL을 신뢰 도메인으로 제한하지 않으면, 공격자는 자신이 호스팅한 JWK Set을 참조하게 만들 수 있다.

**5. kid 파라미터 조작**

서버가 `kid` 값을 파일 경로나 DB 조회에 그대로 사용하면, 공격자는 경로 순회(path traversal)나 SQL 인젝션을 시도할 수 있다.

**6. 클레임 위변조(직렬화 불일치)**

Compact 직렬화와 JSON 직렬화의 처리 방식이 어긋나면, 서명 검증을 통과한 값과 실제 사용되는 클레임이 달라질 수 있다. 예: python-jwt의 CVE-2022-39227(CVSS 9.1).

**7. 키 관리 실패**

- 하드코딩된 서명 키: CVE-2025-7079(bluebell-plus), CVE-2025-6950(Moxa 장비)
- `iss` 클레임 형식 위반: CVE-2025-30144(fast-jwt) — 문자열이어야 할 `iss`에 배열을 허용
- 키 회전과 폐기가 미흡하면, 유출된 키로 위조 토큰을 계속 만들 수 있다. signature 검증은 정상 통과하므로 탐지가 늦어진다.

**8. 토큰 탈취 후 재사용**

만료 시간이 길거나 폐기 메커니즘이 없으면, 계정을 비활성화하거나 비밀번호를 재설정해도 기존 토큰은 만료 전까지 유효하다.

**9. CVE-2022-21449 (Psychic Signatures)**

Java 15\~18의 ECDSA 서명 검증에는 결함이 있었다. 이 결함 때문에 특정 조건에서는 signature 값 조작만으로 검증을 우회할 수 있었다.

> 참고: https://www.jwt.io/introduction
