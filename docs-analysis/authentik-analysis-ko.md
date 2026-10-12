# authentik 전수조사 분석 및 수익화 전략 정리

> 이 문서는 `bmshin94/authentik` 저장소를 전수조사한 결과와, 이를 바탕으로 한
> 활용·수익화 전략을 정리한 기록입니다.

| 항목 | 내용 |
| --- | --- |
| 내 저장소 | <https://github.com/bmshin94/authentik> |
| 원본(업스트림) | <https://github.com/goauthentik/authentik> |
| 공식 홈페이지 | <https://goauthentik.io> |
| 공식 문서 | <https://docs.goauthentik.io> |
| 연동 가이드 | <https://integrations.goauthentik.io> |
| API 레퍼런스 | <https://api.goauthentik.io> |
| Helm 차트 | <https://github.com/goauthentik/helm> |
| 분석 대상 버전 | `2026.11.0-rc1` |
| 작성일 | 2026-10-12 |
| 작업 브랜치 | `claude/kind-davinci-bdnuj3` |

---

## 1. 결론 요약

이 저장소는 **오픈소스 IdP(Identity Provider) / SSO 서버 제품의 전체 소스코드**입니다.
Okta, Auth0, Microsoft Entra ID, Ping Identity를 대체하는 자체 호스팅 인증 서버입니다.

- 플러그인도, Claude 스킬도, MCP 서버도 **아닙니다.** 독립 실행되는 서버 애플리케이션입니다.
- 루트의 `CLAUDE.md` / `AGENTS.md`는 **이 프로젝트를 개발할 때 AI 에이전트에게 주는 가이드 문서**입니다.
- 내 포크는 업스트림과 사실상 동일하며, `CLAUDE.md`(21줄) 하나만 추가된 상태입니다 (`Merge PR #1`).

### 라이선스 구조

| 영역 | 라이선스 |
| --- | --- |
| 루트 전반 | MIT |
| `authentik/enterprise/` | authentik EE 상용 라이선스 (운영 사용 시 구독 필요) |
| `website/` | CC BY-SA 4.0 (출처 표기 + 동일 라이선스) |
| 클라이언트 측 JavaScript | MIT Expat |

---

## 2. 규모 (전수조사 수치)

```
추적 파일          6,527개
저장소 크기        168MB

Python            200,192줄  (2,261 파일)   서버 본체
TypeScript        330,593줄  (2,086 파일)   웹 UI 3개 앱
MDX (문서)        155,784줄  (755 파일)     문서 사이트 3개
Go                 49,113줄  (231 파일)     아웃포스트
Rust               22,455줄  (211 파일)     신형 서버/워커/프록시
YAML               66,729줄                블루프린트 + CI

REST API 엔드포인트    615개  (schema.yml, 1.8MB, 자동 생성)
DB 마이그레이션        647개
파이썬 테스트 파일     323개
문서 페이지            494개
서드파티 연동 가이드   246개  (19개 카테고리)
블루프린트 YAML         45개
인증 스테이지           23종
```

---

## 3. 폴더 구조와 실제 내용

```
authentik/          Django 5.2 + DRF 앱 25개 (심장부)
  core/             사용자·애플리케이션·토큰·Actor 모델
  flows/            플로우 엔진 (planner.py, stage.py)
  stages/           23종: password, prompt, consent, invitation, captcha, deny,
                    authenticator_totp/webauthn/duo/sms/email/static,
                    user_login/logout/write/delete, redirect, email, identification
  providers/        내보내는 프로토콜: oauth2(107파일), saml(62파일),
                    ldap, proxy, rac, radius, scim
  sources/          받아오는 신원: ldap, oauth, saml, scim, kerberos, plex, telegram
  policies/         expression, geoip, reputation, password, expiry, event_matcher
  enterprise/       EE 전용 (별도 라이선스)
      agents/       AI/머신 에이전트 신원  <-- 핵심 발견
      requests/     PAM: 권한 신청·승인·시간제한 부여
      providers/    google_workspace, microsoft_entra, ssf, ws_federation
      stages/       mtls, account_lockdown, source
      audit/ reports/ endpoints/ policies/
  endpoints/        디바이스 에이전트, PSSO, 컴플라이언스 커넥터
  rbac/ crypto/ events/ brands/ tenants/ outposts/ blueprints/ tasks/

web/                TypeScript, Lit 3 + PatternFly 4
  src/admin/        관리자 콘솔 (agents/ 포함)
  src/user/         사용자 포털 (agents/, requests/ 포함)
  src/flow/         로그인 화면 렌더러 (FlowExecutor.ts)
  src/rac/          원격 접속 클라이언트

website/            Docusaurus 3, 사이트 3개
  docs/             docs.goauthentik.io (494 페이지)
  integrations/     integrations.goauthentik.io (246개 서비스)
  api/              api.goauthentik.io (schema.yml에서 자동 생성)

cmd/ + internal/    Go 1.26 아웃포스트 바이너리 (ldap, rac, radius)
src/ + packages/ak-*  Rust(axum) 서버/워커/프록시 + 공용 크레이트
packages/client-{go,rust,ts}   자동 생성 API 클라이언트 (손으로 수정 금지)
packages/django-{dramatiq,channels,cache}-postgres   Postgres 기반 인프라
blueprints/         선언형 YAML (기동 시 자동 적용, 45개)
lifecycle/          부팅·마이그레이션·gunicorn·ak CLI·컨테이너 엔트리포인트
tests/              e2e, integration, geoip, openid_conformance
Makefile            모든 명령의 허브 (make help)
```

