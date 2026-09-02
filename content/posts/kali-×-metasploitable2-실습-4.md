+++
title = "Kali × Metasploitable2 실습 (4)"
date = "2026-09-02T15:28:00.000+09:00"
draft = false
summary = "XSS를 익스플로잇해보자"
tags = [ "Proxmox", "Kali Linux", "Metasploitable2", "네트워크", "홈랩" ]
series = [ "Kali-Metasploit Lab" ]
+++

지난 편에서 설정 오류로 뚫는 케이스 다섯 개를 정리했습니다. 이번 편에는 Metasploitable2에 번들된 DVWA로 웹 애플리케이션 취약점을 다뤄 보겠습니다. SQL Injection은 이전에 [다른 글](/posts/sql-injection/)에서 low/medium/high를 다 다뤄봤던 터라 이번 편에서는 재실습을 생략하고 XSS부터 시작했습니다.

## Reflected XSS

### Low

입력창 하나 있는 페이지라 별 고민 없이 제일 흔한 alert문을 넣어봤습니다.

```javascript
<script>alert(1);</script>
```

Low 레벨이라 그런지 바로 alert(1)이 떴습니다.

### Medium

같은 페이로드를 그대로 넣었더니 이번엔 안 터지고 화면에 이렇게 떴습니다.

```plain
Hello alert(1);
```

`<script>` 글자가 사라진 걸 보고 소스를 확인해보니

```php
$name = str_replace('<script>', '', $_GET['name']);
```

정확히 `<script>` 문자열 하나만 지우는 필터였습니다. `str_replace`는 기본적으로 대소문자를 구분하니까 `<Script>alert(1);</Script>`로 바꿔서 넣었더니 그대로 통과했습니다.

### High

이번엔 `script`라는 단어 자체를 안 쓰면 되지 않을까 싶어서 아래 페이로드로 시도했습니다.

```javascript
<img src=x onerror=alert(1)>
```

```plain
Hello &lt;img src=x onerror=alert(1)&gt;
```

`<`, `>`가 `&lt;`, `&gt;`로 인코딩돼 있었습니다. low/medium이 "위험한 문자열을 찾아서 지우는" 블랙리스트 방식이었다면 high는 입력값의 특수문자를 통째로 이스케이프하는 방식이었습니다. `<`가 애초에 텍스트로만 취급되니 이 지점은 이론적으로 XSS 공격이 되지 않습니다.

## Stored XSS

### Low

메시지 칸에 Reflected와같은 페이로드를 넣었더니 alert(1)이 떴습니다. reflected와 다르게 stored 방식은 페이지에 스크립트를 삽입하는 방식이기 때문에 페이지를 새로고침할 때마다 alert가 다시 실행되었습니다.

### Medium

일단 Low 레벨처럼 Message부터 시도해봤습니다.

Message 칸에는 `<script>`든 `<Script>`든 `<img onerror>`든 뭘 넣어도 태그 자체가 통째로 사라졌습니다. 대소문자 모두 필터링 되는 걸 보아하니 이전과는 다른 더 강한 필터가 걸려 있는 것 같았습니다.

그렇다면 Name 칸을 공략해야 한다고 생각했는데 Name 칸에는 입력 길이 제한이 있는듯했습니다. 그래서 어떡하지 고민하면서 소스코드를 봤는데 html에 maxlength=10이라고 정의되어있어서 그걸 지우고 Name 칸에 `<Script>alert(1);</script>`를 입력하니 새로고침할 때마다 alert이 떴습니다.

### High

Name, Message 둘 다 `&lt;Script&gt;...&lt;/script&gt;`로 인코딩돼서 텍스트로만 보였습니다. Reflected의 High와 같은 이스케이프 방식인 거 같아 여기도 우회가 안 되는 거 같습니다.

## 정리

| 종류 | Low | Medium | High |
| --- | --- | --- | --- |
| Reflected | 뚫림 | 대소문자 우회로 뚫림 | 방어 확인 |
| Stored | 뚫림 (새로고침마다 재실행) | Name 칸 대소문자 우회로 뚫림, Message 칸은 방어됨 | 방어 확인 |

low/medium은 블랙리스트 방식이라 우회 여지가 있었고 high는 입력값을 통째로 이스케이프하는 방식이라 실제로 뚫리지 않는다는 걸 확인할 수 있었습니다. 다음 편은 Command Injection, 파일 업로드 취약점 그리고 CSRF를 다룰 예정입니다.
