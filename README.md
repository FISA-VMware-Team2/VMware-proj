#  VMware 가상화 실습

| ![](https://avatars.githubusercontent.com/u/107402806?v=4) | ![](https://avatars.githubusercontent.com/u/113874212?v=4) | ![](https://avatars.githubusercontent.com/u/215618778?v=4) |![](https://avatars.githubusercontent.com/u/181299322?v=4)|![](https://avatars.githubusercontent.com/u/89902255?v=4)|![](https://avatars.githubusercontent.com/u/113014598?v=4)|
|:---:|:---:|:---:|:---:|:---:|:---:|
| **김소연**<br>[@thdus](https://github.com/thdus) | **사재헌**<br>[@Zaixian5](https://github.com/Zaixian5) | **서주영**<br>[@young2z1](https://github.com/young2z1) | **신성혁**<br>[@ssh221](https://github.com/ssh221) | **유예원**<br>[@Yewon0106](https://github.com/Yewon0106) | **주용완**<br>[@YongwanJoo](https://github.com/YongwanJoo) |

## 프로젝트 소개

우리 FISA 6기 클라우드 엔지니어링 과정에서 학습한 vSphere과 서버 구축을 실습하였다.

가상머신과 워크스테이션 환경에서 ESXi와 vCenter를 직접 설치하고 네트워크를 설정하며 가상화 인프라의 뼈대를 익히고, DNS, NTP, NAT 등의 서버를 직접 만들어 보며 프라이빗 클라우드 엔지니어링의 기초를 이해하는 것을 목표로 하였다.

<br/>

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

## 구성원 별 Data Center 개요



> 본 실습에서는 vSphere 환경에서 **Data Center → Cluster → Resource Pool 구조**를 구성하여 가상 자원을 효율적으로 분배하고 관리하는 구조를 구현



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

---

## DHCP (Dynamic Host Configuration Protocol)

<aside>
💡

DHCP는 네트워크에 연결된 장비에 IP 주소, 게이트웨이, DNS 등의 네트워크 정보를 자동으로 할당해 주는 서비스

</aside>

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
- 외부 리포지토리 접근
    
    등의 작업이 어려워짐
    

## NTP (Network Time Protocol)

<aside>
💡

여러 서버와 장비의 시간을 동일하게 맞추기 위한 서비스

</aside>

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

1. **gpedit.msc 수정**
    1. 로컬 그룹 정책 편집기 실행
    2. 컴퓨터 구성 > 관리 템플릿 > 시스템 > Windows 시간 서비스 > 시간 공급자 > Windows NTP 서버 사용 클릭
    3. 컴퓨터 구성 > 관리 템플릿 > 시스템 >Windows 시간 서비스 > 글로벌 구성 설정 클릭
  
<img width="889" height="609" alt="Image" src="https://github.com/user-attachments/assets/92e31c31-5c70-405c-8111-2f16d18271ad" />

<img width="865" height="596" alt="Image" src="https://github.com/user-attachments/assets/c193a18d-e175-4fe7-8740-5648931b79ad" />

<img width="865" height="596" alt="Image" src="https://github.com/user-attachments/assets/fd30dd4b-35fe-4656-b6a6-9cfe33480303" />

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

## DNS (Domain Name System)

<aside>
💡

IP 주소와 호스트 이름을 매핑하여 사용자가 숫자 IP 대신 이름으로 시스템에 접근할 수 있게 해 주는 서비스

</aside>

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