---

## 4. 핵심 설계 — 5개 개념

| 개념 | 역할 | 비유 |
| --- | --- | --- |
| **Flow** | 로그인/가입/복구 절차의 시나리오 | 출입 심사 매뉴얼 |
| **Stage** | 시나리오의 한 단계 (23종) | 매뉴얼의 한 장면 |
| **Provider** | authentik이 앱에게 말을 거는 방식 (7종) | 경비실 통신 규격 |
| **Source** | 외부에서 신원을 끌어오는 통로 (7종) | 외부 신분증 인정 |
| **Policy** | 통과시킬지 판정 (IP/평판/표현식) | 추가 규칙 |

**Outpost**: LDAP·RADIUS·프록시처럼 상시 대기해야 하는 역할을 Go/Rust 별도 바이너리로 분리.
중앙 서버가 원격 관리합니다.

**Proxy Provider**: 로그인 기능이 전혀 없는 앱 앞에 세워, 앱을 **수정하지 않고** 인증을 붙입니다.
홈랩/레거시 환경의 킬러 기능입니다.

### 인프라 특징 (직접 확인)

`lifecycle/container/compose.yml`의 서비스는 **`postgresql`, `server`, `worker` 3개뿐 — Redis가 없습니다.**
큐(Dramatiq), 캐시, 웹소켓 채널을 모두 Postgres로 처리하도록 자체 패키지를 만들어 넣었습니다
(`packages/django-dramatiq-postgres`, `django-postgres-cache`, `django-channels-postgres`).
→ 운영 부담이 크게 줄어든 구조이며, 소규모 셀프호스팅에 결정적으로 유리합니다.

### 4개 언어를 쓰는 이유

| 언어 | 담당 | 선택 이유 |
| --- | --- | --- |
| Python 3.14 / Django 5.2 | 정책·플로우·DB·API | 로직 복잡도 높고 변경 잦음 |
| TypeScript / Lit 3 | 관리 콘솔·로그인 화면 | 웹 UI |
| Go 1.26 | LDAP/RADIUS/RAC 아웃포스트 | 단일 바이너리, 네트워크 서버 |
| Rust (2024, axum) | 고성능 프록시·워커 | 속도 + 메모리 안전성 |

이 네 조각을 묶는 접착제가 **`schema.yml`(615 API)** 입니다.
Python에서 API를 바꾸면 `make gen` → Go/Rust/TS 클라이언트가 자동 재생성됩니다.

---

## 5. 핵심 발견: `enterprise/agents` — AI 에이전트 신원 체계

`authentik/enterprise/agents/models.py`의 `Agent` 모델은 AI/자동화 시대를 위한 머신 신원입니다.

```
class Agent(Actor):
    # 에이전트는 로그인 유저가 아니라 머신 신원이다.
    # 서비스 어카운트로 표시하고 비밀번호 인증을 끈다.
    # 발급된 API 토큰으로 인증한다. 기본값 MIRROR이므로
    # 자신이 대행하는 유저의 권한을 절대 초과할 수 없다.
```

`Agent`는 `Actor`를 상속하고, `Actor`는 `parent` 유저와 `policy_behavior`를 가집니다.

| 권한 상속 모드 | 동작 | 용도 |
| --- | --- | --- |
| `MIRROR` (기본) | 주인의 **현재** 권한을 실시간 반영 | 권한 변동 자동 추적 |
| `COPY` | 생성 시점 권한 스냅샷 | 고정 권한 묶음 |
| `NONE` | 아무것도 상속하지 않음 | 최소 권한, 셀프서비스 강제 기본값 |

