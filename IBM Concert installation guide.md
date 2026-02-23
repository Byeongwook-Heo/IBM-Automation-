# IBM Concert 2.2.x VM 설치 가이드 (상세판)

> 목적: IBM Concert 2.2.x를 **VM 환경**에 설치할 때, 사전 준비부터 운영 전환까지 한 번에 수행할 수 있도록 정리한 실행 가이드입니다.
> 범위: 아래 IBM Docs 주제를 모두 반영한 운영 문서 템플릿입니다.
>
> - Installing on VM
> - Setting up alternate storage path
> - Setting up custom PostgreSQL configuration
> - Setting up external data repositories for Concert
> - Installing in air-gapped environment with private registry (VM)
> - Installing in air-gapped environment without private registry (VM)

---

## 0) 먼저 확인: 공식 문서 매핑

아래 링크를 설치 작업표에 직접 첨부해, 단계별로 체크하세요.

1. VM 설치: https://www.ibm.com/docs/en/concert/2.2.x?topic=vm-installing
2. 대체 스토리지 경로: https://www.ibm.com/docs/en/concert/2.2.x?topic=vm-setting-up-alternate-storage-path
3. PostgreSQL 커스텀 구성: https://www.ibm.com/docs/en/concert/2.2.x?topic=vm-setting-up-custom-postgresql-configuration
4. 외부 데이터 저장소: https://www.ibm.com/docs/en/concert/2.2.x?topic=vm-setting-up-external-data-repositories-concert
5. Air-gapped(사설 레지스트리 사용): https://www.ibm.com/docs/en/concert/2.2.x?topic=iiagev-installing-in-air-gapped-environment-private-registry-vm
6. Air-gapped(사설 레지스트리 미사용): https://www.ibm.com/docs/en/concert/2.2.x?topic=iiagev-installing-in-air-gapped-environment-without-private-registry-vm

> 중요: 버전별 파라미터/포트/이미지명은 변경될 수 있으므로, 실제 적용값은 위 공식 문서 기준으로 최종 확정합니다.

---

## 1) 사전 환경 구성 (Pre-flight)

## 1.1 인프라/OS

- [ ] 지원 OS/커널 버전 확인
- [ ] VM 사이징 확정 (POC/운영 분리)
- [ ] 데이터/로그/백업 경로 용량 확보
- [ ] NTP 동기화 완료
- [ ] 운영 계정 및 sudo 권한 구성

권장 분리 예시:

- `/opt/concert` : 애플리케이션 바이너리/설정
- `/var/lib/concert` : 애플리케이션 데이터
- `/var/log/concert` : 로그
- `/backup/concert` : 백업 산출물

## 1.2 네트워크/보안

- [ ] 사용자 → Concert UI FQDN HTTPS 접근 허용
- [ ] Concert → DB/SMTP/IdP/외부 연동 API 통신 허용
- [ ] 방화벽/보안그룹 최소 권한 정책 반영
- [ ] TLS 인증서(FQDN, SAN, 체인) 검증
- [ ] 내부 CA 사용 시 trust store 반영

## 1.3 계정/RBAC/감사

- [ ] 설치 서비스 계정 준비
- [ ] 운영자/개발자/조회자 권한 모델 확정
- [ ] 감사 로그 보존기간/보관 위치 정의
- [ ] 비밀정보(암호, 토큰)는 Vault 또는 Secret Manager에 저장

---

## 2) 설치 Workflow (운영 표준)

### 단계 1. 요청/범위 확정

- 설치 대상 환경(DEV/STG/PRD) 및 일정 확정
- 연동 시스템 목록(CI/CD, ITSM, 모니터링, SSO) 확정
- 민감정보 취급 여부 및 보안 요구사항 확정

### 단계 2. 설계/승인

- 아키텍처/네트워크/포트 정책 문서화
- 저장소/DB/백업 설계 확정
- CAB(변경심의) 또는 내부 승인 완료

### 단계 3. 구축/설치

- VM 준비 및 시스템 설정 적용
- Concert 설치 + 저장소/DB 설정 반영
- 설치 후 기능/헬스체크 수행

### 단계 4. 검증/UAT

- 로그인/권한/워크플로/알림/API 기능 검증
- 백업/복원 리허설
- 성능 및 장애 대응 시나리오 점검

