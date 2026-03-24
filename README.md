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