`policy_behavior`는 **생성 후 변경 불가**입니다 (`save()`에서 `ValidationError`).
권한 승격 공격 경로를 구조적으로 차단합니다.

### Human-in-the-loop 승인 플로우

`enterprise/requests/api/grant_requests.py` (코드 주석 요지):

> 에이전트에겐 브라우저가 없으므로 승인 절차를 스스로 실행할 수 없다 —
> 그래서 자기가 대행하는 사람에게 건네줄 URL을 반환한다.
> 에이전트의 요청은 그 주인만이 결정한다.
> 에이전트는 주인이 이미 가진 권한만 요청할 수 있다.

```
AI: POST /api/v3/requests/grant-requests/agent/   "DB 읽기 권한 필요"
 -> 응답: { fulfill_url: "https://..." }
 -> 사람이 URL 클릭 -> 승인 -> 시간제한 권한 부여
 -> 만료 시 자동 회수
 -> 모든 행동은 "주인 유저"로 감사 기록 (tests/test_attribution.py)
```

### 용어 혼동 주의

- `enterprise/agents` = **AI/자동화 신원** (Agent 계정)
- `endpoints/.../authentik-agent` = **디바이스 에이전트** (Windows/Mac/Linux 설치형)

완전히 다른 것이며, 공식 문서도 이 혼동을 경고합니다.

---

## 6. OSS vs Enterprise 경계

| 무료 (MIT) | 유료 (EE) |
| --- | --- |
| SAML, OAuth2/OIDC, LDAP, RADIUS, SCIM, Proxy, RAC | **Agent 계정**, PAM(권한 신청/승인) |
| 23개 스테이지, 2FA (TOTP/WebAuthn/SMS/Email/Duo) | Google Workspace / Entra ID 프로비저닝 |
| 정책 엔진, 플로우 엔진, 블루프린트 | mTLS·스마트카드, RADIUS EAP-TLS, WS-Federation |
| 멀티테넌시, 브랜딩, 감사 이벤트 | 강화 감사로그, CSV 내보내기, 계정 잠금, SSF |
| 246개 연동 가이드 | 디바이스 컴플라이언스, 로컬 디바이스 로그인 |

핵심 SSO는 전부 무료입니다. 유료는 대기업 거버넌스/컴플라이언스 영역입니다.

---

## 7. 언제 쓰는가

