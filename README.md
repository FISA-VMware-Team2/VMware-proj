#  VMware 가상화 실습

| ![](https://avatars.githubusercontent.com/u/107402806?v=4) | ![](https://avatars.githubusercontent.com/u/113874212?v=4) | ![](https://avatars.githubusercontent.com/u/215618778?v=4) |![](https://avatars.githubusercontent.com/u/181299322?v=4)|![](https://avatars.githubusercontent.com/u/89902255?v=4)|![](https://avatars.githubusercontent.com/u/113014598?v=4)|
|:---:|:---:|:---:|:---:|:---:|:---:|
| **김소연**<br>[@thdus](https://github.com/thdus) | **사재헌**<br>[@Zaixian5](https://github.com/Zaixian5) | **서주영**<br>[@young2z1](https://github.com/young2z1) | **신성혁**<br>[@ssh221](https://github.com/ssh221) | **유예원**<br>[@Yewon0106](https://github.com/Yewon0106) | **주용완**<br>[@YongwanJoo](https://github.com/YongwanJoo) |

## 프로젝트 소개

우리 FISA 6기 클라우드 엔지니어링 과정에서 학습한 vSphere과 서버 구축을 실습하였다.

가상머신과 워크스테이션 환경에서 ESXi와 vCenter를 직접 설치하고 네트워크를 설정하며 가상화 인프라의 뼈대를 익히고, DNS, NTP, NAT 등의 서버를 직접 만들어 보며 프라이빗 클라우드 엔지니어링의 기초를 이해하는 것을 목표로 하였다.

<br/>

---
## 📑 목차

1. [전체 아키텍처](#️-전체-아키텍처)
2. [네트워크 설계](#-네트워크-설계)
3. [vcenter 구성도](#vcenter-구성도)
4. [Server ESXi 아키텍처](#server-esxi-아키텍처)
5. [ESXi 클러스터 구조](#esxi-클러스터-구조)
6. [상단 ESXi Host 구성](#상단-esxi-host-구성)
7. [각 vSwitch 상세 설명](#각-vswitch-상세-설명)
8. [구성원 별 Data Center 개요](#구성원-별-data-center-개요)
9. [DHCP (Dynamic Host Configuration Protocol)](#dhcp-dynamic-host-configuration-protocol)
10. [NTP (Network Time Protocol)](#ntp-network-time-protocol)
11. [DNS (Domain Name System)](#dns-domain-name-system)
12. [공유 스토리지(Shared Storage)](#공유-스토리지shared-storage)
13. [Cluster](#cluster)
14. [Resource Pool](#resource-pool)
15. [ESXi 사용자 계정 생성 및 권한 부여](#-esxi-사용자-계정-생성-및-권한-부여)
16. [Lockdown Mode (잠금 모드)](#lockdown-mode-잠금-모드)
17. [Alarm (알람 / 모니터링)](#--alarm-알람--모니터링)
18. [HA (High Availability)](#hahigh-availability)
19. [FT (Fault Tolerance)](#ft-fault-tolerance)
20. [트러블 슈팅](#트러블-슈팅)

---

## 🏛️ 전체 아키텍처
<img width="5586" height="2888" alt="Image" src="https://github.com/user-attachments/assets/c3a30625-9425-4e09-996b-e4ded026a540" />

<br/>

## 🌐 네트워크 설계
### 📌 Network Configuration

| 용도 | 대역 | 설명 |
| :--- | :--- | :--- |
| **Management & VM Network** (Black) | `172.16.10.X/24` | 호스트 관리 및 VM의 서비스 통신용 |
| **Fault Tolerance 망** (Red) | `172.16.20.X/24` | FT 기능을 위한 호스트 간 실시간 데이터 동기화용 |
| **vMotion 망** (Pink) | `172.16.30.X/24` | vMotion 시 메모리 데이터 전송 전용 |
| **Storage 망** (Green) | `172.16.40.X/24` | 공유 스토리지(iSCSI / NFS) 접근용 |

### 📌 설계 의도
- 트래픽 분리 → 성능 + 안정성 확보
- vMotion / FT 전용망 확보

## vcenter 구성도

<img width="729" height="630" alt="1" src="https://github.com/user-attachments/assets/b2f8ef54-4840-4d0f-8334-3f80599e8d85" />

---

## Server ESXi 아키텍처

### 서비스 서버

<img width="820" height="167" alt="2" src="https://github.com/user-attachments/assets/4004863d-1d31-46e5-be9b-f77f39b50918" />

| 서버 종류 | 할당 IP | 역할 |
| --- | --- | --- |
| DNS (vcenter.team2.com) | 172.16.10.2 / 24 | 도메인 기반 통신 담당 |
| NAT |   • 172.16.10.4 / 24<br>• 자동 IP | 내부망 서버들이 외부 인터넷에 접속할 수 있도록 주소를 변환 |
| NTP | 172.16.10.3 / 24 | 모든 호스트와 가상머신의 시간들을 동기화 |
| Shared Sorage (iSCSI) |   • 172.16.10.11 / 24<br>• 172.16.40.1 / 24 |   • 모든 호스트가 공유하는 저장소<br>• 윈도우 기반 iSCSI으로 구성됨 |
| Web Server (NFS) | 172.16.10.5 / 24 | 파일 공유 및 웹 서비스 제공  |

### 가상 스위치

- 가상 트래픽을 물리 랜카드로 전달하는 중계기 역할

<img width="1676" height="168" alt="3" src="https://github.com/user-attachments/assets/2ffbf08b-44c8-4add-b4a6-999b70c1102e" />


| 가상 스위치 | 구성 요소 (Port Group / VMkerenel) | 연결된 외부 서비스 | 상세 설명 |
| --- | --- | --- | --- |
| vSwitch0 |   • VM Network<br>• Management Network (172.16.10.101 / 24) |   • VM Network<br> •DNS<br>• NAT<br>• NTP<br>• Shared Storage (iSCSI)<br>• NFS<br>• 가상 ESXi의 VM Network |   • 물리 호스트(10.101)의 관리 기능을 수행하며, DNS/NTP/NAT 등의 서버들을 연결<br>• 가상 ESXi들의 트래픽이 외부망으로 나가는 관문 역할 |
| vSwitch1 | I-PG (Internet Port Group) |   • NAT<br>• DHCP | NAT 서버를 통해 내부 사설망 VM들이 DHCP와 통신할 수 있는 물리적 통로 제공 |
| vSwitch2 |   • Storage-PG<br>• iSCSI-VMK (172.16.40.2 / 24) |   • Shared Storage (iSCSI)<br>• 가상 ESXi의 iSCI-VMK<br>• DHCP | iSCSI 프로토콜 기반의 대용량 데이터 I/O 처리 |
| vSwitch3 | vMotion-PG | 가상 ESXi의 vMotion-VMK | vMotion 실행 시 발생하는 호스트 간 메모리 복제 트래픽 처리 |
| vSwitch4 | FT-PG | 가상 ESXi의 FT-VMK | FT 기능이 활성화된 VM의 데이터  상태를 실시간으로 미러링 |

### 물리 랜카드

- 가상 호스트들을 구동하는 실제 물리 서버로, 모든 가상 트래픽은 Server ESXi의 vmnic을 통해 외부로 나감

<img width="1627" height="481" alt="4" src="https://github.com/user-attachments/assets/7a64e529-9968-41c5-8bea-95204a095ad3" />


| 종류 | 용도  | 할당 대역 | 설명 |
| --- | --- | --- | --- |
| vmnic0 | 관리용 | 172.16.10.X / 24 | Management Network와 연결되어 외부 스위치로 트래픽 전달 |
| vmnic1 | 인터넷용 | 자동 할당 | DHCP 서버와 연결되어 가상머신들에 외부 인터넷 제공 |
| vmnic2 | 스토리지용 | 172.16.40.X / 24 | Shared Storage 장비와 직접 연결된 전용 포트 |

---

### ESXi (서버 위의 ESXi )

- 독립적으로 동작하는 가상 호스트들

<img width="1919" height="478" alt="5" src="https://github.com/user-attachments/assets/52642742-c5a5-4d73-aa7a-4f9bab8a779d" />


- 가상 호스트별 IP 설정
    - ESXi-01
        
        
        | 종류 | 할당 IP |
        | --- | --- |
        | Management | 172.16.10.82 / 24 |
        | iSCSI-VMK | 172.16.40.89 / 24 |
        | vMotion-VMK | 172.16.30.82 / 24 |
        | FT-VMK | 172.16.20.82 / 24 |
    - ESXi-02
        
        
        | 종류 | 할당 IP |
        | --- | --- |
        | Management | 172.16.10.81 / 24 |
        | iSCSI-VMK | 172.16.40.88 / 24 |
        | vMotion-VMK | 172.16.30.81 / 24 |
        | FT-VMK | 172.16.20.81 / 24 |
- vSwitch
    - 각 호스트 (ESXI1, ESXI2) 내부에 용도별로 4개의 가상 스위치를 배치하였음
        
        
        | 가상 스위치 | 구성 요소 (Port Group / VMkernel) | 할당 IP (ex : ESXI-01 기준) | 설명 |
        | --- | --- | --- | --- |
        | vSwitch0 | VM Network (PG) / Management (VMK) | 172.16.10.81 / 24 | 관리 및 VM 서비스 통합 스위치 |
        | vSwitch1 | iSCI (VMK ) | 172.16.40.88 / 24 | 스토리지 전용 스위치 |
        | vSwitch2 | vMotion (VMK ) | 172.16.30.81 / 24 | vMotion 전용 스위치 |
        | vSwitch3 | FT (VMK ) | 172.16.20.81 / 24 | Fault Tolerance 전용 스위치 |

<br/>
<br/>

## ESXi 클러스터 구조

### 전체 구조

<aside>

여러 ESXi Host를 하나의 클러스터로 묶고, 네트워크를 역할별로 분리해 안정성과 확장성을 확보한 구조

</aside>

---

### 구성

- ESXi-01 ~ ESXi-12 (총 12대 서버)
- 각각 독립된 물리 서버
- 하나의 **Cluster**로 묶임
<img width="757" height="466" alt="Image" src="https://github.com/user-attachments/assets/ebccfddf-0da9-483b-b8c8-22c874ed0797" />

> ESXi 한 대 내부 구조를 확대한 상세 설계도
> 

---

## 상단 ESXi Host 구성

## ESXi-01 / ESXi-02

- 각각 하나의 물리 서버
- IP:
    - ESXi-01 → 172.16.10.11
    - ESXi-02 → 172.16.10.12

이 IP는 **관리망(Management Network)**

---

### VM 구성

각 Host 내부 

- Windows VM
- Rocky Linux VM

```
1개의 ESXi = 여러 VM을 실행하는 가상화 서버
```

---

### vSwitch 구조

- vSwitch 구성

| vSwitch | 역할 |
| --- | --- |
| vSwitch0 | VM Network + Management |
| vSwitch1 | Storage |
| vSwitch2 | vMotion |
| vSwitch3 | Fault Tolerance |

## 각 vSwitch 상세 설명

## ⚫ vSwitch0 (가장 중요)

### 포함 네트워크

- VM Network (보라)
- Management (주황)

---

### 역할

두 가지 기능 동시에 수행

VM 트래픽

- 사용자 접속 (웹, 서비스)

관리 트래픽

- vCenter 접속
- ESXi 관리

---

## 🟢 vSwitch1 (Storage)

### 역할

- NFS / iSCSI 연결

### 모든 ESXi가 같은 스토리지 사용 가능

```
ESXi-01 ↔ Storage ↔ ESXi-02
```

VM 이동 가능 (vMotion, HA)

---

## 🟣 vSwitch2 (vMotion)

### 역할

- VM 메모리 이동 전용 네트워크

---

### 동작

```
ESXi-01 → ESXi-02
VM 이동 (서비스 끊김 없음)
```

Live Migration

---

## 🔴 vSwitch3 (Fault Tolerance)

### 역할

- VM 복제 트래픽

---

Primary VM + Secondary VM 동기화

```
Primary VM → Secondary VM 실시간 복제
```

장애 시 즉시 전환 (무중단)

---

## 구성원 별 Data Center 개요



> 본 실습에서는 vSphere 환경에서 **Data Center → Cluster → Resource Pool 구조**를 구성하여 가상 자원을 효율적으로 분배하고 관리하는 구조를 구현


<br/>

---
### 구성 구조
<img width="1415" height="1347" alt="Image" src="https://github.com/user-attachments/assets/8b71fad3-0675-4706-8fe4-e88bbadc8329" />


- **Data Center**
    - 전체 가상화 인프라를 포함하는 최상위 논리 단위
    - ESXi 호스트, Cluster, 네트워크, 스토리지 등을 포함
- **Cluster**
    - 여러 ESXi 호스트를 하나의 리소스 집합으로 묶어 관리
    - 자원 공유 및 VM 배치를 효율적으로 수행
- **Resource Pool**
    - Cluster 내에서 CPU, Memory 자원을 논리적으로 분할한 단위
    - VM을 그룹 단위로 관리하고 자원 할당 정책을 적용 가능

---

### Resource Pool 구성 의도

실습에서는 실제 기업 환경을 가정하여 Resource Pool을 다음과 같이 구성

- **Linux Resource Pool**
    - Linux 기반 VM을 위한 자원 영역
    - 내부적으로 IT, HR 부서 단위로 분리
- **Windows Resource Pool**
    - Windows 기반 VM을 위한 자원 영역
    - 동일하게 IT, HR 부서 단위로 분리

---

### IT / HR Resource Pool 의미

- **IT RP / HR RP**는 실제 조직 구조를 가정한 논리적 구분
- 각 부서별로 VM을 분리하여 관리함으로써:
    - 자원 사용을 명확하게 구분 가능
    - 향후 CPU/Memory 제한 또는 우선순위 설정 가능
    - 운영 환경과 유사한 구조를 실습 가능

---

### 정리

이 구조를 통해 다음과 같은 효과를 얻을 수 있음

- 자원을 단순히 VM 단위가 아니라 **조직/용도 기준으로 관리**
- Cluster 자원을 **논리적으로 분할하여 효율적인 운영 가능**
- 실제 기업 환경과 유사한 **계층적 인프라 구조 이해**

<br/>

---

## DHCP (Dynamic Host Configuration Protocol)


> 💡 DHCP는 네트워크에 연결된 장비에 IP 주소, 게이트웨이, DNS 등의 네트워크 정보를 자동으로 할당해 주는 서비스



### 왜 필요한가

실습 환경에서는 관리 네트워크(172.16.10.x), 스토리지 네트워크(172.16.40.x)처럼 직접 고정 IP를 부여하는 구간도 있었으나 외부 네트워크와 연결되는 일부 구간은 자동 IP 할당이 더 편리
DHCP를 사용하면 수동으로 IP를 일일이 입력하지 않아도 되므로 초기 구성과 테스트가 쉬워짐

### 실습 환경에서의 구성

- 인터넷 연결용 네트워크에서 DHCP를 사용
- 일부 인터페이스는 자동 IP를 받아 외부 연결 확인에 활용
- 관리망, vMotion망, 스토리지망은 목적이 명확하므로 주로 고정 IP 사용

### 적용 내용

- vmnic1 또는 외부 연결용 네트워크를 DHCP 기반으로 사용
- NAT/브리지 환경에서 자동 IP를 받아 인터넷 연결 여부를 확인
- 고정 IP가 필요한 관리 트래픽과는 분리하여 사용

### 확인 방법

- `ip addr` 또는 `nmcli` 명령으로 자동 할당된 IP 확인
- 게이트웨이 및 DNS 정보 확인
- 외부 ping 또는 패키지 설치 테스트로 네트워크 연결 검증

### 문제 발생 시 영향

DHCP가 정상 동작하지 않으면 외부 네트워크 연결이 되지 않아 

- 패키지 설치
- 업데이트
- 외부 리포지토리 접근 등의 작업이 어려워짐

<br/>

## NTP (Network Time Protocol)


> 💡 여러 서버와 장비의 시간을 동일하게 맞추기 위한 서비스



### 왜 필요한가

vSphere 환경에서는 ESXi 호스트, vCenter, DNS, 스토리지, VM의 시간이 서로 어긋나면
인증, 로그 분석, 서비스 연동, 장애 추적에 문제가 생길 수 있음
특히 클러스터 환경에서는 모든 호스트의 시간이 일관되게 유지되는 것이 매우 중요

### 실습 환경에서의 구성

- NTP 서버: 172.16.10.3
- 관리 네트워크 대역(172.16.10.x)을 통해 각 구성 요소가 NTP 서버를 참조
- ESXi 호스트 및 관련 VM이 동일한 기준 시간을 사용하도록 설정

### 적용 내용

- NTP 서버를 별도로 두고 각 시스템이 해당 서버를 참조하도록 구성
- ESXi 호스트에서 시간 동기화 서버로 NTP 서버를 등록
- 필요한 경우 리눅스 VM에서도 동일한 NTP 서버를 사용하도록 설정

### 🛠️ 설정 방법

1. **방화벽 설정 및 인바운드 추가**
    1. 방화벽 > 고급 설정 > 새 규칙
    2. 포트 선택 > UDP 선택 > 이름 설정 > 마침
<img width="1400" height="738" alt="Image" src="https://github.com/user-attachments/assets/a8357959-8580-428c-955f-5b61519c113f" />

<img width="359" height="400" alt="Image" src="https://github.com/user-attachments/assets/bcea0648-05d4-4fd2-9c92-f8333d7c0261" />

2. **gpedit.msc 수정**
    1. 로컬 그룹 정책 편집기 실행
    2. 컴퓨터 구성 > 관리 템플릿 > 시스템 > Windows 시간 서비스 > 시간 공급자 > Windows NTP 서버 사용 클릭
    3. 컴퓨터 구성 > 관리 템플릿 > 시스템 >Windows 시간 서비스 > 글로벌 구성 설정 클릭
  
<img width="889" height="609" alt="Image" src="https://github.com/user-attachments/assets/92e31c31-5c70-405c-8111-2f16d18271ad" />

<img width="865" height="596" alt="Image" src="https://github.com/user-attachments/assets/c193a18d-e175-4fe7-8740-5648931b79ad" />


출처 : https://ohsong-city.tistory.com/24

### 확인 방법

- ESXi에서 NTP 설정 및 서비스 상태 확인
- 리눅스에서 `timedatectl` 또는 NTP 관련 명령으로 동기화 상태 확인
- 여러 시스템의 현재 시간이 일치하는지 비교

### 문제 발생 시 영향

시간이 맞지 않으면

- 인증서 검증 실패
- vCenter/호스트 간 연결 문제
- 로그 시간 불일치로 인한 장애 분석 어려움
- 클러스터 운영 시 예기치 않은 문제
등이 발생할 수 있음

<br/>

## DNS (Domain Name System)


> 💡 IP 주소와 호스트 이름을 매핑하여 사용자가 숫자 IP 대신 이름으로 시스템에 접근할 수 있게 해 주는 서비스



### 왜 필요한가?

가상화 환경에서는 ESXi 호스트, vCenter, 각종 서버를 IP로만 관리하면 복잡하고 실수하기 쉬움
DNS를 사용하면 `vcenter.team2.com`처럼 의미 있는 이름으로 접근할 수 있어 관리가 편리
또한 일부 서비스는 이름 해석이 정확해야 정상적으로 동작

### 실습 환경에서의 구성

- DNS 서버: 172.16.10.2
- 관리 네트워크 대역에서 DNS 질의를 처리
- vCenter를 `vcenter.team2.com` 도메인으로 구성
- 주요 서버와 인프라 자원에 대해 이름 기반 접근 가능하도록 설정

### 적용 내용

- vCenter에 대한 DNS 레코드 등록
- 필요 시 ESXi 호스트 및 주요 VM에 대한 이름 등록
- 각 시스템의 DNS 서버 주소를 172.16.10.2로 지정하여 이름 해석 가능하도록 구성

### 🛠️ DNS 설정 방법

1. **역할 및 기능 추가**
    1. 서버 역할 > DNS 서버 선택 및 설치

<img width="940" height="639" alt="Image" src="https://github.com/user-attachments/assets/69c30ac1-884c-41d7-b159-be37db0f6562" />

<img width="956" height="559" alt="image" src="https://github.com/user-attachments/assets/1a0a5416-94d8-480f-8558-a55bf35ec749" />

2. **영역 파일 만들기**
    1. 도구 > DNS 선택 > 정방향 조회 영역에서 새 영역 > 주 영역 선택 및 영역 이름 설정
  
<img width="378" height="200" alt="image" src="https://github.com/user-attachments/assets/459b2cef-0af1-4e42-91c8-86759beea6dd" />

<img width="248" height="250" alt="image" src="https://github.com/user-attachments/assets/86db916d-85ed-4b78-817a-9a56d548a9f7" />

<img width="575" height="443" alt="Image" src="https://github.com/user-attachments/assets/ecb0be39-2742-48d0-81c5-012c506e0f57" />

3. **영역 파일 설정**
    1. 우클릭 > 새 호스트
    2. 도메인 이름, 등록할 IP 입력
  
<img width="699" height="478" alt="Image" src="https://github.com/user-attachments/assets/3f02c0e2-a87a-4c22-8fee-a4074d0ac44a" />

<img width="393" height="401" alt="Image" src="https://github.com/user-attachments/assets/1b50d44d-fb46-4498-a603-756c3a22e417" />

출처 : https://getmovie.tistory.com/entry/DNS-%EC%84%9C%EB%B2%84-%EC%84%A4%EC%A0%95-Window

### 확인 방법

- `nslookup vcenter.team2.com`
- `ping vcenter.team2.com`
- 각 서버에서 이름으로 상호 접근이 가능한지 테스트

### 문제 발생 시 영향

DNS가 정상 동작하지 않으면

- vCenter 접속 불편
- 이름 기반 관리 어려움
- 서비스 간 연결 실패 가능성
- 운영 시 IP를 직접 기억해야 하므로 관리 효율 저하가 발생할 수 있음


<br><br>

## 공유 스토리지(Shared Storage)

### 스토리지(Storage)란?
>💡
> **일반적 개념**
> - 컴퓨터 시스템에서 데이터, 애플리케이션, 운영체제 등을 전원이 꺼져도 보존되도록 영구적으로 저장하는 공간(하드웨어 장치 및 시스템)

>💡
> **vSphere 관점의 개념**
> - 가상 머신(VM)이 동작하기 위해 필요한 모든 파일(가상 디스크 파일(.vmdk), 가상 머신 설정 파일(.vmx) 등)이 실제 저장되는 물리적 공간
> - vSphere에서는 이러한 물리적 스토리지를 논리적으로 묶어 **데이터스토어(Datastore)**라는 형태로 포맷하여 ESXi 호스트에 제공

### 공유 스토리지(Shared Storage)란?
>💡
> **일반적 개념**
> - 특정 서버나 컴퓨터에 종속되지 않고, **다수의 서버가 네트워크를 통해 동시에 접근하여 데이터를 읽고 쓸 수 있도록 구성된 중앙 집중형 스토리지 환경**

>💡
> **vSphere 관점의 개념**
> - 여러 대의 **ESXi 호스트가 동일한 데이터스토어(Datastore)에 동시에 연결**되어 있는 상태
> - 가상 머신(VM)을 실행하는 컴퓨팅 자원(CPU, Memory)은 각 ESXi 호스트에서 제공하지만, VM의 실제 파일(운영체제, 데이터 등)은 모두 이 공유 스토리지(NAS 또는 SAN)에 저장

### 스토리지 종류

- **DAS (Direct Attached Storage : 직접 연결 스토리지)**
    - 서버나 컴퓨터 내/외부에 전용 케이블(SATA, SCSI 등)을 통해 직접 연결되는 스토리지 (예: 일반 PC의 C드라이브)
    - **블록(Block)** 단위로 데이터를 저장하고 접근함
- **NAS (Network Attached Storage : 네트워크 연결 스토리지)**
    - 일반적인 이더넷 네트워크(TCP/IP, LAN)를 통해 연결되는 네트워크 기반 스토리지
    - **파일(File)**로 데이터를 저장하고 접근
    - 운영체제(OS)가 탑재되어 있어 파일 서버 역할을 하며, 다수의 서버가 쉽게 데이터를 공유할 수 있음
    - **NFS(리눅스 전용)**, **SMB(윈도우 전용)** 등 프로토콜 사용
- **SAN (Storage Area Network : 스토리지 전용 네트워크)**
    - 일반 네트워크와 분리되어, 서버와 스토리지 간에 데이터를 전송하기 위해 구성된 **스토리지 전용 고속 네트워크**
    - **블록(Block) 단위**로 데이터를 저장. 즉, 서버가 SAN 스토리지를 마치 내부 로컬 디스크(DAS)처럼 인식함.
    - 전용 광 케이블(FC)과 스위치(FC Switch), 랜카드(HBA)가 있음

### vSphere에서 사용할 수 있는 공유 스토리지 종류

- **NFS(Network File System)**
    - 리눅스 전용 파일 공유 프로토콜
    - **파일 단위** 데이터 전송
    - vSphere는 NFS 스토리지를 **VMFS 파일 시스템으로 포맷할 필요 없이** 그대로 데이터 스토어로 사용 가능하다.
- **iSCSI(Internet Small Computer System Interface)**
    - SAN 기반 공유 스토리지 프로토콜
    - **블록 단위** 데이터 전송
    - 기존 **SCSI 어댑터를 소프트웨어적으로 가상화** 하여, **TCP/IP 프로토콜로 스토리지에 접근**할 수 있도록 하는 기술
    - vSphere에서 iSCSI를 데이터스토어로 사용할 때는 NFS와는 다르게 **VMFS 형태로 포맷이 필요**
    
    > **iSCSI 대상(Target)**: iSCSI 스토리지를 공유해 주는 서버
    **iSCSI 초기자(Initiator)**: iSCSI 스토리지를 연결해 사용하는 서버
    > 

### 공유 스토리지가 필요한 이유(vSphere 관점에서)

vSphere에서 아래 기능을 사용하려면 반드시 공유 스토리지가 필요하다.

- **vMotion**: VM을 다른 호스트로 옮기는 기술
- **DRS(Distributed Resource Scheduler)**: 호스트 상태를 분석하여 최적의 호스트로 자동으로 vMotion을 해주는 기술 ****
- **HA(High Availability)**: 호스트가 갑작스럽게 중지되었을 경우, 해당 호스트에서 구동되던 VM을 다른 호스트로 옮겨 재부팅 하는 기술
- **FT(Fault Tolerance)**: 호스트가 갑자기 중지될 경우를 대비하여 다른 호스트에 VM을 실시간 복사하는 기술
- **컨텐츠 라이브러리**: VM 템플릿이나 OVF, OVA, ISO 파일 등을 저장하고 사용자들과 공유할 수 있는 기능

> 최신 버전 vCenter에서는 공유 스토리지 없이도 vMotion이 가능하지만, 네트워크 사용량이 높아 권장되지는 않음
> 

### 윈도우 서버에 iSCSI 만들기

1. **IP 주소 설정**
    - 윈도우 서버에 네트워크 인터페이스 두 개를 아래와 같이 만든다.
        - **관리용 IP**: `172.16.10.100` → VM Network 포트 그룹과 연결
        - **공유 스토리지용 IP**: `172.16.40.1` → Storage-PG 포트 그룹과 연결
2. **iSCSI 역할 설치 (Target 서버)**
    - [서버 관리자] > [관리] > [역할 및 기능 추가] 클릭
    - 역할 서비스에서 [파일 및 저장소 서비스] > [파일 및 iSCSI 서비스] > [iSCSI 대상 서버]를 선택하여 설치
3. **iSCSI 가상 디스크 생성 및 초기자 입력**
    - [파일 및 저장소 서비스] > [iSCSI] 탭에서 **iSCSI용 가상 디스크 마법사**를 실행
    - 디스크 크기, 이름, 위치를 지정
    - 이 디스크에 접근할 초기자(클라이언트)의 IQN, DNS 이름, IP 주소 등을 매핑 
    
    
    >💡
    >**IQN(iSCSI Qualified Name)이란?**\
    >iSCSI 네트워크에서 대상(Target, 스토리지)과 초기자(Initiator, 클라이언트)를 고유하게 식별하기 위해 사용하는 표준 명명 규칙으로 IP 주소, DNS 보다 권장된다. 보통 아래와 같은 형태를 띈다.
    >
    >**iqn.yyyy-mm.{도메인역순}:{고유문자열}**
    >
    >예시:
    > - iqn.2013-01.com.myserver124
    > - iqn.2010-10.com.synology:target1
    > - vCenter에서 iSCSI 스토리지 어댑터를 추가하면 고유한 IQN이 표시된다.

### 실습 환경에 iSCSI 데이터스토어 만들기

- iSCSI 전용 커널 포트 추가
    1. 서버 ESXi 가상 스위치에 `172.16.40.2` 주소로 커널 포트 `iSCSI-VMK`를 만든다
    2. 서버 ESXi 가상 스위치에 iSCSI 연결을 위한 포트 그룹 `storage-PG` 를 만든다.
    
    
    <img width="1352" height="714" alt="iscsi-vmk" src="https://github.com/user-attachments/assets/797b5764-f06b-4663-acf8-c839f871411b" />

    <img width="1919" height="872" alt="vmk목록" src="https://github.com/user-attachments/assets/3b6c36b4-6ae0-4c97-bef3-98ee8e738fc7" />

    
    3. 다른 ESXi 호스트에서도 같은 네트워크 대역의 커널 포트를 만든다.

### iSCSI 전용 스토리지 어댑터 추가

1. 어댑터 추가

```bash
[스토리지 어댑터] > [어댑터 추가] > [iSCSI 어댑터 추가]
```

1. [네트워크 포트 바인딩]에 이전에 만든 전용 커널 포트 추가
2. [동적 검색]에 iSCSI 서버 주소 입력
3. [스토리지 재검색]

### 데이터스토어 생성

1. 위 과정을 성공적으로 진행했다면, 초기자 ESXi 호스트에서 데이터스토어를 만들때 스토리지 목록에 iSCSI 디스크가 표시된다.
2. iSCSI 디스크를 선택하고 VMFS 형식으로 포맷하면 공유 데이터스토어 사용 가능
    
    > 이 데이터스토어는 한 번만 만들어도 모든 초기자에게 표시된다.
    > 
<br>

## vMotion

### vMotin이란?

>- 가상머신(VM)이 동작 중인 상태에서 가상머신을 끄지 않고 다른 서버로 이동시키는 기술
>- Live Migration 또는 Hot Migration이라고도 함
>- **※**주의 사항**※**
    - 무중단 상태로 이동되는 것처럼 보이지만 빠르게 메모리를 복제하여 카피하는 방식
    데이터 손실에 민감한 DB 서버나 웹 서버의 경우는 사용을 주의가 필요

### 작동 원리

1. 상태 복사
    - Destination 호스트에 이동시킬 VM의 빈 껍데기 생성
2. 메모리 전송
    - 현재 VM의 메모리 상태를 Destination 호스트로 복사
3. 동기화
    - 전송할 메모리가 아주 적은 양만 남으면, 원본 VM을 밀리초 단위로 정지시키고 남은 데이터와 장치 상태를 전송
4. 전환
    - Destination 호스트에서 VM을 활성화하고, 원본 호스트의 VM을 삭제

### 사용 전 준비

1. ESXI 준비 사항
    1. 공유 스토리지 연결
        - vMotion이 진행될 원본 ESXi 서버와 대상 ESXi 서버는 공유 스토리지와 연결되어있어야 하며, 공유 스토리지에서 새성된 LUN에 같이 접근이 가능해야 함
        - 가상 머신 생성 시 가상 디스크 파일의 위치는 반드시 공유 스토리지의 데이터스토어야 함
    2. vMotion용 포트 그룹명과 가상 머신용 포트 그룹명의 일치
        - vMotion 용으로 생성된 포트 그룹명은 반드시 vMotion을 시행하려는 ESXi 서버끼리 일치해야 함
2. 가상 머신 준비 사항
    1. 외부용 업링크와 연결된 가상 스위치
        - 가상 머신이 Internal 가상 스위치와 연결되어 있을 경우 vMotion은 진행되지 않음
        반드시 외부 업링크와 연결된 가상 스위치와 구성돼야 함
    2. CPU Affinity 설정
        - CPU Affinity 설정이 강제적으로 되어 있으면 vMotion은 진행되지 않음
    3. 스냅샷 파일을 가지고 있는 경우
        - 스냅샷을 적용한 다음 지우지 않고 vMotion을 진행해도 동작은 하나, 스냅샷을 지우거나 revert한 후 vMotion을 실행하기를 권장
3. CPU 호환성 확인
    - vMotion을 실행하는 서버 간 다음 사항이 일치해야 함
        - CPU 제조 업체 (인텔 - 인텔, AMD - AMD)
        - CPU 세대

### 실습

#### 실습 환경 구성

- vMotion 전용 네트워크 대역을 분리하고 호스트별 VMkernel을 설정
- 네트워크 아키텍처
    - vMotion 전용망
        - 172.16.30.X / 24 대역을 할당
    - 가상 스위치 구성
        - 각 ESXi 호스트에 표준 스위치 vSwitch2를 생성하고 물리 어댑터를 매핑
            
            <img width="1348" height="748" alt="3" src="https://github.com/user-attachments/assets/aaaebcd3-a1f0-443e-b902-aea33a5ca7b8" />

            
        - VMkernel 설정
            - 각 호스트에 172.16.30.X / 24 대역의 vMotion 전용 IP 할당
            - ex)
                - ESXi-01 (172.16.10.11) : vmk IP 172.16.30.11
                - ESXi-02 (172.16.10.12) : vmk IP 172.16.30.12
- 스토리지 구성
    - Shared Storage
        - 모든 호스트가 172.16.40.1 (NFS) 대역의 공유 스토리지를 데이터스토어로 마운트하여 동일한 VM 파일에 접근하도록 설정

#### 실습 과정

1. 사전 점검
    - ESXi-01 (172.16.30.X / 24)과 ESXi-02(172.16.30.X / 24) 간의 통신 상태를 확인하여 vMotion 전용 통로가 정상 작동하는지 점검
2. 마이그레이션 실행
    - ESXi-01에서 구동 중인 Linux01(172.16.10.1x1 / 24) VM을 대상으로 마이그레이션 마법사 실행
3. 유형 선택
    - [계산 리소스 및 스토리지 모두 변경] 옵션을 선택하여 실행 호스트를 ESXi-02로 이전
4. 연속성 테스트
    - 이동 중 외부에서 ping 172.16.10.1x1 -t 명령어를 실행하여 네트워크의 연결이 끊기는지 확인
5. 결과 확인
    - 메모리 및 디스크 데이터의 복제가 완료되는 시점에 약간의 응답 지연이 발생하였지만, 세션의 끊김 없이 Linux01이 성공적으로 이동하는 것을 확인

<br>

## DRS(Distributed Resource Scheduler)

### DRS란?

>- 클러스터 내의 자원(CPU, 메모리) 상태를 5분마다 체크하여, 자원의 불균형이 발생했을 때 vMotion을 자동으로 실행해주는 기능

### 주요 기능

- Load Balancing
    - 특정 호스트에 부하가 쏠리지 않도록 VM을 재배치
- 초기 배치
    - VM 전원을 켤 때 가장 널널한 호스트를 자동으로 선택

### 작동 원리

1. 5분마다 실시간 모니터링
    - vCenter에서 클러스터에 있는 호스트들의 자원 상태를 5분 주기로 체크
    - 체크 대상
        - CPU 사용량, 메모리 점유율
2. 자원 불균형 감지
    - 각 호스트의 부하 정도를 계산하여 DRS 점수 산출
3. 최적의 배치 계산
    - vCenter에서 VM을 어느 서버로 보낼지 계산
    - Benefit > Cost인 경우에만 이동을 결정
    - 계산 고려 사항
        - Cost
            - vMotion을 실행하는 동안 호스트에 발생하는 부하
        - Benefit
            - 이동 후 VM이 얻게 될 성능 향상
4. vMotion 실행
    - 계산이 끝나면 설정한 자동화 수준에 따라 VM이 이동됨
        - 자동화 수준
        
        | 모드 | 설명 |
        | --- | --- |
        | 수동(Manual) | 이동 권장 사항만 제시하고, 실행은 관리자가 직접 클릭 |
        | 부분 자동화(Partially Automated) | VM 전원을 켤 떄의 배치는 자동이나, 운영 중 이동은 관리자의 승인이 필요 |
        | 완전 자동화(Fully Automated) | 관리자의 개입 없이 DRS가 판단하여 실시간으로 vMotion 실행 |

(+) Affinity Rules

- 자원의 효율성보다 관리자의 운영 정책이 더 우선 시 되어야 할 때 사용
- 이 규칙이 비용-편익 계산보다 더 높은 우선 순위를 가짐
- 종류
    
    
    | 종류 | 설명 | 장점 |
    | --- | --- | --- |
    | VM-VM Affinity | 두 VM을 항상 같은 호스트에 둠 | 호스트 내부 통신을 이용하므로 네트워크 속도가 빠름 |
    | VM-VM-Anti-Affinity | 두 VM을 절대 같은 호스트에 두지 않음 | 호스트 한 대가 장애로 죽어도, 다른 호스트에 있는 서버가 살아있어 서비스 가용성이 보장됨 |
    | VM-Host-Affinity | 특정 VM 그룹을 특정 호스트 그룹 안에서만 돌게 제한 |  |

### 사용 전 준비

- 2대 이상의 ESXi 호스트
    - 최소 2대 이상의 ESXi 호스트들이 하나의 클러스터로 묶여있어야 함
- 공유 스토리지
    - 모든 호스트가 동일한 데이터스토어를 보고 있어야 함
- vMotion 네트워크 구성
    - 각 호스트에 vMotion 서비스가 활성화된 VMkernel 포트가 있어야
- CPU 호환성 설정
    - 이동할 호스트 간의 CPU 제조사가 같아야하며, 세대가 다르면 기능을 맞춰야 함

### 실습

#### 실습 환경 구성

- 클러스터 설계
    - ESXi-01(172.16.10.x1 / 24)과 ESXi-02(172.16.10.x2 / 24)를 xxx-cluster로 그룹화
    - 클러스터 설정에서 vSphere DRS 기능을 활성화
- 자동화 수준 설정
    - 자동화 수준을 관리자가 검토를 하고 직접 승인하는 방식으로 설정하는 Manual 모드로 설정
       
         <img width="1350" height="736" alt="1" src="https://github.com/user-attachments/assets/23b0ae55-36d3-4a0a-9aab-3e30b8e4078b" />
        

#### 실습 진행 내용

1. 스트레스 테스트
    - ESXi-01 호스트의 Linux01 VM 내부에서 CPU에 부하를 발생시켜 해당 호스트의 자원 점유율이 90%(지정한 %) 이상이 되도록 유도
2. 불균형 감지
    - vCenter 모니터링 탭에서 해당 VM들의 DRS 점수가 하락하는 것을 확인
3. 권장 사항 확인
    - DRS 엔진의 분석 결과를 토대로, 자원의 여유가 있는 ESXi-02로 VM을 이동하라는 권장 사항 확인
4. vMotion 실행
    - 생성된 권장 사항에 대해 [적용] 버튼을 클릭하여 관리자의 승인 아래 수동 마이그레이션을 실행
5. 최종 검증
    - VM 재배치 완료 후, 클러스터 내 호스트들의 자원 사용률이 평준화되었고, 가상 머신들의 DRS 점수가 회복된 것을 확인
<img width="1341" height="710" alt="2" src="https://github.com/user-attachments/assets/22f0959b-74a2-4432-8d59-69bfa9c8926b" />

<br>

## Cluster

> VMware vCenter에서 **여러 ESXi 호스트를 하나로 묶어 관리**하는 논리적 단위
> 

여러 ESXi 호스트를 하나로 묶음으로써 CPU, Memory 등 자원을 통합 관리하는 구조

- 추후 등장할 HA, DRS, vMotion과 같은 기능을 사용하기 위한 **기반 환경**

---

### 사용 이유

1. 개별 서버 단위로 관리하면 **비효율적**
2. 특정 호스트에 부하 집중된다면 **성능 저하** 발생
3. 장애 발생 시 대응 어려움

<aside>

⇒ 따라서, Cluster를 사용함으로써 **자원 통합 + 자동 분산 + 장애 대응** 가능

</aside>

---

### 주요 기능

1. **리소스 통합 관리**
: 여러 호스트의 CPU, 메모리를 하나처럼 사용
2. **vMotion**
: VM을 다운타임 없이 다른 호스트로 이동
3. **DRS** (= 자동 자원 분배)
: 부하에 따라 VM을 자동으로 재배치
4. **HA**
: 호스트 장애 시 VM을 다른 호스트에서 자동 재시작

---

### 구조

```jsx
vCenter
 └─ **Cluster**
      ├─ ESXi Host 1
      ├─ ESXi Host 2
      └─ ESXi Host 3
```

---

### 동작 흐름

1. VM 실행 → 특정 Host에 배치
2. 부하 발생 → DRS가 다른 Host로 이동
3. Host 장애 발생 → HA가 VM 재시작

---

### 실습 과정
1. Cluster 생성

1-1) **Datacenter 우클릭 → New Cluster 생성**
    - ‘새 클러스터’ 클릭
    
    <img width="669" height="296" alt="Image" src="https://github.com/user-attachments/assets/72e43ee7-32f5-4f9f-97ae-87f54ed90908" />

    
1-2) **옵션 설정**
    - DRS 활성화
    - HA 활성화
    
    ![image.png](attachment:af0d2457-d561-49c4-84c9-e9d49f514139:image.png)
    

사진 출처: https://yoonix.tistory.com/56

2. ESXi Host 추가

2-1) **생성한 Cluster에 Host 추가**
    - Host를 Cluster로 끌어서 놓으면 이동됨
    
    → 각 Host의 CPU, Memory가 하나의 Pool로 통합됨
    

3. VM 배치 및 테스트

3-1) **VM을 특정 Host가 아닌, Cluster에 배치**
3-2) **vMotion을 통해 Host 간 이동 테스트**
    
    ⇒ 결과: VM 위치와 상관없이 서비스 지속 가능
    

---

### Cluster 구성 시 주의사항

- 모든 Host는 동일한 네트워크 및 스토리지 접근 필요
- vMotion을 위한 **VMkernel 설정** 필수
- HA 사용 시 충분한 여유 자원 확보 필요

---
<br><br>

## Resource Pool

### 개념

> **Cluster 내에서** CPU와 메모리를 논리적으로 **나누어 관리**하는 단위 <br>
→ VM들을 그룹으로 묶어서 **자원 사용을 제어**하는 구조
> 

---

### 사용 이유

1. VM 간 **자원 경쟁** 발생
2. 중요 서비스 **성능 저하** 가능성 존재

<aside>

⇒ Resource Pool을 통해, **자원 보장 + 제한 + 우선순위 제어** 가능

</aside>

---

### 주요 설정 요소

- **Reservation (= 최소 보장)**
: 특정 VM 또는 그룹에 최소 자원 보장
- **Limit (= 최대 제한)**
: 사용할 수 있는 자원의 최대치 제한
- **Shares (= 우선순위)**
: 자원이 부족할 때 누가 더 많이 사용할지 결정

---

### 구조

```jsx
Cluster
 ├─ **Resource Pool A** (개발)
 │   ├─ VM1
 │   └─ VM2
 └─ **Resource Pool B** (운영)
     ├─ VM3
     └─ VM4
```

---

### 동작 원리

1. 모든 VM이 자원 요청
2. Cluster 전체 자원 기준으로 분배
3. Resource Pool 정책 적용
    - Reservation → 먼저 확보
    - Shares → 우선순위 반영
    - Limit → 최대치 제한

---

### 실습 과정

1. Resource Pool 생성

1-1) **Cluster 우클릭 → New Resource Pool 생성**
1-2) **용도별 Pool 생성**
    
    ex) 개발 Pool, 운영 Pool etc . . .
    

2. VM 분류 및 할당

2-1) **각 VM을 용도에 맞는 Pool로 분류**
    
    ex) 테스트 VM → 개발 Pool
    
    ⇒ VM을 목적 기반으로 그룹화
    

3. 자원 정책 설정

3-1) **각 Pool별로 Reservation, Limit, Shares 설정**
    
    
    |  | **Reservation** | **Limit** | **Shares** |
    | --- | --- | --- | --- |
    | **운영 Pool** | 설정 - 최소 자원 보장 |  | High |
    | **개발 Pool** |  | 설정 - 자원 사용 제한 | Low |

4. 동작 테스트

4-1) **여러 VM을 동시에 실행**
4-2) **부하 상황 가정**
    
    ⇒ Resource Pool 설정값에 따른 결과:
    
    - 운영 Pool VM이 우선적으로 자원 확보
    - 개발 VM은 제한된 자원 내에서 동작

---

### Resource Pool 설계 시 주의사항

- 과도한 Reservation 설정 시 다른 VM 사용 불가
- Limit 설정 시 성능 저하 발생 가능
- Shares는 상대적인 값이므로 전체 구조 고려 필요


---

## Cluster + Resource Pool

```java
vCenter
 └─ Cluster
      ├─ ESXi Host 1
      ├─ ESXi Host 2
      ├─ Resource Pool (운영)
      │    └─ Web VM
      └─ Resource Pool (개발)
           └─ Test VM 
```
## 🔐 ESXi 사용자 계정 생성 및 권한 부여


>💡사용자 계정을 생성하는 것과 실제 관리 권한을 부여하는 것은 별개의 단계


 **계정 생성 = 로그인 가능**

 **권한 부여 = 작업 수행 가능**

---

### 1단계: 사용자 계정 생성 (Create User)

**접속 경로**

```
https://esxi07.team1.com 접속 → root 로그인
  → 관리 (Manage)
    → 보안 및 사용자 (Security & users)
      → 사용자 (Users)
        → 사용자 추가 (Add user)
```

---

### 2단계: 권한 할당 (Assign Permissions)

**접속 경로**

```
왼쪽 최상단 → 호스트 (Host)
  → 작업 (Actions)
    → 권한 (Permissions)
      → 권한 추가 (Add permission
```

<img width="1919" height="1013" alt="Image" src="https://github.com/user-attachments/assets/a2a0f016-3c54-49de-9d87-54c7a6e12212" />

<img width="1919" height="1013" alt="Image" src="https://github.com/user-attachments/assets/bdbcd22c-bf94-4d18-b939-c12f6482dc01" />

<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/ca846b8a-20d1-4f48-9082-f1a140cb52e1" />


---

## 🔒Lockdown Mode (잠금 모드)

>💡직접 접근(웹, SSH)을 차단하여 보안을 강화하는 기능

---

### 모드별 접근 제한 비교

| 접근 경로 | Disabled | Normal | Strict |
| --- | --- | --- | --- |
| vCenter 관리 | ✅ | ✅ (필수) | ✅ (필수) |
| ESXi 웹/SSH | ✅ | ❌ | ❌ |
| DCUI (물리 콘솔) | ✅ | ⚠️ (예외 사용자만) | ❌ |
| 보안 수준 | 낮음 | 높음 | 매우 높음 |

---

### 모드별 특징

### Disabled (기본)

- 모든 접근 허용
- 실습 환경에 적합
- 보안 취약

---

### Normal (권장)

- 웹 / SSH 차단
- DCUI는 예외 사용자만 접근 가능
- vCenter 장애 시 복구 가능

**기업 표준 모드**

---

### Strict (최고 보안)

- 모든 직접 접근 차단 (DCUI 포함)
- vCenter 없으면 관리 불가

 **금융/국방 수준 보안**

---

### Lockdown Mode 설정 방법

```
vCenter → ESXi 호스트 선택
  → 구성 (Configure)
    → 시스템 (System)
      → 보안 프로필 (Security Profile)
        → 잠금 모드 (Lockdown Mode) → 편집 (Edit)
```

모드 선택 (Normal / Strict)

---

## 예외 사용자 (Exception User)

>💡잠금 모드에서도 직접 접근 가능한 **특수 관리자 계정**

---

### 역할

| 잠금 모드 | 일반 사용자 | 예외 사용자 |
| --- | --- | --- |
| Disabled | 모든 접근 가능 | 의미 없음 |
| Normal | 직접 접근 불가 | DCUI 접근 가능 |
| Strict | 모든 접근 불가 | 유일한 접근 가능 계정 |

---

### 설정 방법

```
Lockdown Mode 설정 화면
  → 예외 사용자 (Exception User)
    → 사용자 추가
```

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/3b38a6db-abe5-4c23-8db0-2189e1a4b3ff" />


## 🔔  Alarm (알람 / 모니터링)

>💡알람(Alarm)은 VMware 환경에서 특정 조건(이벤트 또는 임계값)이 발생했을 때  **자동으로 상태를 감지하고 관리자에게 경고를 발생시키는 기능**

---

### 알람 구성 요소

알람은 다음 3가지 요소로 구성됩니다.

| 구성 요소 | 설명 |
| --- | --- |
| Trigger | 알람이 발생하는 조건 (CPU, Memory, 상태 변화 등) |
| Status | 상태 변화 (Green → Yellow → Red) |
| Action | 알림 방식 (이메일, 이벤트 기록 등) |

| 상태 | 의미 |
| --- | --- |
| 🟢 Green | 정상 |
| ⚠️ Yellow (주의) | 이상 징후 / 자동 복구 발생 |
| 🔴 Red | 심각한 장애 |

---

### 알람 생성 방법 (vCenter 기준)

### 1단계: 알람 생성

```
vCenter 접속
  → 객체 선택 (Host / VM / Datastore)
    → 구성 (Configure)
      → Alarm Definitions
        → New Alarm
```

---

### 2단계: 알람 설정

### 기본 설정

- 이름:  Host CPU Usage Alarm
- 대상: Host

---

### Trigger 설정

예: CPU 사용률 기반

- 조건: CPU Usage
- 임계값:
    - Warning: 80%
    - Critical: 90%

---

### Action 설정

- 알람 발생 시 이벤트 기록
- 상태 변경 시 알림 발생

<img width="1582" height="785" alt="Image" src="https://github.com/user-attachments/assets/cbb4945e-bd32-487d-9e14-fe32e61a5aa2" />

---

## HA(High Availability)

> 클러스터 내 호스트 (ESXi) 및 호스트 내의 VM/내부 애플리케이션에 장애 발생 시, VM을 클러스터 내의 정상 호스트에서 재시작 하거나 이상이 있는 VM을 재시동 시키는 장애 대처 방안

---

### HA가 필요한 3가지 사항

1. VM이 올라가 있는 호스트에 문제가 있는 경우
2. 호스트는 정상이지만 VM에 문제가 있는 경우
3. 호스트와 VM 모두 정상이지만 VM 내부 애플리케이션에 문제가 있는 경우

---

### 1. VM이 올라가 있는 호스트에 문제가 있는 경우

> 클러스터 내의 다른 호스트에 VM을 재시작함으로 장애 대응 가능

부하를 자원에 맞게 분산하는 DRS와 혼동하는 경우가 많지만 각 각의 목적이 장애 대응과 자원 활용성이라는 점에서 차이가 존재

### Admission Control

vSphere에서 HA를 달성하기 위해 Admission Control을 진행할 수 있는데, Admission Control이란 장애 발생 시 VM을 재시작할 수 있는 “여유 리소스를 미리 확보하는 정책”

Admission Control을 설정하지 않는 경우 평소에 클러스터 내의 모든 호스트 자원을 전부 사용하여 호스트에 장애가 생겨도 다른 호스트에 VM을 재시작할 자원이 없어 HA 동작 실패 가능

### Admission Control 방법


<img width="1376" height="1192" alt="image" src="https://github.com/user-attachments/assets/fb2f3df7-700a-42b1-be3c-4aa9aabc9c43" />
출처: https://www.bdrshield.com/blog/vmware-for-beginners-vsphere-ha-configuration-part-12c/

1. Host Failures Cluster Tolerates (가장 일반적)
    1-1. 클러스터 내 호스트 몇 대까지 장애 허용할 것인지 정함
    1-2. 가장 직관적으로 이해 가능
2. Percentage of Cluster Resources Reserved
    2-1. CPU/Memory를 %로 예약하여 HA를 구성하는 방식
    2-2. 클러스터가 클수록 더 유연하고 정확한 관리가 가능
3. Specify Failover Hosts
    3-1. 특정 호스트를 HA를 위한 **대기**용으로 비워두는 방식
    3-2. 자원 낭비가 심하기 때문에 거의 안씀
4. Slot 
    4-1. Slot: VM 하나가 필요로 하는 최소 리소스 단위
    4-2. Slot의 크기는 **가장 큰 VM**을 기준으로 하기 때문에 자원 낭비가 심함
    4-3. 과거에는 많이 사용하였으나 현재는 잘 사용되지 않음

---

### 2. 호스트에는 문제가 없지만 VM에 문제가 있는 경우

> VM을 해당 호스트에서 재시작하여 장애 대응

VM의 경우 Guest OS가 응답을 멈추거나 설정 문제로 인해 장애가 발생 가능
HA가 VMware Tools 기반의 VM heartbeat를 확인하여 응답이 없으면 VM만 재시작 하여 대응 가능
- 해당 경우 Heartbeat를 위한 데이터 스토어가 2개 이상 필요

---

### 3. 호스트와 VM 모두 정상이지만 내부 애플리케이션에 문제가 있는 경우

> DB나 WAS에 장애가 발생하여 애플리케이션만 장애가 발생한 경우 VM을 재시작하거나 애플리케이션 자체를 재시작 하는 방법으로 HA를 달성 가능

---

### HA 설정 방법

<img width="855" height="600" alt="image" src="https://github.com/user-attachments/assets/dd3c0ea7-57f8-494d-9646-e99d0a0a4ef5" />
출처: https://www.cloudbolt.io/vmware-administration/vmware-ha/

HA의 경우 다음과 같은 방식으로 설정 가능

1. vCenter의 Datacenter 내의 cluster를 우클릭
2. 설정 클릭
3. vSphere Availability 클릭
4. Edit 내의 vSphere HA 활성화

각 항목에 대한 설명

- Host Failure Response: 호스트 자체가 죽었을 때 설정
- Response for Host Isolation: 호스트는 정상이지만 클러스터와 통신이 안되는 고립 상태인 설정
- Datastore with PDL(Permanent Device Loss): 데이터 스토리지가 영구적으로 사라졌다고 판단되는 상태의 설정
- Datastore with APD(All Paths Down): 데이터 스토리지로 가는 모든 경로가 끊겨 있으나 영구적인지 아닌지 확실하지 않은 상태의 설정
- VM Monitoring: 호스트가 정상이고 VM이 켜져있으나 OS에 장애가 생긴 상태의 설정 (Heartbeat를 통해 판단)

---

## FT (Fault Tolerance)

> HA의 경우 VM 재시작을 위한 중단 시간이 존재하지만 서비스의 연속성이 매우 중요한 경우 서비스에 단 1초에 중단도 허용하지 않는 Fault Tolerance 를 이용

HA와는 다르게 FT는 VM을 실행 할 때 다른 호스트에 똑같은 복사본(VM)을 실시간으로 하나 더 만듬

- Zero Downtime: 주 서버의 VM에 장애가 생겨도 보조 서버의 VM이 즉시 이어받기 때문에 서비스 중단 시간이 0
- Zero Data Loss: 데이터 손실이 전혀 없고 네트워크 연결이 유지
- 실시간 동기화: Fast Checkpointing 기술을 사용하여 주 VM의 메모리 상태를 보조 VM에 지속적으로 복사하여 두 서버를 항상 동일한 상태로 유지

---

### FT의 단계별 동작 방식

1. 초기 복제: FT를 활성화 시 주 VM의 메모리 데이터를 보조 VM이 생성되는 호스트로 복제하여 보조 VM 생성
2. 데이터 동기화: 주 VM에서 발생하는 모든 변화를 **FT Logging Network** 를 통해 보조 VM으로 실시간 전달
3. 장애 발생 시 전환: 주 VM에 장애 발생 시 보조 VM이 즉시 주 VM 역할 시행
4. 자동 복구: 보조 VM으로 전환 이후 vSpehere HA와 연동하여, 다른 호스트에 새로운 보조 VM을 자동 생성하여 이중화 상태 복구

---

### FT 설정 방법

<img width="1346" height="852" alt="image" src="https://github.com/user-attachments/assets/baf48e3d-63ce-4791-87ff-72370223ca90" />
출처: https://www.bdrshield.com/blog/what-is-vmware-vsphere-fault-tolerance-and-how-does-it-works-part14/

1. FT 전용 커널 설정
    1. FT를 다른 네트워크와 같이 사용할 시, 트래픽이 몰려서 서비스 중단이 가능하기 때문에 별도의 커널 설정이 필요
    2. 이 때 생성하는 VMKernel의 경우 주 VM 상태 추적을 위해 Fault Tolerance logging을 체크 필요

<img width="1372" height="1042" alt="image" src="https://github.com/user-attachments/assets/612c5ff5-dab5-4890-af15-82a86c4fa3af" />
출처: https://www.bdrshield.com/blog/what-is-vmware-vsphere-fault-tolerance-and-how-does-it-works-part14/

2. Fault Tolerance 설정
    1. vSphere Client에서 FT를 설정할 VM 우클릭
    2. Fault Tolerance 클릭
    3. Turn On Fault Tolerance 클릭
    4. 주 VM, 보조 VM 생성 확인

---

### FT 설계 시 주의 사항

1. 리소스 제한: VM 당 최대 8개의 vCPU와 128GB 메모리까지만 지원
2. 호스트 당 제한: 한 개의 호스트에 FT VM은 최대 4개, vCPU는 8개까지만 올릴 수 있음
3. 네트워크: 실시간 데이터 복제를 위해 **FT Logging Network** 필요
4. 디스크: 보조 VM은 독립된 vmdk 파일을 가지기 때문에 디스크 자원이 2배로 필요함

---

## 트러블 슈팅

### 1. vCenter 서버 부팅 드라이브 인식 오류

> **현상**: vCenter 서버 초기 구축 후 부팅 시, OS가 설치된 드라이브를 찾지 못하고 부팅이 실패하는 현상 발생

- **원인**: 서버 내 여러 개의 드라이브 중 OS가 설치되지 않은 잘못된 드라이브가 부팅 우선순위 상단에 배치됨
- **해결 방법**: 
    1. 서버 부팅 시 BIOS(또는 Boot Menu) 진입
    2. 부팅 우선순위(Boot Priority) 설정 확인
    3. OS가 설치된 **NVMe 드라이브**를 최우선 순위로 변경 후 재부팅
- **결과**: 정상적으로 vCenter 서버 OS 진입 및 서비스 활성화 확인

---

### 2. FT(Fault Tolerance) 리소스 부족 및 환경 이전

> **현상**: 로컬 PC(RAM 32GB) 환경에서 FT 및 HA 구성을 시도했으나, Admission Control 정책 및 리소스 예약(Reservation) 조건 미충족으로 인해 장애 조치 구성 실패

- **원인**: 
    - **메모리 부족**: FT는 주 VM과 보조 VM의 메모리를 실시간 동기화하며 리소스를 2배로 점유함
    - **Admission Control 제약**: 클러스터 내 Failover를 위한 여유 자원(Slot, Percentage 등)을 확보해야 하는데, 32GB 환경에서는 가상 ESXi 호스트 2대와 VM들을 동시에 구동하기에 메모리 용량이 절대적으로 부족함
- **해결 방법**: 
    1. 실습 환경을 로컬 PC에서 **고사양 vCenter 서버(RAM 128GB)**로 이전
    2. 해당 서버 내부에서 **가상 ESXi 호스트 2대(Nested ESXi)**를 새롭게 생성
    3. 충분한 메모리 자원을 바탕으로 클러스터 자원 여유 공간 확보
- **결과**: 자원 부족 메시지 없이 HA 및 FT 기능이 정상적으로 활성화되었으며, 무중단 서비스 전환 테스트 성공

---
