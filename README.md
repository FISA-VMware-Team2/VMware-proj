# 📍 VMware 가상화 실습

## 프로젝트 소개

우리 FISA 6기 클라우드 엔지니어링 과정에서 학습한 vSphere과 서버 구축을 실습하였다.

가상머신과 워크스테이션 환경에서 ESXi와 vCenter를 직접 설치하고 네트워크를 설정하며 가상화 인프라의 뼈대를 익히고, DNS, NTP, NAT 등의 서버를 직접 만들어 보며 프라이빗 클라우드 엔지니어링의 기초를 이해하는 것을 목표로 하였다.

<br/>

## ✍️ 전체 아키텍처
<img width="5586" height="2888" alt="Image" src="https://github.com/user-attachments/assets/c3a30625-9425-4e09-996b-e4ded026a540" />

<br>

## 🌐 네트워크 설계
### 📌 네트워크 분리

| 용도 | 대역 | 설명 |
|Management & VM Network 망 (O Black)|172.16.10.X / 24|호스트 관리 및 VM의 서비스 통신용|
|Fault Tolerance 망 (O Red)|172.16.20.X / 24|FT 기능을 위한 호스트 간 실시간 데이터 동기화용|
|vMotion 망 (O Pink)|172.16.30.X / 24|vMotion 시 메모리 데이터 전송 전용|
|Storage 망 (O Green)|172.16.40.X / 24|공유 스토리지(ISCI / NFS) 접근 전|

### 📌 설계 의도
- 트래픽 분리 → 성능 + 안정성 확보
- vMotion / FT 전용망 확보