| 상황 | 쓰는 이유 |
| --- | --- |
| 사내 도구 10개+가 각각 로그인 | SSO 통합, 입·퇴사 계정 일괄 처리 |
| Okta/Auth0 비용 폭증 | MAU 과금 없이 자체 호스팅 |
| 2FA/MFA 전사 의무화 | 스테이지 추가만으로 즉시 적용 |
| 홈랩/NAS/Proxmox/*arr 보호 | Proxy Provider로 로그인 없는 앱에 인증 부착 |
| 레거시가 LDAP/RADIUS만 지원 | 아웃포스트로 프로토콜 변환 |
| 고객사별 로그인 브랜딩 (B2B SaaS) | `brands/` + `tenants/` 멀티테넌시 |
| AI 에이전트가 사내 API를 호출 | Agent 계정 + 승인 플로우 + 감사 추적 |

---

## 8. 설치 및 사용법

### 8-1. Docker Compose (권장, 5~10분)

```bash
mkdir authentik && cd authentik

wget -O compose.yml https://goauthentik.io/docker-compose.yml

echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')" >> .env
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" >> .env

docker compose pull
docker compose up -d
```

- 초기 설정: <http://localhost:9000/if/flow/initial-setup/>
- 관리자 콘솔: `/if/admin/`  ·  사용자 포털: `/if/user/`
- 필요 사양: CPU 2코어 / RAM 2GB 이상. 컨테이너 3개 (Redis 불필요)

### 8-2. 다른 설치 방법

| 방법 | 용도 |
| --- | --- |
| Kubernetes (Helm) | 운영/대규모 — `helm repo add authentik https://charts.goauthentik.io` |
| AWS CloudFormation | AWS 공식 템플릿 |
| DigitalOcean Marketplace | 원클릭 배포 |

### 8-3. 첫 연동 실습 순서

1. Applications → Providers → Create → **OAuth2/OpenID Provider**
2. Applications → Create → 위 Provider 연결, slug 지정
3. Client ID / Secret / authorize·token·userinfo URL 복사
4. 대상 앱(Grafana, GitLab, Nextcloud 등)의 OIDC 설정에 붙여넣기
5. `website/integrations/`에 246개 서비스별 복붙 가이드가 이미 존재

### 8-4. 소스 개발 (이 저장소 기준)

```bash
make install           # Node + web + Python(uv) 전부 설치
make gen-dev-config    # 로컬 개발 설정 생성
make dev-reset         # DB 드롭 -> 재생성 -> 마이그레이션
make run               # 서버+워커 (uv run ak allinone)
make run-watch         # 자동 리로드 (watchexec 필요)

make test              # 파이썬 테스트 (경로 지정: make test authentik/providers/saml)
make web-test / go-test / rust-test
make lint-fix && make lint
make gen               # API 변경 시 schema.yml + 클라이언트 재생성 (필수)
make help              # 전체 명령 목록
```

**필수 규칙**

- 시리얼라이저/뷰셋 수정 → 반드시 `make gen` + 마이그레이션 커밋
- `schema.yml`, `packages/client-*` 는 손으로 수정 금지
- 웹에서 API 호출은 `@goauthentik/api`만 사용 (fetch/axios 금지)
- 제품명은 항상 소문자 `authentik`
- 커밋에 Claude co-author 트레일러를 넣지 않음 (업스트림 기여 시)

**요구 버전**: Python 3.14, Go 1.26, Node >= 24, pnpm >= 12.4.
이 저장소는 `2026.11.0-rc1` 개발 버전이므로 운영에는 안정 태그를 사용하세요.

---

## 9. 분류 — 플러그인 / 스킬 / MCP?

| 분류 | 해당 | 근거 |
| --- | --- | --- |
| 플러그인 | 아니오 | 자체 DB·웹서버·워커를 가진 독립 시스템 |
| Claude 스킬 | 아니오 | `.claude/skills/`, `SKILL.md` 없음 |
| MCP 서버 | 아니오 | 저장소 전체 `grep -ri "mcp"` 결과 **0건** |
| **독립 서버 애플리케이션** | **예** | Django+DRF + Postgres + 워커 + Go/Rust 아웃포스트 |

정확한 분류는 셀프호스팅 **IdP/SSO 플랫폼** (Keycloak, Authelia, Zitadel, Okta와 같은 범주)입니다.

단, 615개 REST API가 OpenAPI로 완전 문서화되어 있으므로 **이걸 감싸는 MCP 서버를 직접 만들 수 있습니다.**
공식 제공은 없습니다 — 이것이 가장 유망한 사업 기회입니다.

---

## 10. API 토큰

| 하려는 일 | 토큰 필요 |
| --- | --- |
| 웹 UI 관리 | 불필요 (브라우저 세션) |
| 앱에 SSO 붙이기 | 불필요 (OAuth Client ID/Secret 또는 SAML 메타데이터) |
| 스크립트/CI/Terraform 자동화 | **필요** |
| AI 에이전트의 API 호출 | **필요 (Agent 토큰)** |

### 토큰 3종류

| 종류 | 발급 경로 | 특징 |
| --- | --- | --- |
| 사용자 API 토큰 | 콘솔 → Tokens | 내 권한으로 동작 |
| 서비스 어카운트 토큰 | 서비스 계정 생성 시 | 사람과 분리된 머신 신원 |
| Agent 토큰 (EE) | `POST /api/v3/agents/agents/` | 생성 시 1회만 노출, 항상 만료, 만료 시 갱신 없이 삭제 |

```bash
# 공통 인증
curl -H "Authorization: Bearer <token>" \
     https://authentik.company/api/v3/core/users/me/

# Agent 생성
curl -X POST \
  -H "Authorization: Bearer <admin-token>" \
  -H "Content-Type: application/json" \
  -d '{"parent":123,"label":"automation-agent","expiring":true,
       "expires":"2027-01-01T00:00:00Z","policy_behavior":"MIRROR"}' \
  https://authentik.company/api/v3/agents/agents/
```

**보안 수칙**

- 토큰은 시크릿 매니저에 저장. Git 커밋·로그 출력 금지
- Agent는 만료 설정 필수 (셀프서비스 Agent는 강제 만료 + 권한 NONE)
- 사람 계정 토큰을 자동화에 재사용하지 말고 전용 신원 발급
- `token_identifier`는 식별자일 뿐 인증에 사용 불가

### Agent API 오퍼레이션

| 작업 | 메서드 | 엔드포인트 |
| --- | --- | --- |
| 목록 | GET | `/api/v3/agents/agents/` |
| 생성 | POST | `/api/v3/agents/agents/` |
| 조회 | GET | `/api/v3/agents/agents/<id>/` |
| 수정 | PUT / PATCH | `/api/v3/agents/agents/<id>/` |
| 삭제 | DELETE | `/api/v3/agents/agents/<id>/` |
| 에이전트 권한 신청 | POST | `/api/v3/requests/grant-requests/agent/` |

`parent`, `policy_behavior`, `token_identifier`는 생성 후 읽기 전용입니다.

---

## 11. AI 에이전트 구축에 도움이 되는가

**매우 큰 도움이 됩니다.** AI 에이전트 실무 투입의 병목은 모델 성능이 아니라 "권한을 어떻게 줄 것인가"이며,
authentik 2026.8+ 는 이를 제품 기능으로 해결했습니다.

1. **에이전트 전용 신원** — 비밀번호 없음, API 토큰만, 부모 유저 종속, 만료 내장
2. **권한 상한선 보장** — MIRROR/COPY/NONE, 생성 후 변경 불가
3. **Human-in-the-loop 승인** — `grant-requests/agent` + `fulfill_url`, 시간제한 부여
4. **감사·책임 추적** — 에이전트 행동을 주인 유저로 귀속 기록
5. **615개 REST API** = 에이전트의 툴 카탈로그 (MCP/함수호출 스키마로 기계 변환 가능)

### 구현 가능한 아키텍처

```
Claude / LangGraph / 자체 에이전트
    | Bearer <agent-token>
MCP 서버 (직접 구현)  <->  authentik REST API 615개
    | 권한 부족 시
POST /requests/grant-requests/agent/  ->  사람에게 승인 URL
    | 승인
시간제한 권한으로 작업 수행  ->  전부 감사 로그
```

### 한계

- Agent 계정·PAM은 **Enterprise(유료)** — 개발/테스트는 무료, 운영은 구독 필요
- MCP 서버 공식 제공 없음 (직접 구현 필요 = 기회)
- 2026.8 신기능이라 커뮤니티 사례가 적음 (선점에는 유리)

---

## 12. React / PHP 로 만들 수 있는가

### 12-1. authentik 자체 재구현 — 사실상 불가

- 코드량 약 60만 줄 (Python 20만 + TS 33만 + Go 4.9만 + Rust 2.2만)
- SAML 2.0 서명/암호화, OAuth2+OIDC 전체, LDAP v3, RADIUS/EAP-TLS, SCIM, WebAuthn/FIDO2
- 인증 서버 버그 = 전사 계정 탈취 (`SECURITY.md`, `tests/openid_conformance/` 존재 이유)
- 숙련 팀 기준 수십 man-year

**결론: 재구현하지 마세요.** 인증은 검증된 것을 쓰는 것이 업계 철칙입니다.

### 12-2. React / PHP 앱을 연동 — 매우 쉬움 (정답)

**React**

```bash
npm install oidc-client-ts react-oidc-context
```

```jsx
import { AuthProvider, useAuth } from "react-oidc-context";

const config = {
  authority: "https://authentik.company/application/o/myapp/",
  client_id: "<client-id>",
  redirect_uri: "https://myapp.company/callback",
  scope: "openid profile email",
};

export default () => (
  <AuthProvider {...config}><App /></AuthProvider>
);
```

**PHP**

```bash
composer require jumbojett/openid-connect-php
```

```php
<?php
use Jumbojett\OpenIDConnectClient;

$oidc = new OpenIDConnectClient(
    'https://authentik.company/application/o/myapp/',
    '<client-id>',
    '<client-secret>'
);
$oidc->setRedirectURL('https://myapp.company/callback.php');
$oidc->addScope(['openid', 'profile', 'email']);
$oidc->authenticate();
$email = $oidc->requestUserInfo('email');
```

- Laravel: `laravel/socialite` + generic OIDC
- Symfony: `knpuniversity/oauth2-client-bundle`
- WordPress: OpenID Connect Generic 플러그인 (코드 0줄)
- 레거시 PHP: **Proxy Provider로 코드 수정 없이 인증 부착**

### 12-3. React / PHP 로 만들면 좋은 것

| 만들 것 | 스택 | 가치 |
| --- | --- | --- |
| 셀프서비스 관리 포털 | React + `@goauthentik/api` | 비개발자용 간소 UI |
| Agent 승인 모바일 웹 | React | `fulfill_url` 승인 전용 경량 앱 |
| 온보딩/오프보딩 대시보드 | Next.js | 입·퇴사 자동화 콘솔 |
| 멀티테넌트 통합 관리 | Laravel | MSP용 고객 일괄 관리 |
| **authentik MCP 서버** | TypeScript | 수요 최고 |

---

## 13. 유튜브 강의 제작 가능성

### 법적 확인

| 항목 | 상태 |
| --- | --- |
| 코드 (MIT) | 교육·상업적 활용 자유 |
| 문서 (CC BY-SA 4.0) | 인용 시 출처 표기 + 파생물 동일 라이선스 |
| EE 기능 | 소개·시연 가능, 소스 배포 불가 |
| 상표 | 소문자 `authentik` 유지, "공식" 오인 표현 금지, "비공식" 명시 권장 |
| AI 생성 미디어 금지 | `AI_POLICY.md`는 프로젝트 기여 정책. 내 채널에는 적용 안 됨 |

### 시장 상황

- 한국어 authentik 콘텐츠 거의 없음 → 블루오션
- 영어권도 2026.x 신기능(Agent/PAM) 영상은 사실상 0 → 선점 가능
- 수요층: 홈랩/셀프호스팅(규모 큼) + 사내 인프라 담당자(구매력 높음)

### 추천 커리큘럼

**시즌 1 — 입문 (유입)**
1. 비밀번호 20개 외우는 삶 끝내기 — 5분 설치
2. 구글 로그인 붙이기 (Social Login)
3. 2FA 전사 적용 (TOTP/WebAuthn)
4. **로그인 없는 앱에 로그인 붙이기 (Proxy Provider)** — 조회수 기대 최고
5. NAS·Proxmox·*arr 스택 통합

**시즌 2 — 실무 (전환)**
6. 플로우/스테이지 설계 (코딩 없이)
7. React 앱 OIDC 연동 라이브 코딩
8. PHP/WordPress/Laravel 연동
9. 정책 엔진 (해외 IP 차단, 평판 기반)
10. 블루프린트 GitOps

**시즌 3 — 차별화 (수익화 핵심)**
11. AI 에이전트에게 사내 권한을 안전하게 주는 방법
12. MIRROR/COPY/NONE — AI 권한 상한선 설계
13. Human-in-the-loop 승인 플로우 구현
14. **authentik MCP 서버 직접 만들기**
15. AI 행동 감사 추적 — 규제 대응

### 제작 팁

- 라이브 데모 중심 (슬라이드만 있으면 이탈)
- 영상마다 GitHub 리포 + `compose.yml` + `.env` 샘플 제공
- 문제 제기형 썸네일, 입문 8~12분 / 실무 15~25분
- 한국어 먼저 → 영어 자막·더빙 (시장 10배)
- 버전 명시 필수 (인증 제품은 버전차가 큼)

---

## 14. 수익화 전략 — 10개 모델

### 전략 원칙

1. **제품으로 정면 승부 금지** — 포크해서 "내 SSO"로 파는 것은 승산 없음 + EE 라이선스 리스크
2. **보완재를 팔아라** — 본체는 공짜여도 운영자의 시간·지식·연동은 유료
3. **지금의 공백은 AI x 아이덴티티** — Agent/PAM은 신기능, 한국어 자료 0, MCP 공식 제공 0

### 모델 1. authentik MCP 서버 (오픈코어) — 최우선 추천

615개 API를 MCP 툴로 노출. AI가 사내 아이덴티티를 직접 조작.

| 티어 | 내용 | 가격 |
| --- | --- | --- |
| 무료 (MIT) | 읽기 전용 — 조회·리포트 | $0 (저변 확보) |
| Pro | 쓰기 툴 + 승인 플로우 + 감사 대시보드 | $29~79/월/조직 |
| Enterprise | 온프렘, SSO, SLA, 커스텀 툴 | $500~2,000/월 |

초기 비용 거의 0, 개발 2~4주. 난이도 2/5. 예상 월 $500~5,000 (1년 내).
차별점은 "쓰기 권한의 공포"를 `grant-requests/agent` 승인 플로우로 해소하는 것.

### 모델 2. AI 에이전트 거버넌스 컨설팅

| 상품 | 범위 | 가격 |
| --- | --- | --- |
| 진단 (2일) | 리스크 보고서 + 로드맵 | 300~500만원 |
| 구축 (2~4주) | Agent 체계 + 승인 플로우 + 감사 설계 | 1,500~4,000만원 |
| 운영 (월) | 모니터링·정책 조정·분기 리뷰 | 월 150~400만원 |

난이도 3/5. 예상 연 5,000만~2억원 (1인 기준). 리드는 유튜브/블로그로 인바운드 확보.

### 모델 3. 구축·마이그레이션 SI

비용 절감이 숫자로 증명됩니다.

```
Okta         : 300명 x $6/월 x 12 = 연 2,160만원
authentik    : 서버 + 운영        = 연 400~800만원
절감          연 1,300~1,700만원  (구축비를 1년 내 회수)
```

| 규모 | 범위 | 가격 |
| --- | --- | --- |
| 소규모 (50명 이하) | 설치 + 앱 5개 + 2FA | 300~600만원 |
| 중규모 (300명) | AD 연동 + 앱 20개 + HA + 교육 | 1,500~3,500만원 |
| 이관 | Okta/Auth0 → authentik 무중단 | 3,000만~1억원 |

수익화 속도 최고, 확장성 낮음 (노동집약).

### 모델 4. 매니지드 호스팅 (SSO-as-a-Service)

| 플랜 | 내용 | 월 요금 |
| --- | --- | --- |
| Starter | 50명, 앱 5개, 공용 인프라 | 15~30만원 |
| Business | 300명, 무제한 앱, 백업, 99.9% | 60~150만원 |
| Dedicated | 전용 인스턴스, VPC, 커스텀 도메인 | 200~500만원 |

원가는 고객당 월 2~5만원 → 마진 80%+, MRR 축적.
난이도 4/5 (24/7 운영 책임). 고객 20개 = 월 1,200만원.
주의: EE 기능 재판매는 구독 라이선스 확인 필수.

### 모델 5. 유튜브 + 유료 강의 (콘텐츠 퍼널)

| 상품 | 가격 |
| --- | --- |
| 애드센스 | 월 20~200만원 |
| 인프런/유데미 강의 | 8~15만원/인 (300명 = 3,000만원) |
| 라이브 부트캠프 (2일) | 40~80만원/인 (15명 = 900만원) |
| 기업 출장 교육 (1일) | 200~400만원/회 (마진 최고) |
| 멤버십 | 월 1~3만원 (100명 = 월 200만원) |

초기 6개월 수익 거의 0 → 모델 3과 병행.

### 모델 6. 한국 특화 연동 플러그인 판매

- 카카오·네이버 소셜 로그인, PASS 본인인증, 공동인증서
- 국내 SMS 게이트웨이 (알리고/솔라피)
- 그룹웨어: 더존, 영림원, 한컴
- 행정전자서명(GPKI), 개인정보보호법 감사 템플릿

가격: 개별 30~200만원 (영구) 또는 월 5~20만원.
글로벌 프로젝트는 한국 시장을 다루지 않습니다 — 언어·규제 장벽이 방어막.
주의: EE 영역은 건드리지 말고 OSS 영역에서만 개발.

### 모델 7. 업종별 턴키 블루프린트 패키지

`blueprints/`의 YAML 선언형 설정을 업종별 완성 템플릿으로 패키징.

```
의료기관팩  — 환자정보 접근통제, 접속기록 보존, 의료법
금융팩      — 전자금융감독규정, 2FA 강제, 특권계정 관리
교육기관팩  — 학사시스템 연동, 학기별 계정 수명주기
공공기관팩  — 망분리, GPKI, 보안감사 리포트
스타트업팩  — Slack/Notion/GitHub/AWS 10종 즉시 연동
```

팩당 200~800만원 (설치 지원 포함). **한 번 만들어 여러 번 판다** = SI와의 결정적 차이.

### 모델 8. 감사·컴플라이언스 리포팅 SaaS

authentik 이벤트 로그 → ISMS-P / ISO 27001 / SOC 2 심사 리포트 자동 생성.
출력 예: 특권계정 접근 내역, 2FA 미적용 계정, AI 에이전트 권한 부여·회수 이력, 퇴사자 처리 증빙.

월 30~100만원/조직. 고객 10개 = 월 500만원.
담당자가 매번 수십 시간 쓰는 작업을 대체하는 것이 가치. AI 감사 추적은 EU AI Act 대응으로도 판매 가능.

### 모델 9. 기술 콘텐츠·뉴스레터·스폰서십

| 수익원 | 금액 |
| --- | --- |
| 유료 뉴스레터 | 월 5,000~15,000원/인 x 500명 = 월 250~750만원 |
| 블로그 스폰서 | 포스트당 50~300만원 |
| 전자책/템플릿 | 2~5만원 x 판매량 |
| 콘퍼런스 발표 | 회당 50~200만원 + 리드 |

본질은 모델 1~4의 리드 생성 엔진.

### 모델 10. 수직 SaaS에 내장 (B2B 임베디드)

내가 만드는 SaaS에 authentik을 인증 엔진으로 내장 (`brands/` + `tenants/` 멀티테넌시).
B2B SaaS에서 "엔터프라이즈 SAML SSO 지원"은 상위 플랜 판매의 필수 조건입니다.
직접 구현 6개월 vs authentik 내장 2주 — 그 시간차가 매출입니다.
상위 플랜에 SSO/SCIM을 붙여 가격 +30~50% 인상 정당화.

---

## 15. 종합 비교 및 실행 로드맵

| # | 모델 | 초기비용 | 난이도 | 수익화 속도 | 확장성 | 추천도 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | MCP 서버 (오픈코어) | 거의 0 | 2/5 | 빠름 | 매우 높음 | 5/5 |
| 2 | AI 거버넌스 컨설팅 | 거의 0 | 3/5 | 중간 | 중간 | 5/5 |
| 5 | 유튜브 + 유료강의 | 낮음 | 2/5 | 느림 | 높음 | 5/5 |
| 3 | 구축·마이그레이션 SI | 거의 0 | 3/5 | 매우 빠름 | 낮음 | 4/5 |
| 7 | 업종별 블루프린트 팩 | 낮음 | 3/5 | 중간 | 높음 | 4/5 |
| 6 | 한국특화 플러그인 | 낮음 | 3/5 | 중간 | 중간 | 4/5 |
| 4 | 매니지드 호스팅 | 높음 | 4/5 | 느림 | 높음 | 3/5 |
| 8 | 컴플라이언스 SaaS | 중간 | 4/5 | 느림 | 높음 | 3/5 |
| 10 | 내 SaaS에 내장 | 중간 | 3/5 | 느림 | 높음 | 3/5 |
| 9 | 콘텐츠·스폰서십 | 거의 0 | 2/5 | 느림 | 중간 | 2/5 |

### 1인 창업자 12개월 실행 순서

```
[0~3개월]  기반 구축 (비용 0)
  - authentik 셀프호스팅 직접 운영
  - 유튜브 한국어 입문 5편            -> 신뢰 자산
  - MCP 서버 무료 버전 공개           -> GitHub 스타 = 영업 자산

[3~6개월]  첫 현금 흐름
  - 유튜브 유입 -> 소규모 구축 수주   -> 즉시 매출
  - MCP Pro 티어 출시
  - AI 에이전트 편 영상 공개          -> 차별화 시작

[6~12개월]  고마진 전환
  - AI 거버넌스 컨설팅 패키지 판매    -> 단가 급상승
  - 업종별 블루프린트 팩 1개 완성     -> 반복 판매
  - 유료 강의 출시

[12개월+]  확장
  - 매니지드 호스팅 또는 컴플라이언스 SaaS -> MRR
```

### 법적 체크리스트

| 항목 | 상태 |
| --- | --- |
| OSS 코드 상업 이용 | 가능 (MIT) |
| EE 기능 운영 사용 | 구독 라이선스 필요 (개발/테스트는 무료) |
| EE 코드 복사·재배포·판매 | 금지 |
| 문서 인용 | CC BY-SA 4.0 — 출처 표기 + 동일 라이선스 |
| 상표 `authentik` | 소문자 유지, "공식" 오인 표현 금지, "비공식" 명시 권장 |
| AI 생성 미디어 | 내 채널 무관 / 업스트림 기여 시 금지 |

**핵심 조언**: 모델 1(MCP 서버)을 오픈소스로 먼저 공개하세요.
GitHub 스타와 유튜브 구독자가 쌓이면 모델 2·3·5의 고객이 찾아옵니다.
영업하지 않고 영업되는 구조를 먼저 만드는 것이 1인 사업의 승부처입니다.

---

## 16. 참고 링크

| 구분 | 주소 |
| --- | --- |
| 내 포크 | <https://github.com/bmshin94/authentik> |
| 업스트림 | <https://github.com/goauthentik/authentik> |
| 홈페이지 / 가격 | <https://goauthentik.io> · <https://goauthentik.io/pricing> |
| 문서 | <https://docs.goauthentik.io> |
| 개발자 문서 | <https://docs.goauthentik.io/docs/developer-docs/> |
| 연동 가이드 | <https://integrations.goauthentik.io> |
| API 레퍼런스 | <https://api.goauthentik.io> |
| Helm 차트 | <https://github.com/goauthentik/helm> |
| Discord | <https://goauthentik.io/discord> |
| Agent 계정 문서 | `website/docs/users-sources/user/account-types/agent-accounts.mdx` |
| Agent 모델 코드 | `authentik/enterprise/agents/models.py` |
| 승인 플로우 코드 | `authentik/enterprise/requests/api/grant_requests.py` |
| 컨테이너 compose | `lifecycle/container/compose.yml` |
| 기여 가이드 | `AGENTS.md` · `CONTRIBUTING.md` · `AI_POLICY.md` · `SECURITY.md` |
