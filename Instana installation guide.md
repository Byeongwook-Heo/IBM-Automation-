# Instana installation guide (Self-Hosted Standard Edition, VM 기준)

> 문서 목적: **일반 VM 환경**에서 IBM Instana **Self-Hosted Standard Edition**(요청하신 “sefe hosted” 버전으로 이해) 백엔드와 Linux 에이전트를 설치하고, OpenTelemetry 연동 및 프록시 구성까지 한 번에 진행할 수 있도록 정리한 가이드입니다.  
> 작성 시 IBM 공식 문서의 설치/용량/네트워크/에이전트/OTel/프록시 가이드를 기준으로 정리했습니다(반드시 최신 공식 문서로 최종 값 확인).

---

## 1) 설치 아키텍처 개요

- **Backend(Self-Hosted)**
  - Instana Core 서비스(수집, 처리, UI)
  - 데이터 저장소(ClickHouse, Cassandra, Elasticsearch 등 Instana 버전에 따른 구성)
  - Object Storage(S3 호환/파일 기반 등 배포 방식에 따라 선택)
- **Agent(Linux)**
  - 각 대상 VM/호스트에 설치되어 메트릭/트레이스/로그/인프라 정보를 수집
- **OpenTelemetry(Optional + 권장)**
  - 애플리케이션에서 OTel SDK/Collector로 트레이스(및 메트릭/로그)를 전송
  - Collector 또는 앱에서 Instana 수집 엔드포인트로 전달
- **Proxy(Optional)**
  - Agent ↔ Backend 사이에 인터넷/망 분리 환경이 있을 경우 HTTP/HTTPS 프록시 경유

---

## 2) 사전 준비사항

### 2.1 필수 정보 체크리스트

- [ ] 설치 대상 구분: **Demo** 또는 **Production**
- [ ] Instana 라이선스/오프라인 아티팩트 접근 권한
- [ ] DNS/FQDN 계획(예: `instana.example.com`)
- [ ] TLS 인증서(사내 CA 또는 공인 인증서)
- [ ] NTP 시간 동기화
- [ ] 방화벽/보안그룹 정책 합의
- [ ] 백업 정책(스냅샷/오브젝트 스토리지/데이터 보존 기간)
- [ ] 운영 계정 및 sudo 권한

### 2.2 최소 요구사항(공식 문서 확인 필수)

> 아래는 **VM 기반 설계 시 자주 쓰는 기준 형태**이며, 정확한 CPU/RAM/디스크/IOPS는 Instana 공식 문서의 최신 버전으로 반드시 확정하세요.

#### Demo 환경(POC/검증)

- 노드 수: 최소 구성(소규모 단일/저노드)
- 용도: 기능 검증, 단기 테스트
- 특징:
  - 고가용성(HA) 미적용 가능
  - 리텐션 짧게(예: 수일~1~2주)
  - 성능/가용성 보장 대상 아님

#### Production 환경(운영)

- 노드 수: 역할별 분리 + 확장 가능한 토폴로지
- 용도: 실제 서비스 모니터링
- 특징:
  - HA(다중 노드/다중 AZ 권장)
  - 충분한 스토리지 성능(IOPS/Throughput) 확보
  - 백업/복구/모니터링/알림 정책 필수
  - 리텐션/수집량 기반 용량 계획 필수

### 2.3 OS/플랫폼 권장

- Backend/Agent용 Linux 배포판: Instana 지원 매트릭스에 포함된 버전 사용
- 커널/파일시스템/xfs 또는 ext4 권장 여부는 공식 문서 기준 적용
- SELinux/AppArmor 정책은 설치 가이드와 충돌 없는지 선검증

---

## 3) 네트워크 및 보안 설계

### 3.1 네트워크 요구사항

- Agent → Backend 통신 허용
- 사용자 브라우저 → Instana UI(FQDN, HTTPS) 허용
- Backend 내부 컴포넌트 간 포트 허용
- DNS 해석 가능해야 하며, 프라이빗 DNS 사용 시 모든 노드에서 동일 해석 필요

