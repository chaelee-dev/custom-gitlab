# custom-gitlab

다른 에이전트의 GitLab 환경 테스트를 위해 Docker로 임시 GitLab CE(Community Edition)를 띄우는 프로젝트입니다.

## 사전 요구사항

- Docker 20.10+ / Docker Compose v2
- 메모리 **최소 4GB**, 권장 6GB 이상 할당 가능한 환경
- 디스크 여유 공간 10GB 이상
- (WSL2 사용 시) `.wslconfig`에서 메모리 4GB 이상 할당 권장

## 빠른 시작

```bash
# 1) 환경 변수 파일 준비
cp .env.example .env
# 필요시 .env 편집 (포트/비밀번호 등)

# 2) 컨테이너 기동 (백그라운드)
docker compose up -d

# 3) 초기화 진행 상태 확인 — healthy 까지 보통 3~5분 소요
docker compose ps
docker compose logs -f gitlab
```

`STATUS`가 `Up (healthy)`이 되면 브라우저에서 접속:

- URL: <http://localhost:48080>
- ID: `root`
- PW: `.env`의 `GITLAB_ROOT_PASSWORD` (기본값 `ChangeMe!12345`)

> 초기 비밀번호가 적용되지 않는 경우 아래 "초기 비밀번호 재설정" 절을 참고하세요.

## 디렉터리 구조

```
.
├── docker-compose.yml   # GitLab CE 서비스 정의
├── .env.example         # 환경 변수 템플릿
├── .env                 # 실제 환경 변수 (커밋 제외)
└── data/                # GitLab 영속 데이터 (커밋 제외)
    ├── config/          # /etc/gitlab     — 설정 파일
    ├── logs/            # /var/log/gitlab — 로그
    └── data/            # /var/opt/gitlab — 저장소/DB
```

## 환경 변수

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `GITLAB_HOSTNAME` | `gitlab.local` | 컨테이너 호스트네임 |
| `GITLAB_EXTERNAL_URL` | `http://localhost:48080` | 외부에서 접근할 URL (포트 포함) |
| `GITLAB_HTTP_PORT` | `48080` | 호스트 HTTP 포트 (충돌 회피용 48xxx 대역) |
| `GITLAB_HTTPS_PORT` | `48443` | 호스트 HTTPS 포트 |
| `GITLAB_SSH_PORT` | `48022` | 호스트 SSH 포트 (호스트 22 충돌 회피용) |
| `GITLAB_ROOT_PASSWORD` | `ChangeMe!12345` | root 초기 비밀번호 (최초 1회) |

## 에이전트 테스트용 사용 가이드

### 1. Personal Access Token 발급

웹 UI:

1. 로그인 → 우측 상단 아바타 → **Edit profile** → **Access tokens**
2. Name, 만료일, Scope 선택 (`api`, `read_repository`, `write_repository` 등)
3. **Create personal access token** → 토큰 즉시 복사 (다시 보이지 않음)

CLI 한 줄로 생성(루트 토큰):

```bash
docker compose exec gitlab gitlab-rails runner "
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
# 버전 확인
curl --header "PRIVATE-TOKEN: <YOUR_TOKEN>" http://localhost:48080/api/v4/version

# 프로젝트 목록
curl --header "PRIVATE-TOKEN: <YOUR_TOKEN>" http://localhost:48080/api/v4/projects
```

### 3. Git clone / push (HTTPS)

```bash
git clone http://root:<YOUR_TOKEN>@localhost:48080/<group>/<project>.git
```

### 4. Git clone / push (SSH)

```bash
# SSH 키 등록 후
git clone ssh://git@localhost:48022/<group>/<project>.git
```

`~/.ssh/config` 예시:

```ssh
Host gitlab.local
    HostName localhost
    Port 48022
    User git
```

## 운영 명령

```bash
# 시작 / 중지 / 재시작
docker compose up -d
docker compose stop
docker compose restart

# 로그 추적
docker compose logs -f gitlab

# 컨테이너 셸 진입
docker compose exec gitlab bash

# GitLab 설정 재적용 (omnibus 옵션 변경 시)
docker compose exec gitlab gitlab-ctl reconfigure

# 서비스 상태 확인
docker compose exec gitlab gitlab-ctl status
```

### 초기 비밀번호 재설정

```bash
docker compose exec gitlab gitlab-rake "gitlab:password:reset[root]"
```

## 데이터 초기화 (완전 삭제)

테스트 후 깨끗하게 비우고 다시 시작하고 싶을 때:

```bash
docker compose down
sudo rm -rf ./data
docker compose up -d
```

> `./data` 디렉터리는 root 권한 파일을 포함하므로 `sudo` 필요.

## 트러블슈팅

- **`STATUS: starting` 상태가 너무 오래 지속됨**
  - 정상이며 최초 기동은 3~10분 소요. `docker compose logs -f gitlab`로 진행 상황 확인.
- **502 Bad Gateway**
  - 아직 초기화 중. healthcheck가 `healthy`가 될 때까지 대기.
- **메모리 부족으로 OOM Kill**
  - Docker Desktop / WSL2 메모리 할당을 4GB 이상으로 늘리기.
- **포트 충돌**
  - `.env`에서 `GITLAB_HTTP_PORT` 등을 사용 중이지 않은 포트로 변경 후 `docker compose up -d`.
- **`external_url`과 실제 접속 포트가 다름**
  - `GITLAB_EXTERNAL_URL`에 반드시 외부 포트를 포함해야 함 (예: `http://localhost:48080`).
- **Git push가 403/HTTPS 에러**
  - 비밀번호 대신 Personal Access Token을 사용. 2FA가 켜져있으면 비밀번호 인증 불가.

## 참고

- GitLab CE Docker 공식 문서: <https://docs.gitlab.com/install/docker/>
- 이 인스턴스는 **테스트 전용**입니다. 운영 환경에서 그대로 사용하지 마세요 (HTTPS 미설정, 약한 비밀번호 등).
