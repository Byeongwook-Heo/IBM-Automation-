# Kubecost 설치 가이드 (IBM Cloud Kubernetes Service / Red Hat OpenShift on IBM Cloud 기준)

> 문서 목적: IBM Cloud 환경(특히 **IKS** 및 **ROKS**)에서 Kubecost를 설치하고, 비용 가시화/할당(Allocation)/알림까지 운영 가능한 형태로 구성하기 위한 실무 가이드입니다.  
> 작성 기준: IBM Cloud 및 Kubecost 공식 문서의 일반적인 설치 흐름을 바탕으로 정리했습니다. 실제 적용 전에는 반드시 사용 중인 클러스터 버전과 IBM 공식 문서 최신판을 확인하세요.

---

## 1) 아키텍처 개요

- **Kubecost Core 구성요소**
  - Cost Analyzer(UI/API)
  - Prometheus(또는 기존 Prometheus 연동)
  - ETL/클라우드 비용 수집 컴포넌트
- **대상 클러스터**
  - IBM Cloud Kubernetes Service(IKS)
  - Red Hat OpenShift on IBM Cloud(ROKS)
- **권장 운영 방식**
  - 운영 클러스터에는 고가용성/리소스 보장(요청/제한) 적용
  - 비용 데이터 보존 기간(retention) 및 저장소 용량 계획 사전 확정

---

## 2) 사전 준비사항

### 2.1 접근/권한

- [ ] IBM Cloud 계정 및 대상 리소스 그룹 권한
- [ ] `ibmcloud` CLI 설치 및 로그인
- [ ] Kubernetes/OpenShift 관리자 권한(`cluster-admin` 또는 동등 권한)
- [ ] Helm 3.x 설치

### 2.2 클러스터/네트워크

- [ ] 클러스터 버전이 Kubecost 지원 범위인지 확인
- [ ] 설치 네임스페이스(예: `kubecost`) 결정
- [ ] Ingress/Route 노출 방식(사내망/공인망) 합의
- [ ] 사내 프록시/방화벽 환경일 경우 이미지 레지스트리 접근 허용

### 2.3 보안/거버넌스

- [ ] TLS 적용 정책(내부 CA/공인 인증서) 확정
- [ ] Secret(클라우드 비용 API 키 등) 저장 정책(Vault/Sealed Secret 등) 확정
- [ ] 비용 데이터 보존 및 백업 정책 확정

---

## 3) 클러스터 접속 설정

### 3.1 IKS/ROKS 공통 로그인 예시

```bash
ibmcloud login --sso
ibmcloud ks cluster config --cluster <CLUSTER_NAME_OR_ID>
kubectl config current-context
```

ROKS를 사용하는 경우 OpenShift CLI(`oc`)로도 접속 확인:

```bash
oc whoami
oc project
```

### 3.2 네임스페이스 생성

```bash
kubectl create namespace kubecost
```

> 이미 존재하면 해당 단계는 생략 가능합니다.

---

## 4) Helm으로 Kubecost 설치

> 아래는 일반적인 Helm 설치 예시입니다. 실제 값(`values.yaml`)은 IBM 공식 가이드 및 운영 정책(스토리지 클래스, 보안 정책, ingress/route 정책)에 맞게 조정하세요.

### 4.1 Helm 저장소 등록

```bash
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm repo update
```

### 4.2 values 파일 작성(예시)

`kubecost-values.yaml`

```yaml
global:
  prometheus:
    enabled: true

kubecostModel:
  etl:
    enabled: true

prometheus:
  server:
    persistentVolume:
      enabled: true
      size: 64Gi

serviceAccount:
  create: true
```

### 4.3 설치 실행

```bash
helm upgrade --install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  -f kubecost-values.yaml
```

### 4.4 배포 상태 확인

```bash
kubectl get pods -n kubecost
kubectl get svc -n kubecost
helm list -n kubecost
```

---

## 5) OpenShift(ROKS) 환경 추가 고려사항

