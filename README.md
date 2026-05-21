# custom-gitlab

다른 에이전트의 GitLab 환경 테스트를 위해 사내 K8s 개발 클러스터에 임시 GitLab CE(Community Edition) 인스턴스를 띄우는 매니페스트입니다.

> 운영 GitLab은 공식 Helm Chart 사용을 권장합니다. 이 매니페스트는 Omnibus 단일 Pod 기반의 "테스트용 단일 인스턴스" 컨셉이며, 고객 EKS의 Helm 기반 배포와는 별개입니다.

## 사전 요구사항

- 사내 K8s 클러스터 접근 권한 (`kubectl` 컨텍스트 설정 완료)
- `kustomize`가 통합된 `kubectl` 1.21+
- Ingress controller (예: `ingress-nginx`)
- 동적 프로비저닝 가능한 기본 `StorageClass`
- 워커 노드 가용 메모리 6GB 이상

## 빠른 시작

**1) 환경에 맞게 매니페스트 값 수정** — 자세한 항목은 아래 [환경별 조정](#환경별-조정) 절 참고. 최소한 다음 두 곳은 반드시 맞춰야 합니다.

- `k8s/configmap.yaml`의 `GITLAB_EXTERNAL_URL` — 사용자 접근 URL (예: `http://gitlab.devserver.example.local`)
- `k8s/ingress.yaml`의 `spec.rules[0].host` — 위 URL의 호스트와 **동일해야 함** (불일치 시 redirect / asset URL이 깨짐)

**2) 배포**

```bash
# 네임스페이스 생성
kubectl apply -f k8s/namespace.yaml

# root 초기 비밀번호 Secret 생성 (매니페스트에 비밀번호 박지 않음)
kubectl -n custom-gitlab create secret generic gitlab-secret \
  --from-literal=GITLAB_ROOT_PASSWORD='<12자 이상 + 특수문자>'

# 나머지 리소스 배포
kubectl apply -k k8s/

# 기동 확인 — Ready 1/1까지 보통 5~10분 소요
kubectl -n custom-gitlab get pod,svc,ingress,pvc
kubectl -n custom-gitlab logs -f statefulset/gitlab
```

Pod이 `Ready`가 되면 `GITLAB_EXTERNAL_URL`(기본 `http://gitlab.local`)로 접속:

- ID: `root`
- PW: Secret에 넣은 `GITLAB_ROOT_PASSWORD`

## 환경별 조정

배포 전에 사내 클러스터 상황에 맞춰 아래 값을 수정합니다.

| 위치 | 키 | 설명 |
|---|---|---|
| `k8s/kustomization.yaml` | `images[0].newTag` | GitLab CE 이미지 태그. `latest` 금지 — 고객 EKS Helm appVersion과 동일 태그로 핀 (예: `18.11.3-ce.0`) |
| `k8s/configmap.yaml` | `GITLAB_EXTERNAL_URL` | 사용자 접근 URL. Ingress 호스트와 정합 — 80/443 외 포트면 URL에 포함 |
| `k8s/ingress.yaml` | `spec.rules[0].host` | 클러스터에서 라우팅할 호스트 (`GITLAB_EXTERNAL_URL`과 정합) |
| `k8s/ingress.yaml` | `spec.ingressClassName` | 사내 Ingress controller (`nginx` / `traefik` / `alb` 등) |
| `k8s/ingress.yaml` | `spec.tls` (선택) | TLS 종단을 Ingress에서 처리할 경우. cert-manager 또는 미리 생성한 Secret 참조 |

> Git clone/push는 HTTPS + Personal Access Token으로 수행합니다. SSH는 의도적으로 노출하지 않습니다.

## 디렉터리 구조

```
.
└── k8s/
    ├── namespace.yaml       # custom-gitlab 네임스페이스
    ├── configmap.yaml       # GITLAB_OMNIBUS_CONFIG 및 환경값
    ├── service.yaml         # headless + ClusterIP(HTTP)
    ├── statefulset.yaml     # Pod spec + PVC(config / logs / data)
    ├── ingress.yaml         # 호스트 라우팅
    └── kustomization.yaml   # 이미지 태그 핀 + 리소스 묶음
```

## 에이전트 테스트용 사용 가이드

### 1. Personal Access Token 발급

웹 UI:

1. 로그인 → 우측 상단 아바타 → **Edit profile** → **Access tokens**
2. Name, 만료일, Scope 선택 (`api`, `read_repository`, `write_repository` 등)
3. **Create personal access token** → 토큰 즉시 복사 (다시 보이지 않음)

