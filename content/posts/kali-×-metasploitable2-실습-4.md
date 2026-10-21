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

`<`, `>`가 `&lt;`, `&gt;`로 인코딩돼 있었습니다. 궁금해서 (3편에서 쓴) NFS로 대상 소스를 직접 봤더니 Reflected는 레벨마다 달랐습니다.

- Low: `'Hello ' . $_GET['name']` — 필터가 아예 없음

- Medium: `str_replace('<script>', '', $_GET['name'])` — `<script>` 문자열만 제거(대소문자 구분)

- High: `htmlspecialchars($_GET['name'])` — <, >, &, "를 HTML 엔티티로 인코딩

즉 위험한 문자열을 지우는 블랙리스트는 Medium 하나고, Low는 필터 자체가 없었습니다. High는 특수문자를 엔티티로 바꿔 `<`가 태그로 해석될 여지를 없앱니다. 또한 출력이 `<pre>` 안의 일반 텍스트 자리라, 이 입력 지점에서는 스크립트 주입이 성립하지 않습니다.

## Stored XSS

### Low

메시지 칸에 Reflected와 같은 페이로드를 넣었더니 alert(1)이 떴습니다. Reflected와 달리 Stored는 입력이 서버(DB)에 저장되고, 이후 그 페이지를 열 때마다 응답에 포함돼 실행됩니다. 그래서 새로고침할 때마다 alert가 다시 떴습니다.

### Medium

일단 Low 레벨처럼 Message부터 시도해봤습니다.

Message 칸에는 `<script>`든 `<Script>`든 `<img onerror>`든 뭘 넣어도 태그 자체가 통째로 사라졌습니다. 대소문자 모두 필터링 되는 걸 보아하니 이전과는 다른 더 강한 필터가 걸려 있는 것 같았습니다.

그렇다면 Name 칸을 공략해야 한다고 생각했는데 Name 칸에는 입력 길이 제한이 있는듯했습니다. 그래서 어떡하지 고민하면서 소스코드를 봤는데 html에 maxlength=10이라고 정의되어있어서 그걸 지우고 Name 칸에 `<Script>alert(1);</script>`를 입력하니 새로고침할 때마다 alert이 떴습니다.

소스로 보니 Message는 `strip_tags`로 태그를 없애고 `htmlspecialchars`까지 걸지만, Name은 `str_replace('<script>', '', ...)` 하나뿐이라 `<Script>`로 우회됐습니다. 그리고 Name의 `maxlength=10`은 HTML 속성이라 브라우저 입력만 막을 뿐 서버는 길이를 검사하지 않습니다. 그래서 그 속성을 지우니 긴 페이로드가 그대로 들어갔습니다

### High

Name, Message 둘 다 `&lt;Script&gt;...&lt;/script&gt;`로 인코딩돼서 텍스트로만 보였습니다. 소스코드를 확인해보니 Stored High는 Name,Message 둘 다 htmlspecialchars로 인코딩해서 Reflected High와 같은 방식으로 막혔습니다.

## 정리

| 종류 | Low | Medium | High |
| --- | --- | --- | --- |
| Reflected | 뚫림 | 대소문자 우회로 뚫림 | 방어 확인 |
| Stored | 뚫림 (새로고침마다 재실행) | Name 칸 대소문자 우회로 뚫림, Message 칸은 방어됨 | 방어 확인 |

Medium은 특정 문자열만 지우는 블랙리스트라 대소문자로 우회됐고, Low는 필터가 아예 없었습니다. High는 `htmlspecialchars`로 특수문자를 엔티티로 인코딩해서 이 출력 위치에서는 뚫리지 않았습니다.