### 단계 5. 운영 전환

- 변경 윈도우 반영
- 롤백 기준 및 책임자 공유
- 하이퍼케어 1~2주 운영

---

## 3) VM 기본 설치 절차 (vm-installing 반영)

> 아래 명령은 운영팀 템플릿 예시입니다. 실제 설치 스크립트/이미지/패키지명은 공식 문서 값으로 치환하세요.

## 3.1 패키지/사용자/디렉터리 준비

```bash
# OS 업데이트(배포판에 맞게 사용)
sudo dnf -y update || sudo apt-get update

# 서비스 계정 생성
sudo useradd -r -m -d /opt/concert -s /sbin/nologin concert || true

# 기본 경로 생성
sudo mkdir -p /opt/concert/{bin,config}
sudo mkdir -p /var/lib/concert /var/log/concert /backup/concert
sudo chown -R concert:concert /opt/concert /var/lib/concert /var/log/concert
```

## 3.2 환경 변수 파일 템플릿

`/opt/concert/config/concert.env`

```env
CONCERT_BASE_URL=https://concert.example.com
CONCERT_DATA_DIR=/var/lib/concert
CONCERT_LOG_DIR=/var/log/concert

# DB (내장/외부 여부에 따라 값 구성)
CONCERT_DB_HOST=db.example.internal
CONCERT_DB_PORT=5432
CONCERT_DB_NAME=concert
CONCERT_DB_USER=concert_user
CONCERT_DB_PASSWORD=<REDACTED>

# TLS
CONCERT_TLS_CERT=/opt/concert/config/tls.crt
CONCERT_TLS_KEY=/opt/concert/config/tls.key
```

## 3.3 서비스 등록/기동(예시)

```bash
sudo systemctl daemon-reload
sudo systemctl enable concert
sudo systemctl start concert
sudo systemctl status concert
```

## 3.4 1차 검증

```bash
# 로그 확인
journalctl -u concert -n 200 --no-pager

# 헬스체크/접속성
curl -Ik https://concert.example.com
```

검증 체크:

- [ ] 서비스 Active
- [ ] UI 접속 가능
- [ ] 관리자 로그인 성공
- [ ] 기본 워크플로 실행 성공

---

## 4) 대체 스토리지 경로 구성 (alternate storage path 반영)

대용량 로그/데이터를 OS 루트 디스크에 두지 않도록 별도 마운트 경로를 권장합니다.

## 4.1 파일시스템 마운트 예시

```bash
# 예: /dev/nvme1n1을 데이터용으로 사용
sudo mkfs.xfs /dev/nvme1n1
sudo mkdir -p /data/concert
sudo mount /dev/nvme1n1 /data/concert

# 재부팅 후 자동 마운트
UUID=$(blkid -s UUID -o value /dev/nvme1n1)
echo "UUID=${UUID} /data/concert xfs defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

## 4.2 경로 재지정 예시

```bash
sudo mkdir -p /data/concert/{data,logs,backup}
sudo chown -R concert:concert /data/concert