CLI 한 줄로 생성(루트 토큰):

```bash
kubectl -n custom-gitlab exec -it statefulset/gitlab -- gitlab-rails runner "
token = User.find_by_username('root').personal_access_tokens.create(
  scopes: ['api','read_repository','write_repository'],
  name: 'agent-test',
  expires_at: 30.days.from_now
)
token.set_token('glpat-agent-test-token-1234567890')
token.save!
puts token.token
"
```

### 2. API 동작 확인

```bash
curl --header "PRIVATE-TOKEN: <YOUR_TOKEN>" http://gitlab.local/api/v4/version
curl --header "PRIVATE-TOKEN: <YOUR_TOKEN>" http://gitlab.local/api/v4/projects
```

### 3. Git clone / push

```bash
git clone http://root:<YOUR_TOKEN>@gitlab.local/<group>/<project>.git
```

## 운영 명령

### 매니페스트 변경 반영 (재배포)

ConfigMap / Ingress / Kustomization 등 `k8s/` 매니페스트를 수정한 뒤에는 `apply` 만으로는 부족합니다 — `envFrom: configMapRef`로 주입한 환경변수는 Pod 시작 시점에 컨테이너에 박혀, ConfigMap이 갱신돼도 실행 중인 Pod에는 반영되지 않습니다. 또한 GitLab은 entrypoint에서 `GITLAB_OMNIBUS_CONFIG` → `/etc/gitlab/gitlab.rb` → `gitlab-ctl reconfigure` 순으로 적용하므로 Pod 재시작이 필요합니다.

```bash
kubectl apply -k k8s/
kubectl -n custom-gitlab rollout restart statefulset/gitlab
kubectl -n custom-gitlab rollout status statefulset/gitlab
```

새 Pod이 `Ready`가 될 때까지 보통 3~10분 소요됩니다. 진행 상황은 `kubectl -n custom-gitlab logs -f statefulset/gitlab` 로 확인.

### 그 외 명령

```bash
# 컨테이너 셸 진입
kubectl -n custom-gitlab exec -it statefulset/gitlab -- bash

# 비밀번호 재설정
kubectl -n custom-gitlab exec -it statefulset/gitlab -- \
  gitlab-rake "gitlab:password:reset[root]"

# GitLab 서비스 상태 확인
kubectl -n custom-gitlab exec -it statefulset/gitlab -- gitlab-ctl status
```

## 전체 삭제 (teardown)

**Pod만 잠시 내리기** — 데이터(PVC) 유지, 다시 띄우면 그대로 복귀.

```bash
kubectl -n custom-gitlab scale statefulset/gitlab --replicas=0
# 다시 띄우기
kubectl -n custom-gitlab scale statefulset/gitlab --replicas=1
```

**완전 삭제** — Pod / PVC(데이터) / Secret / Ingress 등 Namespace 안의 모든 리소스 제거.

```bash
kubectl delete namespace custom-gitlab
```

> Namespace 삭제는 안에 있는 PVC도 같이 제거하므로 영속 데이터가 모두 사라집니다. 데이터를 보존해야 하면 위의 일시 정지를 사용하세요.

## 트러블슈팅

- **Pod이 `Running`인데 `Ready`가 안 됨**
  - 정상이며 최초 기동은 3~10분 소요. `kubectl -n custom-gitlab logs -f statefulset/gitlab`로 진행 상황 확인.
- **502 / 503**
  - 아직 초기화 중. readiness probe가 통과할 때까지 대기.
- **PVC `Pending`**
  - 기본 `StorageClass`가 없거나 동적 프로비저닝이 안 되는 환경. `statefulset.yaml`의 `volumeClaimTemplates`에 `storageClassName` 명시.
- **`OOMKilled`**
  - StatefulSet의 `resources.limits.memory`를 노드 가용 메모리에 맞춰 상향.
- **`external_url`과 실제 접속 호스트가 다름**
  - `configmap.yaml`의 `GITLAB_EXTERNAL_URL`과 `ingress.yaml`의 `host`가 동일해야 함 (포트 포함).
- **Git push가 403/인증 에러**
  - 비밀번호 대신 Personal Access Token을 사용. 2FA가 켜져있으면 비밀번호 인증 불가.

## 참고

- GitLab CE 공식 문서: <https://docs.gitlab.com/>
- 이 인스턴스는 **테스트 전용**입니다. 운영 환경에서 그대로 사용하지 마세요 (TLS 미기본 설정, 단일 Pod, 약한 비밀번호 등).
