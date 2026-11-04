# ReCu GitOps

> ReCu의 dev/prod 애플리케이션 배포 상태를 Git으로 선언하고, Argo CD를 통해 K3s 클러스터에 반영하는 저장소입니다.

소스·이미지 빌드와 클러스터 배포를 분리합니다. backend CI가 만든 이미지 변경 PR을 검토·병합하면 dev는 자동 동기화하며, prod는 dev에서 검증한 동일 이미지를 승격한 뒤 수동 동기화합니다. frontend·시연용 message-producer와 prod HTTPS Ingress도 함께 관리합니다.

- 프로젝트 상태: dev/prod GitOps 구성 및 배포 완료 보고 반영
- 서비스: [ReCu](https://recu.kro.kr)
- 기본 브랜치: `main`
- 문서: [Argo CD Bootstrap](argocd/install/README.md)
- 문서 기준일: 2026-11-04, 구성 설명은 원격 `main` 기준

## 주요 기능

| 기능 | 개발·운영에서 얻는 결과 |
| --- | --- |
| 공통 base·환경 overlay | 서비스 선언을 재사용하고 Namespace·설정·이미지·노드 배치를 환경별로 분리 |
| 서비스별 이미지 patch | 변경한 백엔드 서비스의 이미지만 PR로 갱신 |
| dev 자동 Sync·prod 수동 Sync | 개발 통합과 운영 승격 절차 구분 |
| Secret 참조 | 비밀값을 저장하지 않고 환경별 사전 생성 Secret 연결 |
| 외부 HTTPS | Traefik Ingress·cert-manager·Let's Encrypt로 prod 프론트 공개 |
| Git 기반 롤백 | 이전 정상 이미지 선언을 PR로 복원하고 동기화 |

## 빠른 시작

### 사전 준비

| 항목 | 필요한 버전·조건 |
| --- | --- |
| 도구 | Git, `kubectl` 또는 Kustomize |
| 클러스터 | K3s 접근 권한, `recu-dev`·`recu-prod` Namespace |
| 노드 | 실제 `environment=dev/prod`, `workload=platform/service/data` 라벨과 자원 확인 |
| 데이터 | 환경별 MariaDB·Kafka·PVC·DB 초기화·토픽 사전 준비 |
| 비밀값 | Config·DB·AI·내부 API·수집·Registry Secret을 각 Namespace에 준비 |
| Argo CD | 설치 선언은 v3.5.7 non-HA, `argocd` Namespace |
| HTTPS | Traefik, cert-manager 설치 선언 v1.21.3, DNS·80/443 접근 준비 |

새 클러스터를 구성할 때의 준비 항목입니다. 기존 환경에서는 먼저 실행 중 리소스·Secret 참조·PVC·Argo CD diff를 확인합니다.

### 설치 및 설정

```bash
git clone -b main https://github.com/LGCNS-MiniProject2-team6/recu-gitops.git
cd recu-gitops
kubectl kustomize overlays/dev-apps
kubectl kustomize overlays/prod-apps
```

렌더링 결과의 Namespace·이미지·Secret 참조·nodeSelector를 확인합니다. `base/`의 초기 이미지보다 환경 overlay의 patch를 적용한 최종 이미지를 기준으로 봅니다.

| 설정 위치 | 내용 |
| --- | --- |
| `platform-runtime` ConfigMap | 프로필·Eureka·Config Git·프론트 Origin |
| `db-runtime` ConfigMap | 서비스별 DB URL·계정 이름 |
| `bus-runtime` ConfigMap | Kafka 주소·Bus 토픽·활성화 |
| `images/<서비스>.json` | 백엔드 Deployment의 최종 이미지 patch |
| `frontend.yaml`, `message-producer.yaml` | 두 워크로드의 Deployment와 이미지 |

ConfigMap은 Kustomize가 생성하고 내용 해시가 이름에 붙습니다. Secret 값은 이 저장소에서 생성·관리하지 않습니다.

| 사전 생성 Secret | 앱에서 사용하는 항목 |
| --- | --- |
| `config-server-secrets` | Config 읽기·Git 인증·`ENCRYPT_KEY`, 선택적 관리 계정 |
| `config-client-secrets` | Gateway·User·AI·Blog의 Config 읽기 인증 |
| `config-metrics-credentials` | Config 전용 Prometheus 인증 |
| `mariadb-secret` | 서비스별 DB 암호, 데이터 계층의 초기화·지표 계정 항목 |
| `ai-secrets` | `GEMINI_API_KEY` |
| `internal-api-secrets` | `INTERNAL_API_TOKEN` |
| `blog-collect-secrets` | Blog와 message-producer가 공유하는 `BLOG_COLLECT_API_KEY` |
| `ghcr-pull` | 비공개 GHCR frontend·message-producer 이미지 pull 인증 |

실제 키 이름은 각 Deployment의 `secretKeyRef`와 데이터 계층 문서를 확인합니다. Secret이 다른 Namespace에 존재해도 재사용되지 않습니다. Config 읽기 값은 Server·Client 간, 내부 토큰은 같은 환경의 업무 서비스 간 일치해야 합니다.

### 실행

일반 이미지 배포는 PR·Argo CD 동기화로 진행합니다. 초기 Argo CD 구성은 클러스터 담당자가 아래 순서로 적용합니다.

```bash
kubectl apply -k argocd/install
kubectl -n argocd rollout status deployment/argocd-server --timeout=300s
kubectl apply -k argocd/projects
kubectl apply -k argocd/applications
```

AppProject·Application 적용 전 Argo CD CRD와 컴포넌트 준비를 확인합니다. 비공개 Git 읽기 인증은 Argo CD에 별도로 등록합니다. backend Actions의 `GITOPS_TOKEN`과 공유하지 않습니다.

UI 접속 예시:

```bash
kubectl -n argocd port-forward svc/argocd-server 18080:443
```

로컬 `https://localhost:18080`으로 접속합니다. 원격 서버에서 port-forward를 실행했다면 SSH 터널이 추가로 필요합니다. Argo CD UI는 외부 Ingress로 공개하지 않습니다. `Ctrl+C`는 port-forward만 종료하고 클러스터 앱을 중지하지 않습니다.

## 사용 예시

### dev 이미지 반영

1. backend `develop` 병합 후 CI가 이미지와 digest·소스 SHA·실행 링크를 포함한 GitOps PR을 생성합니다.
2. 대상 서비스·이전 이미지·호환 API 계약·렌더링 결과를 검토합니다.
3. PR을 `main`에 병합하면 `recu-dev-apps`가 자동 Sync합니다.
4. Argo CD의 Sync revision·Health, 실제 Deployment 이미지·rollout, 로그인·업무 기능을 확인합니다.

### prod 승격

1. dev 검증을 통과한 동일 tag/digest를 `overlays/prod-apps`에 반영하는 PR을 작성합니다.
2. prod 프로필·DB·Secret·Origin·노드 배치는 유지하고 호환 이미지 조합을 검토합니다.
3. 검토·병합 후 담당자가 `recu-prod-apps`를 수동 Sync합니다.
4. HTTPS·로그인·기업 조회 및 변경한 기능을 확인합니다.

frontend 이미지 CI는 현재 GHCR 업로드까지만 수행하므로, frontend 이미지 선언 변경 PR은 별도로 작성합니다. 기존 서비스의 이미지 PR을 뒤늦게 병합해 구버전으로 되돌리지 않도록 미병합 PR의 소스·태그를 확인합니다.

## 기술 구성

| 영역 | 기술·구성 | 사용하는 목적 |
| --- | --- | --- |
| 클러스터 | K3s, Namespace, nodeSelector | dev/prod 및 워크로드 역할별 배치 |
| 배포 선언 | Kustomize base·overlay·JSON patch | 환경별 설정·이미지 조합 |
| 동기화 | Argo CD Application·AppProject | Git 선언 반영·상태·허용 리소스 관리 |
| 이미지 | Docker Hub, GHCR | 백엔드·프론트·시연 수집기 이미지 |
| 공개 경로 | Traefik, cert-manager, Let's Encrypt | 도메인·Ingress·TLS |

```text
GitHub Actions → 이미지 Registry → GitOps PR → main
  ├─ recu-dev-apps  → 자동 Sync → recu-dev
  └─ recu-prod-apps → 수동 Sync → recu-prod

Internet → recu.kro.kr → Traefik HTTPS Ingress
  → frontend nginx → API Gateway → User / AI / Blog
```

| 경로 | 책임 |
| --- | --- |
| `base/platform/` | Discovery·Config·Gateway의 Deployment·Service |
| `base/services/` | User·AI·Blog의 Deployment·Service |
| `overlays/dev-apps/` | dev 이미지·ConfigMap·프론트·시연 수집기 |
| `overlays/prod-apps/` | prod 이미지·ConfigMap·프론트·시연 수집기·Ingress |
| `argocd/install/` | non-HA Argo CD 설치·prod platform 노드 배치 |
| `argocd/projects/`, `argocd/applications/` | 허용 범위·브랜치·경로·동기화 정책 |
| `cert-manager/install/`, `cert-manager/issuers/` | 인증서 컨트롤러 설치·발급자 선언 |

앱 동기화 대상은 백엔드 6종과 frontend·message-producer입니다. 현재 AppProject는 두 앱 Namespace에서 ConfigMap·Service·Deployment·Ingress를 허용합니다. MariaDB·Kafka·모니터링·PVC·Secret 실값은 앱 Application 범위 밖이며, [backend 인프라 구성](https://github.com/LGCNS-MiniProject2-team6/backend/tree/develop/deploy/k8s)에서 별도로 준비합니다.

message-producer는 실제 수집기를 대신하는 시연 서버로 Kafka에 직접 연결하지 않고 Gateway 수집 API를 호출합니다. 인증 없는 관리 화면은 Pod 내부 loopback에만 열고 공개 Service·Ingress를 두지 않습니다.

## 테스트 및 빌드

이 저장소는 애플리케이션을 빌드하지 않습니다. 변경 검증은 렌더링과 배포 후 확인을 나눠 수행합니다.

```bash
kubectl kustomize overlays/dev-apps
kubectl kustomize overlays/prod-apps
kubectl -n argocd get applications
kubectl -n recu-dev get deployments,pods,services
kubectl -n recu-dev rollout status deployment/blog-service --timeout=300s
kubectl -n recu-dev get deployment blog-service -o jsonpath='{.spec.template.spec.containers[0].image}'
```

앞의 두 명령은 로컬 렌더링이며 클러스터에 적용하지 않습니다. 이후 명령은 클러스터 접근 권한이 필요합니다. prod 확인 시 Namespace를 `recu-prod`로 바꿉니다.

성공 기준은 기대한 이미지·환경 참조, Argo CD `Synced / Healthy`, 정상 rollout·readiness, 실제 업무 기능과 지표입니다. 렌더링 성공·이미지 업로드 성공만으로 배포 완료로 판단하지 않습니다.

## 배포 및 운영

### 동기화·승인 정책

| 항목 | dev | prod |
| --- | --- | --- |
| Application | `recu-dev-apps` | `recu-prod-apps` |
| Namespace | `recu-dev` | `recu-prod` |
| Git 경로 | `overlays/dev-apps` | `overlays/prod-apps` |
| 동기화 | 자동 | 수동 |
| 자동 prune·self-heal | 모두 비활성 | 자동 Sync 없음, 수동 Sync도 prune 없이 진행하는 운영 방침 |
| 이미지 선택 | 변경 서비스 CI 산출물 | dev에서 검증한 동일 이미지 |

- `main` 직접 push 대신 PR을 사용하고 자동 PR 병합은 하지 않습니다.
- 운영 방침은 dev 작성자 외 검토, prod 인프라·릴리스 담당자의 추가 검토 후 승격입니다.
- `.github/CODEOWNERS`는 dev overlay 검토 담당으로 `@Unpart`를 지정합니다. CODEOWNERS 지정 자체와 GitHub의 승인 강제 설정은 별개입니다.
- 2026-11-04 GitHub 조회에서 `prod-required-reviewers` ruleset은 `disabled`였습니다. prod 2인 검토 방침이 해당 규칙으로 강제되는 상태는 아니므로 병합 전 검토 기록을 확인하고, 강제 적용이 필요하면 저장소 관리자가 규칙을 활성화해야 합니다.

### HTTPS·데이터·관측

prod Ingress는 `recu.kro.kr`, `traefik`, ClusterIssuer `letsencrypt-prod`, TLS Secret `recu-kro-kr-tls`를 참조합니다. DNS·외부 네트워크·cert-manager·발급자는 앱 Sync 전에 준비합니다. Gateway의 prod Origin도 `https://recu.kro.kr`로 맞춥니다.

배포 보고에는 dev/prod 데이터 계층 분리, prod Kafka/MariaDB overlay 분리, Secret 준비, Argo CD prod `Synced / Healthy`, HTTPS·Google 로그인·기업 목록/상세·게시글 조회 성공이 기록되어 있습니다. 이번 README 작성에서 클러스터·인증서·실제 사용자 흐름을 다시 실행해 확인하지는 않았습니다.

애플리케이션 지표·DB·Kafka 관측 구성은 [모니터링 안내](https://github.com/LGCNS-MiniProject2-team6/backend/tree/develop/monitoring)를 참고합니다. dev 관측 구성과 prod 관측 적용 상태는 구분합니다.

### 롤백

1. 직전 기능 검증 기록의 정상 이미지 조합을 선택합니다.
2. 해당 환경 이미지 선언을 되돌리는 PR을 검토·병합합니다.
3. dev는 자동 Sync, prod는 담당자가 수동 Sync합니다.
4. 실제 이미지·rollout·기능·지표를 확인합니다.

긴급 수동 조치 후에도 Git 선언을 정상 상태와 일치시킵니다. 앱 이미지 롤백은 DB 스키마·데이터를 복구하지 않습니다. 해시가 붙은 이전 ConfigMap은 Deployment 참조 여부를 확인한 뒤 정리합니다. EC2 재시작 뒤 임시 SSH 터널·port-forward·Quick Tunnel은 다시 연결해야 합니다.

## 문제 해결

| 증상 | 확인할 내용 | 조치 |
| --- | --- | --- |
| `CreateContainerConfigError` | `describe pod`의 Secret·key·UID 오류 | Namespace별 참조 준비, 이름 기반 USER 검증 실패 시 이미지에 맞는 숫자 UID 확인 |
| `ImagePullBackOff` | Registry 권한·이미지 tag·`ghcr-pull` | 실제 이미지 존재·접근 권한과 Namespace별 pull Secret 확인 |
| Pod `Pending` | nodeSelector와 실제 노드 라벨·여유 자원 | environment·workload 라벨 및 자원 요청을 맞춤 |
| 로그인 후 기업 조회 503 | User·Blog 이미지 계약·내부 토큰 | dev에서 검증한 호환 이미지 조합과 토큰 주입 확인 |
| `OOMKilled`·반복 재시작 | 앱 로그·메모리·limit | 측정 후 리소스 조정. 현재 Blog 선언 limit은 768Mi |
| prod 병합 후 변화 없음 | Application revision·Sync 정책 | 승인·병합 확인 후 prod 수동 Sync |
| Config target 수집 실패 | `config-metrics-credentials`·Prometheus 마운트 | 전용 지표 인증과 수집 설정을 함께 확인 |
| 인증서 미발급 | DNS·80/443·Issuer·Certificate 상태 | Traefik·cert-manager와 HTTP 검증 경로 확인 |

## 문서 및 협업

| 문서·저장소 | 내용 |
| --- | --- |
| [Backend](https://github.com/LGCNS-MiniProject2-team6/backend) | 서비스·이미지 CI·데이터·관측 |
| [Frontend](https://github.com/LGCNS-MiniProject2-team6/frontend) | 화면·정적 이미지·GHCR CI |
| [Config](https://github.com/LGCNS-MiniProject2-team6/recu-config) | 중앙 설정·Bus·라우트·CORS |
| [Bootstrap](argocd/install/README.md) | Argo CD 설치 순서. 현재 허용 범위는 AppProject 선언 참조 |
| [dev Application](argocd/applications/recu-dev-apps.yaml) · [prod Application](argocd/applications/recu-prod-apps.yaml) | 경로·동기화 정책 |
| [AppProject](argocd/projects/recu-project.yaml) | 저장소·Namespace·리소스 허용 범위 |
| [이미지 자동화 안내](https://github.com/LGCNS-MiniProject2-team6/backend/blob/develop/docs/GITOPS_DEV_IMAGE_AUTOMATION.md) | 서비스별 patch와 자동 PR 계약 |

PR에는 대상 환경·서비스, tag/digest·소스 SHA·CI 링크, 검증 결과·복구 방법을 남깁니다. 비밀번호·토큰·kubeconfig·개인키를 첨부하지 않습니다. 운영 문의는 역할2, 이미지 자동화는 역할4, 데이터·관측은 역할3과 조율합니다.

## 팀 및 문의

| 작업 영역 | 담당자 | 담당 내용 |
| --- | --- | --- |
| DevOps 역할2 | [@Unpart](https://github.com/Unpart) | K3s·노드·Namespace·Secret·앱 배포, 앱 전용 overlay, Argo CD 설치·AppProject·Application·동기화, 외부 도메인·Ingress·TLS |
| DevOps 역할4 | [@junwoo-qkr](https://github.com/junwoo-qkr) | backend 이미지 CI·서비스별 GitOps PR, 이미지·digest·소스 추적·승격·롤백 PR 지원 |
| DevOps 역할3 | [@HYH0804](https://github.com/HYH0804) | MariaDB·Kafka·PVC·토픽·관측 구성, dev frontend·시연 수집기 배포·UID·메모리 후속 조정 |
| 플랫폼·Gateway 연동 | [@yhi9839](https://github.com/yhi9839) · [@pdpdpdjjj23](https://github.com/pdpdpdjjj23) | 각각 Config·Eureka·Gateway 라우팅과 Gateway JWT·사용자 ID 전달 계약 |
| 업무 서비스 연동 | [@woomoo3](https://github.com/woomoo3) · [@pdpdpdjjj23](https://github.com/pdpdpdjjj23) · [@Kimoojin](https://github.com/Kimoojin) | 각각 User·AI·Blog 내부 API 호환성·배포 후 기능 확인 |
| Frontend 화면·이미지 연동 | [@yhi9839](https://github.com/yhi9839) · [@HYH0804](https://github.com/HYH0804) | 각각 브라우저·OAuth 콜백·화면 연동과 nginx·정적 이미지·GHCR CI |

담당 계정은 recu-gitops와 연동 저장소의 병합 PR·커밋 이력 기준입니다. 배포 선언 직접 변경과 서비스·화면 연동 역할을 구분합니다.

## 향후 계획

- [ ] prod 추가 승인 규칙의 활성화 여부·실제 승인 강제 동작 확인
- [ ] dev/prod Git 기반 롤백·데이터 복구 절차의 실행 기록 보강
- [ ] 이미지·설정·기능 검증 기록을 연결한 배포 이력 유지

## 라이선스

저장소에 라이선스 파일이 확인되지 않았습니다. 사용·재배포 조건은 팀에 문의하고 공개 배포 전에 확정합니다.
