---
# the default layout is 'page'
icon: fas fa-info-circle
order: 3
---

## 소개

로봇 중앙 관제 시스템을 개발하고 있습니다. 관심사가 많아서 이것저것 공부하고 있습니다.

## 할 수 있는 것

**주로 하는 일**
- **Go**: 로봇 연동 서버와 현장 제어 로직을 설계하고 구현합니다. 사내 Go readability 리뷰어로 코드 리뷰도 합니다.
- **gRPC / Protobuf**: 여러 로봇 플랫폼을 하나의 API로 다루도록 설계하고 구현합니다.
- **MQTT (EMQX)**: 로봇 상태 발행과 명령 처리를 맡고 있습니다.
- **VDA 5050**: 서드파티 AMR 연동과 프로토콜 적합성 검증을 합니다.
- **설계와 검증**: 설계 문서부터 구현, 테스트, 현장 배포까지 직접 합니다.

**실무에서 써 본 것**
- 공장 설비 PLC 연동(Modbus TCP) 시나리오 설계와 테스트 도구
- 현장 Linux 서버 구축 (네트워크 이중화 포함)
- 시리얼/TCP 장치 연동
- Docker · Kubernetes(배포 모듈 작성) · Terraform · GCP(Cloud Run) · MySQL · RabbitMQ · Python(스크립트·테스트 도구)

**공부하고 있는 것**
- [Linux 컨테이너 내부](/categories/container/), [운영체제](/categories/os/), [TLS/mTLS](/categories/network/), [Go 런타임](/categories/go/)

## 다음에 쓸 글

**보안 / 네트워크**
- Vault는 무엇이고, 인증서 갱신은 어떻게 하는지
- TLS 사이에 프록시와 LB가 끼어들면 어떻게 되는지
- k8s Service는 실시간으로 바뀌는 Pod IP를 실제로 어떻게 관리하는가

**OS / 컨테이너**
- OS 영속성 파트 (OSTEP Persistence)
- 컨테이너의 PV (Persistent Volume)
- CRI, CSI, CNI