### 3.2 포트 정책

- 인바운드/아웃바운드 포트는 **공식 포트 매트릭스** 기준으로 오픈
- 운영에서는 최소 권한 원칙 적용:
  - 관리 네트워크 전용 포트 분리
  - Agent 수집 포트와 UI/API 포트 분리

### 3.3 TLS/인증서

- 운영 환경은 HTTPS 강제
- 인증서 체인(중간/루트 CA 포함) 점검
- Agent/Collector가 서버 인증서 검증 가능하도록 CA 배포

---

## 4) UUID 및 스토리지 구성

### 4.1 UUID 관리

- Instana 설치/클러스터 식별에 필요한 UUID(또는 cluster/tenant 식별값)는 설치 전에 확정
- UUID 관리 원칙:
  - 환경별 분리(Demo/Prod UUID 분리)
  - 문서화(CMDB/Secret Vault 보관)
  - 재배포 자동화 시 동일 환경 UUID 재사용 정책 명확화

### 4.2 스토리지 설계

- 최소 항목:
  - OS 디스크
  - 데이터 디스크(백엔드 데이터 저장)
  - 백업 대상 볼륨/오브젝트 스토리지
- 권장 사항:
  - 데이터 디스크는 고성능(SSD/NVMe) 사용
  - 데이터/로그 분리 마운트
  - 파일시스템 마운트 옵션(공식 권장값) 적용
  - 디스크 사용량/IOPS 모니터링 임계치 사전 정의

### 4.3 리텐션 기반 용량 산정

- 산정 요소:
  - 에이전트 수
  - 초당 메트릭/트레이스 수집량
  - 로그 연계 여부
  - 보관 기간(일)
- Production은 초기 산정치 대비 최소 30~50% 버퍼 권장

---

## 5) Backend 설치 절차(Self-Hosted Standard)

> 실제 설치 명령/스크립트는 배포 방식(온라인/오프라인, 패키지/컨테이너)에 따라 다르므로 IBM 공식 문서의 해당 섹션을 그대로 따르세요.

### 5.1 VM 준비

1. 역할별 VM 생성(Demo는 축소, Prod는 분리/확장)
2. 호스트네임, /etc/hosts 또는 DNS 등록
3. 시간 동기화(NTP) 확인
4. 필수 패키지 설치(curl, jq, tar 등 운영 표준)
5. 방화벽 룰 적용

### 5.2 설치 자산 준비

- 라이선스 파일/키
- 설치 번들(오프라인이면 이미지/패키지 미러)
- TLS 인증서 및 키
- UUID/환경 변수 파일

### 5.3 Instana backend 배포

1. 설치용 사용자/권한 준비
2. 공식 설치 스크립트 또는 Helm/Operator(해당되는 방식) 실행
3. 스토리지 경로/클래스/볼륨 매핑
4. 인증서 적용
5. 초기 관리자 계정 생성
6. 상태 점검(서비스 헬스체크, UI 접속)

### 5.4 설치 검증

- UI 로그인 가능 여부
- 백엔드 서비스 상태 Green/Healthy
- 이벤트/인프라 페이지 로딩 정상
- 기본 알림 채널 테스트(선택)

---

## 6) Linux Agent 배포

### 6.1 대상 서버 공통 준비

- 지원 OS 확인
- 호스트별 outbound 통신(backend 또는 proxy) 허용
- root 또는 sudo 설치 권한 확보

### 6.2 Agent 설치(표준 흐름)

1. Instana UI에서 Agent 설치 스크립트 확인
2. Linux 배포판별 명령 실행(예: shell installer 또는 패키지 방식)
3. Agent 키(토큰) 및 backend endpoint 설정
4. 서비스 시작 및 부팅 시 자동 시작 등록

### 6.3 Agent 상태 확인

- 서비스 상태 확인: `systemctl status instana-agent`
- 로그 확인: `/var/log/instana/agent.log`(환경별 경로 상이 가능)
- UI에서 Host/Process 식별 여부 확인

