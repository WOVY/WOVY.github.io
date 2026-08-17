+++
title = "Kali × Metasploitable2 실습 (0)"
date = "2026-08-17T15:52:00.000+09:00"
draft = true
summary = "Proxmox에서 Kali와 Metasploitable2가 서로 다른 격리 브리지에 물려 있어 통신이 안 되던 문제를 진단하고, 같은 세그먼트로 옮겨 핑 테스트까지 확인한 과정을 정리합니다."
tags = [ "Proxmox", "Kali Linux", "Metasploitable2", "네트워크", "홈랩" ]
series = [ "kali-metasploit lab" ]
+++

## 왜 이 랩을 시작했나

작년에 보안 공부 해보겠다고 미니 PC 사서 proxmox에 Kali, Metasploit 설치하고 실습해보려다가 바쁘다는 핑계로 방치하고 있었습니다. 이제는 진짜 한 번 해봐야지 하는 결심에 블로그에 기록하면서 실습해보려고 합니다.

계획은 이렇습니다. 클로드와 함께 계획했는데

1. 정찰·스캐닝
2. 백도어·원격 코드 실행
3. 설정 오류로 침투하기
4. 웹 애플리케이션 취약점 (SQLi, XSS, CSRF...)
5. 인증 공격 (브루트포스, 해시 크래킹)
6. 권한 상승
7. 탐지자 시점에서 다시 보기
8. 리포트 써보기

이번 글은 그 이전 단계 그러니까 **0단계**입니다. 로드맵을 짜놓고 막상 시작하려니까 두 VM이 서로 통신이 안 되더라고요. 분명 작년에 세팅했을 때는 통신되는 거 확인했었는데,,,

## 환경

- Kali : 공격자 역할
- Metasploitable2 : 일부러 취약하게 만들어둔 피해자 역할
- 둘 다 인터넷도 안 되고 LAN도 안 뚫리는 순수 격리 브리지에 물려 있어야 정상입니다. 실습 트래픽이 집 안 다른 기기로 새 나가면 안되니깐요.

이론상으로는 이 브리지 안에서 서로 자유롭게 통신하면서, 바깥세상과는 완전히 차단돼 있어야 합니다.

## 문제: 핑이 안 나갑니다

두 VM을 켜고 콘솔에 들어가서 첫 번째로 한 일은 그냥 서로 핑을 날려보는 거였습니다. 결과는 아무 응답 없더라고요.

당연히 방화벽이나 설정 문제겠거니 하고 호스트에서 브리지 상태부터 확인했습니다.

```plain
bash

$ qm config 100 | grep net0
net0: virtio=BC:24:11:64:9E:19,bridge=vmbr1,firewall=1

$ qm config 101 | grep net0
net0: virtio=BC:24:11:A4:7A:48,bridge=vmbr2,firewall=1
```

원인은  Kali는 `vmbr1`, Metasploitable2는 `vmbr2`으로 애초에 서로 다른 네트워크에 있있는 거 같았습니다.

호스트 쪽 브리지 설정도 다시 봤습니다.

```plain
auto vmbr1

iface vmbr1 inet manual

      bridge-ports none

auto vmbr2

iface vmbr2 inet manual

      bridge-ports none
```

둘 다 \`bridge-ports none\`, 즉 완전히 독립된 브리지입니다. 둘을 이어주는 장치도 없고, IP도 없어서 호스트가 라우팅을 해줄 방법도 없습니다. 방화벽 문제가 아니라 애초에 \*\*패킷이 지나갈 길 자체가 없는\*\* 상황이었던 겁니다. \`bridge fdb\`로 봐도 그림이 명확했습니다.

```plain
bc:24:11:64:9e:19 dev fwpr100p0 master vmbr1   ← Kali

bc:24:11:a4:7a:48 dev fwpr101p0 master vmbr2   ← Metasploitable2
```

완전히 다른 섬에 떨어진 두 VM이었던 셈입니다.

## 고치기

Metasploitable2를 Kali가 있는 `vmbr1`로 옮기면 간단하게 해결될 거 같았습니다.

혹시 몰라서 옮기기 전에 스냅샷부터 하나 남겨뒀습니다. 실습이라는 게 원래 뭔가 잘못됐을 때 되돌릴 구멍을 만들어두고 시작하는 게 국룰이니까요.

```bash
$ qm set 101 -net0 virtio=BC:24:11:A4:7A:48,bridge=vmbr1,firewall=1
update VM 101: -net0 virtio=BC:24:11:A4:7A:48,bridge=vmbr1,firewall=1
```

VM을 끄지 않고도 바로 적용됐습니다. 브리지를 옮기는 건 VM 입장에서는 네트워크 케이블을 다른 스위치에 꽂아주는 것과 비슷해서 재부팅 없어도 바로 반영된다고 하네요.

## 체크

설정만 바꿔놓고 "됐겠지" 하고 넘어가면 나중에 꼭 사달이 나더라고요. 그래서 각 VM 콘솔에 직접 들어가서 확인했습니다.

Kali에서 IP부터 확인하고,

```plain
$ ip a

2: eth0: ... 
    inet 192.168.100.10/24 brd 192.168.100.255 scope global eth0
```

Metasploitable2로 핑을 날려봤습니다.

```plain
$ ping -c 4 192.168.100.100

PING 192.168.100.100 (192.168.100.100) 56(84) bytes of data.
64 bytes from 192.168.100.100: icmp_seq=1 ttl=64 time=0.512 ms
64 bytes from 192.168.100.100: icmp_seq=2 ttl=64 time=0.470 ms
64 bytes from 192.168.100.100: icmp_seq=3 ttl=64 time=0.687 ms
64 bytes from 192.168.100.100: icmp_seq=4 ttl=64 time=0.601 ms

--- 192.168.100.100 ping statistics ---

4 packets transmitted, 4 received, 0% packet loss, time 3056ms
```

생각보다 쉽게 해결했고 겸사겸사 `netdiscover`로도 스캔해봤는데, Metasploitable2가 정확히 잡혔습니다.

확인이 끝난 시점에서 스냅샷을 하나 남겨뒀습니다. 나중에 실습하다가 VM을 망가뜨려도 여기로 바로 돌아올 수 있습니다.

## 다음 편 예고

네트워크는 이제 확실히 뚫려 있습니다. 다음 글부터는  **정찰과 스캐닝**으로 넘어갑니다. `nmap`으로 Metasploitable2가 어떤 포트를 열어두고 있는지 훑어보는 것부터 시작할 거 같습니다.