# 심볼릭 링크 또는 env 파일로 경로 전환
sudo ln -sfn /data/concert/data /var/lib/concert
sudo ln -sfn /data/concert/logs /var/log/concert
```

체크포인트:

- [ ] 재부팅 후 자동 마운트 정상
- [ ] 서비스 계정 쓰기 권한 정상
- [ ] 디스크 사용량/IOPS 모니터링 알림 설정

---

## 5) PostgreSQL 커스텀 구성 (custom postgresql configuration 반영)

> 운영 환경에서는 기본 DB 설정 대신, 워크로드에 맞춘 커스텀 파라미터 적용을 권장합니다.

## 5.1 필수 방향

- `max_connections`: 동시 사용자/API 호출량 반영
- `shared_buffers`, `work_mem`: 메모리 용량 기반 조정
- `wal_level`, `checkpoint_timeout`: 복구/성능 균형 고려
- `log_min_duration_statement`: 느린 쿼리 추적 기준 설정

## 5.2 적용 절차(일반 예시)

1. PostgreSQL 구성 백업
2. 파라미터 수정
3. 재시작 또는 reload
4. 연결/성능/오류 로그 검증

검증 쿼리 예시:

```sql
SHOW max_connections;
SHOW shared_buffers;
SHOW work_mem;
```

체크포인트:

- [ ] Concert 애플리케이션 DB 연결 정상
- [ ] 슬로우 쿼리 로그 기준 정상
- [ ] 피크 시간대 DB CPU/IO 병목 없음

---

## 6) 외부 데이터 저장소 구성 (external data repositories 반영)

Concert가 내부 저장소 외에 외부 저장소(예: 객체 스토리지/외부 DB)를 사용하도록 구성할 때의 표준 절차입니다.

## 6.1 준비

- [ ] 저장소 엔드포인트/DNS 준비
- [ ] 인증 정보(Access Key/Secret, TLS) 발급
- [ ] 네트워크 ACL 및 방화벽 허용
- [ ] 데이터 보존/암호화 정책 확정

## 6.2 구성 원칙

- 운영/백업 저장소 분리
- 최소 권한 정책(읽기/쓰기/관리 권한 분리)
- 전송 구간 TLS 강제
- 장애 시 재시도/타임아웃 정책 명확화

## 6.3 검증

- [ ] 연결 테스트 성공
- [ ] 읽기/쓰기 테스트 성공
- [ ] 접근 로그(감사 로그) 확인
- [ ] 장애 시 fallback/재시도 동작 확인

---

## 7) Air-gapped 설치 (사설 레지스트리 사용)

(참고: private registry VM 설치 주제)

## 7.1 흐름

1. 인터넷 연결 구간에서 이미지/패키지 수집
2. 사설 레지스트리로 이미지 푸시
3. 폐쇄망 VM에서 사설 레지스트리 pull로 설치
4. 체크섬/서명 검증 후 기동

## 7.2 체크리스트

- [ ] 이미지 태그/다이제스트 목록 고정
- [ ] 사설 레지스트리 인증서/자격증명 배포
- [ ] 취약점 스캔 및 승인 이력 보관
- [ ] 오프라인 설치 번들 버전 이력 관리

## 7.3 주의사항

- 태그 드리프트 방지(다이제스트 고정)
- 인증서 신뢰체인 미배포로 인한 pull 실패 사전 점검

---

## 8) Air-gapped 설치 (사설 레지스트리 미사용)

(참고: without private registry VM 설치 주제)

## 8.1 흐름

1. 온라인 환경에서 설치 번들/이미지 tar 확보
2. 무결성 체크섬 생성 및 승인
3. 물리 매체 또는 보안 파일전송으로 폐쇄망 반입
4. 대상 VM에 로컬 로드 후 설치 진행

## 8.2 체크리스트

- [ ] 번들 파일 체크섬(SHA256) 검증
- [ ] 반입 매체 보안 승인 이력 기록
- [ ] 반입 후 파일 권한/소유권 점검
- [ ] 설치 로그 및 산출물 별도 보관

---

## 9) 설치 후 운영 점검 (Day-2)

### Daily

- [ ] 서비스 헬스/API 응답 확인
- [ ] 실패 워크플로/적체 확인
- [ ] 중요 알림 누락 여부 확인

### Weekly

- [ ] 백업 성공 여부 + 샘플 복원 1건
- [ ] 계정/권한 변경 이력 점검
- [ ] DB/스토리지 임계치 추세 확인

### Monthly

- [ ] 패치/업그레이드 계획 갱신
- [ ] 인증서/비밀정보 만료 점검
- [ ] 용량 계획 재산정

---

## 10) 트러블슈팅 가이드

- **UI 접속 불가**: DNS, 인증서 체인, 리버스 프록시 헤더 확인
- **SSO 실패**: Redirect URI, 클레임 매핑, 서버 시간 동기화 확인
- **DB 연결 실패**: 방화벽, 계정 권한, SSL 모드, 파라미터 불일치 확인
- **성능 저하**: DB 파라미터/디스크 IOPS/워크플로 동시성 점검
- **Air-gapped 설치 실패**: 번들 무결성, 이미지 태그, 의존 파일 누락 확인

---

## 11) 인수인계 산출물

- 아키텍처/네트워크 다이어그램
- 설치 파라미터 파일(`concert.env`) 및 버전 이력
- DB/스토리지 구성값 및 변경 이력
- 백업/복원 Runbook + 리허설 결과
- 장애대응/롤백 Runbook
- 월간 운영점검 체크리스트