### 6.4 대량 배포 권장

- Ansible/Salt/SSH 병렬 배포
- Golden Image 또는 cloud-init에 agent bootstrap 포함
- 환경별 토큰/엔드포인트를 변수화

---

## 7) Agent ↔ Backend 사이 Proxy 구성

프록시 서버가 필요한 경우 Agent 시스템 서비스 환경변수에 프록시를 주입합니다.

### 7.1 프록시 변수 예시(systemd)

`/etc/systemd/system/instana-agent.service.d/proxy.conf`

```ini
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=http://proxy.example.com:3128"
Environment="NO_PROXY=localhost,127.0.0.1,.internal.example.com,instana-backend.example.com"
```

적용:

```bash
sudo systemctl daemon-reload
sudo systemctl restart instana-agent
```

### 7.2 점검 포인트

- 프록시에서 backend FQDN으로 TLS 통신 허용
- 인증 프록시라면 계정/비밀번호 보안 저장 방식 점검
- NO_PROXY에 내부 주소 누락 없도록 검토

---

## 8) OpenTelemetry 추가 방법

Instana는 OpenTelemetry를 통해 애플리케이션 텔레메트리를 수집할 수 있습니다.

### 8.1 연동 방식

- 방식 A: 애플리케이션 OTel SDK → OTel Collector → Instana
- 방식 B: 애플리케이션 OTel SDK → Instana 엔드포인트(환경에 따라 직접 전송)

운영에서는 **Collector 경유(방식 A)**를 권장합니다(파이프라인 제어/재시도/버퍼링 유리).

### 8.2 OTel Collector 예시 구성(개념 예시)

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch: {}

exporters:
  otlp:
    endpoint: instana-backend.example.com:4317
    tls:
      insecure: false

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

> 실제 endpoint/protocol/header(토큰) 값은 Instana 공식 OTel 문서 기준으로 설정하세요.

### 8.3 애플리케이션 설정 포인트

- 서비스명(`service.name`) 표준화
- 환경 태그(`deployment.environment`) 통일
- 샘플링 정책(특히 Production) 정의
- 민감정보 마스킹 정책 적용

### 8.4 검증

- Instana UI에서 서비스/트레이스가 생성되는지 확인
- Collector 로그에서 export 오류(인증/TLS/네트워크) 확인

---

## 9) Demo vs Production 운영 기준

### Demo

- 빠른 구축 우선
- 단일 AZ, 낮은 리텐션 허용
- 백업/DR 최소화 가능

### Production

- HA/확장성 우선
- 성능 시험(피크 수집량) 후 전환
- 백업/복구 리허설 필수
- 변경관리(CAB), 점검창, 롤백 절차 문서화

---

## 10) 트러블슈팅 체크리스트

1. **Agent가 안 보임**
   - 토큰/엔드포인트 오입력
   - 방화벽/프록시 차단
   - TLS 인증서 신뢰체인 문제
2. **UI 접속 불가**
   - DNS/FQDN 오류
   - 인증서 CN/SAN 불일치
   - LB/Ingress 설정 누락
3. **OTel 데이터 미수집**
   - Collector pipeline/exporter 오류
   - OTLP 포트 차단
   - 리소스 속성 미설정으로 서비스 매핑 실패

---

## 11) 공식 문서 참조(반드시 최신판 확인)

- IBM Instana Self-Hosted 설치 가이드
- IBM Instana 시스템 요구사항/사이징 가이드
- IBM Instana 네트워크/포트 요구사항
- IBM Instana Host Agent(Linux) 설치 가이드
- IBM Instana OpenTelemetry 연동 가이드
- IBM Instana 프록시 환경 설정 가이드

> 위 항목은 버전별로 경로/요구사항이 바뀔 수 있으므로, 설치 직전 IBM 공식 문서의 해당 버전(예: 현재 운영 예정 버전)의 값을 최종 기준으로 삼으세요.