- SCC(Security Context Constraints) 정책으로 인해 Pod 실행 권한이 제한될 수 있으므로, Kubecost 차트 권장 설정값을 우선 사용
- Route 노출 시 TLS 종료 지점(Edge/Passthrough/Re-encrypt) 정책을 사전 결정
- OpenShift Monitoring/사용 중인 Prometheus와 중복 수집 여부 점검

Route 예시(개념):

```bash
oc expose svc kubecost-cost-analyzer -n kubecost
oc get route -n kubecost
```

> 실제 서비스명은 Helm 차트 버전에 따라 다를 수 있습니다.

---

## 6) IBM Cloud 비용 연동 포인트

Kubecost에서 클라우드 비용 정확도를 높이려면 IBM Cloud 청구/사용량 데이터 연동이 필요할 수 있습니다.

- IBM Cloud 계정 단위 비용 데이터 접근 권한 확인
- 계정/리소스 그룹/태그 전략 표준화
- 비용 할당 태그(`team`, `service`, `env` 등) 강제 정책 적용

권장 순서:

1. 태그 표준 수립(예: `owner`, `cost-center`, `environment`)
2. 네임스페이스/워크로드 라벨 표준화
3. Kubecost Allocation 리포트에서 미분류(Unallocated) 비율 최소화

---

## 7) 접근 제어 및 노출

### 7.1 포트포워딩(초기 검증용)

```bash
kubectl port-forward --namespace kubecost deployment/kubecost-cost-analyzer 9090
```

브라우저에서 `http://localhost:9090` 접속.

### 7.2 운영 노출

- IKS: Ingress + TLS + 인증(사내 SSO/IdP)
- ROKS: Route + TLS + OAuth Proxy/외부 인증

운영 권장사항:

- 공인 노출 시 IP 제한 또는 WAF 적용
- 기본 계정/약한 인증 설정 금지
- 감사 로그 및 접근 로그 보관

---

## 8) 설치 후 검증 체크리스트

- [ ] Kubecost UI 정상 접속
- [ ] 최근 24시간 비용 데이터 표시
- [ ] 네임스페이스/워크로드별 Allocation 확인
- [ ] Prometheus 메트릭 수집 정상
- [ ] 리소스 요청/제한값 미설정 워크로드 식별 가능

유용한 점검 명령:

```bash
kubectl get all -n kubecost
kubectl logs -n kubecost deploy/kubecost-cost-analyzer --tail=200
kubectl top pod -n kubecost
```

---

## 9) 운영 권장사항 (Production)

- **리소스 관리**: Kubecost 컴포넌트에 requests/limits 명시
- **스토리지 관리**: Prometheus PV 및 ETL 데이터 증가량 모니터링
- **업그레이드 관리**: Helm 차트/앱 버전 업그레이드 전 스테이징 검증
- **백업**: values 파일/Secret/대시보드 설정 백업
- **거버넌스**: 태그·라벨 표준 준수율을 KPI로 관리

---

## 10) 트러블슈팅

1. **비용 데이터가 비어있음**
   - 클라우드 비용 연동 설정 누락
   - Prometheus 데이터 부족(수집 시작 직후)
2. **Pod CrashLoopBackOff**
   - 리소스 부족(CPU/메모리)
   - 보안 정책(SCC/PSP/OPA) 충돌
3. **UI 접속 실패**
   - Service/Ingress/Route 설정 오류
   - TLS 인증서 체인 문제
4. **비용 할당 정확도 낮음**
   - 태그/라벨 표준 미적용
   - Unallocated 리소스 과다

---

## 11) IBM 공식 문서 참조 (최신판 확인 필수)

- IBM Cloud Kubernetes Service 문서(클러스터 접속/운영)
- Red Hat OpenShift on IBM Cloud 문서(Route, 보안, 운영)
- IBM Cloud CLI(`ibmcloud ks`, `oc`) 관련 문서
- Kubecost 공식 설치 문서(Helm chart values 및 버전 호환성)

> 실제 명령/파라미터/지원 버전은 문서 버전에 따라 변경될 수 있습니다. 설치 직전 IBM 공식 문서의 최신 요구사항을 최종 기준으로 적용하세요.
