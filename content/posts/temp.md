+++
title = "Kali × Metasploitable2 실습 (1)"
date = "2026-08-29T12:26:00.000+09:00"
draft = false
summary = "Metasploitable2를 nmap으로 스캔 후 취약점 점검"
tags = [ "Proxmox", "Kali Linux", "Metasploitable2", "네트워크", "홈랩" ]
series = [ "kali-metasploit lab" ]
+++

이번엔 스캐닝부터 시작합니다. 침투 테스트의 첫 단계는 항상 정찰입니다. 대상이 뭘 열어두고 있는지 알아야 어디를 공략해야 할지 알 수 있으니깐요.

## 환경

- 공격자: Kali (`192.168.100.10`)
- 대상: Metasploitable2 (`192.168.100.100`)

## 전체 포트부터 훑기

일단 전체 포트부터 빠르게 훑었습니다.

```plain
$ nmap -p- -T4 --min-rate 1000 192.168.100.100
```

23개 포트가 열려 있다는 걸 확인했고 여기서 나온 포트만 골라서 서비스·버전까지 자세히 봤습니다.

```plain
$ nmap -sC -sV -p <포트 목록> -oN scan1.txt 192.168.100.100
```

| 포트 | 프로토콜 | 서비스 | 버전 |
| --- | --- | --- | --- |
| 21 | tcp | ftp | vsftpd 2.3.4 (익명 로그인 허용) |
| 22 | tcp | ssh | OpenSSH 4.7p1 Debian 8ubuntu1 |
| 23 | tcp | telnet | Linux telnetd |
| 25 | tcp | smtp | Postfix smtpd |
| 53 | tcp | domain | ISC BIND 9.4.2 |
| 80 | tcp | http | Apache httpd 2.2.8 (Ubuntu) DAV/2 |
| 111 | tcp | rpcbind | 2 (RPC #100000) |
| 139 | tcp | netbios-ssn | Samba smbd 3.X - 4.X |
| 445 | tcp | netbios-ssn | Samba smbd 3.0.20-Debian |
| 512 | tcp | exec | netkit-rsh rexecd |
| 513 | tcp | login | OpenBSD or Solaris rlogind |
| 514 | tcp | shell | Netkit rshd |
| 1099 | tcp | java-rmi | GNU Classpath grmiregistry |
| 1524 | tcp | bindshell | Metasploitable root shell |
| 2049 | tcp | nfs | 2-4 (RPC #100003) |
| 2121 | tcp | ftp | ProFTPD 1.3.1 |
| 3306 | tcp | mysql | MySQL 5.0.51a-3ubuntu5 |
| 3632 | tcp | distccd | v1 (GNU 4.2.4 Ubuntu 4.2.4-1ubuntu4) |
| 5432 | tcp | postgresql | PostgreSQL DB 8.3.0 - 8.3.7 |
| 5900 | tcp | vnc | VNC protocol 3.3 |
| 6000 | tcp | X11 | (access denied) |
| 6667 | tcp | irc | UnrealIRCd |
| 6697 | tcp | irc | UnrealIRCd |
| 8009 | tcp | ajp13 | Apache Jserv Protocol v1.3 |
| 8180 | tcp | http | Apache Tomcat/Coyote JSP engine 1.1 |

nmap 결과를 ai와 함께 취약점이 있는지 점검한 결과 즉시 익스플로잇 가능한 서비스는 아래와 같았습니다.

- 1524/tcp bindshell: 루트 쉘이 열려있음.
- 21/tcp vsftpd 2.3.4: 백도어가 심어진 버전(CVE-2011-2523).
- 3632/tcp distccd: 원격 코드 실행이 가능(CVE-2004-2687).
- 6667/tcp UnrealIRCd: 백도어 버전(CVE-2010-2075).

이 외에도 취약한 포트들이 많지만 일단 즉시 익스플로잇 가능한 서비스들만 추렸습니다.

## UDP 스캔은 생각보다 애매,,,

TCP만 보고 끝내면 안 되니깐 UDP도 돌려봤습니다.

```plain
$ nmap -sU --top-ports 50 192.168.100.100
```

`53`(domain), `111`(rpcbind), `137`(netbios-ns), `2049`(nfs)는 open이나 `69`(tftp)랑 `138`(netbios-dgm)은 `open|filtered`로 나왔습니다.

`open|filtered`가 뭐지 싶어서 검색해보니 UDP는 연결 개념이 없어서 응답이 안 오면 nmap은 원인을 알 수가 없습니다. 포트가 열려 있는데 패킷에 반응을 안 했을 수도 있고 포트가 닫혀 있거나 막혀있을 수도 있다고 하네요. 그래서 `open|filtered`로 표기합니다.

## 체크

- 열린 TCP 포트 25개의 서비스·버전까지 표로 정리 완료
- UDP는 대역 특성상 확답을 못 주는 포트가 있다는 것도 직접 겪음
- 스캔 결과와 실제 서비스가 맞는지 배너 그래빙으로 대조하는 건 아직 안 함 — `nc 192.168.100.100 21` 같은 걸로 직접 확인하는 게 다음 할 일

## 다음

다음 글은 **백도어 & 원격 실행**입니다. vsftpd 백도어, UnrealIRCd 백도어, distccd RCE, 그리고 이미 열려 있던 1524번 루트 쉘까지 — Metasploit으로 하나씩 잡아볼 예정입니다.
