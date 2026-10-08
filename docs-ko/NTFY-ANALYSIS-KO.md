# ntfy 전수조사 분석 & 활용/수익화 정리 (한국어)

> 작성일: 2026-10-08
> 분석 대상: **ntfy** — HTTP 기반 Pub/Sub 푸시 알림 서비스
> 분석 범위: 저장소 전체 (Go 서버/CLI 약 56,000줄 + React 웹앱 164파일 + 문서 158파일)

---

## 🔗 GitHub 주소

| 구분 | 주소 |
|---|---|
| **원본 저장소 (upstream)** | https://github.com/binwiederhier/ntfy |
| **이 저장소 (fork)** | https://github.com/bmshin94/ntfy |
| 안드로이드 앱 | https://github.com/binwiederhier/ntfy-android |
| iOS 앱 | https://github.com/binwiederhier/ntfy-ios |
| 공개 서비스 | https://ntfy.sh |
| 웹앱 | https://ntfy.sh/app |
| 공식 문서 | https://ntfy.sh/docs/ |
| Google Play | https://play.google.com/store/apps/details?id=io.heckel.ntfy |
| F-Droid | https://f-droid.org/en/packages/io.heckel.ntfy/ |
| App Store | https://apps.apple.com/us/app/ntfy/id1625396347 |
| Discord | https://discord.gg/cT7ECsZj9w |
| Matrix | https://matrix.to/#/#ntfy:matrix.org |
| Go 패키지 | https://pkg.go.dev/heckel.io/ntfy/v2 |

---

## 1. 한 줄 요약

```bash
curl -d "백업 완료!" ntfy.sh/mytopic
```

> **이 한 줄이면 내 스마트폰에 푸시 알림이 날아옵니다.**
> 회원가입 · 앱 개발 · API 키 · Firebase 설정 전부 불필요.

| 항목 | 내용 |
|---|---|
| 정체 | HTTP 기반 Pub/Sub **푸시 알림 서버 + CLI + 모바일/웹앱** (완제품 서비스) |
| 언어 | **Go 1.25.8** (서버/CLI, 약 56,000줄) + **React 19 + MUI 9 + Vite 8** (웹앱) |
| 모듈명 | `heckel.io/ntfy/v2` |
| 라이선스 | **Apache 2.0 + GPLv2 듀얼** (상업적 이용 가능) |
| 원작자 | Philipp C. Heckel (binwiederhier) |
| 스타 | ⭐ 24,000+ |

---

## 2. 작동 원리

```
[1] 발행(Publish)                [2] ntfy 서버          [3] 구독(Subscribe)
 서버 스크립트 / cron
 Grafana / GitHub Actions  ──POST/PUT──▶  토픽(topic) ──▶  📱 안드로이드 앱
 Python / PHP / Node                      "mytopic"        🍎 iOS 앱
 curl 한 줄                                                💻 웹브라우저 (PWA)
                                                           ⌨️  ntfy subscribe CLI
```

### 핵심 개념 = "토픽(topic)" 하나

- 토픽 = 내가 아무렇게나 정한 문자열 (`myserver-alerts` 등)
- **미리 만들 필요 없음.** 메시지를 보내는 순간 생성됨
- 공개 `ntfy.sh`에서는 **토픽 이름 = 비밀번호**

```bash
❌ ntfy.sh/alerts                   # 누구나 짐작 가능 → 내 알림 노출
❌ ntfy.sh/test                     # 최악
✅ ntfy.sh/bmshin-srv-a9Kx72mQp4    # 추측 불가 = 사실상 비밀번호
```

진짜 자물쇠(비밀번호/토큰)가 필요하면 → **자가호스팅** 또는 **Pro 플랜**

---

## 3. 폴더 전수조사

### 3.1 핵심 엔진

| 폴더/파일 | 파일수 | 역할 |
|---|---|---|
| `main.go` | 1 | 진입점 (48줄). `urfave/cli`로 CLI 구동 |
| `cmd/` | 27 | CLI 명령 전체: `serve`(50KB) · `publish` · `subscribe` · `user` · `access` · `token` · `tier` · `webpush` |
| `server/` | 58 | **심장부.** `server.go` 하나가 **84KB** |
| `message/` | 8 | 메시지 캐시 DB (SQLite/PostgreSQL). `since=`, `poll=1` 지원 |
| `model/` | 1 | `Message` 구조체 (전 패키지 공유) |

### 3.2 `server/` 내부 상세

| 파일 | 크기 | 역할 |
|---|---|---|
| `server.go` | 84KB | 라우터 + 발행/구독 핸들러 |
| `server_account.go` | 36KB | 회원가입/로그인/토큰/예약토픽 |
| `server_payments.go` | 24KB | **Stripe 결제, 구독 업그레이드** |
| `server_firebase.go` | 11KB | FCM (안드로이드 배터리 절약) |
| `server_webpush.go` | 7KB | 브라우저 웹푸시 (VAPID) |
| `server_matrix.go` | 7KB | Matrix 메신저 Push Gateway |
| `server_twilio.go` | 2KB | 📞 전화 발신 / SMS |
| `smtp_server.go` | 10KB | ✉️ **이메일 수신 → 푸시로 변환** |
| `server_template.go` | 10KB | 메시지 템플릿 엔진 |
| `visitor.go` | 19KB | 🚦 Rate limit, 어뷰징 방지 |
| `config.go` | 20KB | 설정 로딩 |
| `server.yml` | 26KB | 주석 포함 전체 설정 예시 |
| `templates/` | - | `grafana.yml` · `github.yml` · `alertmanager.yml` |
| `server_test.go` | **257KB** | 테스트가 본 코드보다 큼 (품질 지표) |

### 3.3 부가 기능 모듈

| 폴더 | 파일수 | 역할 |
|---|---|---|
| `user/` | 12 | 사용자/토큰/권한. `manager.go` 70KB. 토큰 접두사 **`tk_`**, 권한 4단계 |
| `db/` + `db/pg/` | 10 | PostgreSQL 커넥션 풀 + **읽기 복제본(replica)** 지원 |
| `attachment/` + `s3/` | 14 | 📎 첨부파일 (로컬 디스크 또는 S3 호환) |
| `webpush/` | 5 | 브라우저 푸시 구독 저장 |
| `mail/` | 4 | 알림 → 이메일 발송 |
| `twilio/` | 3 | 전화/SMS 발신 |
| `payments/` | 2 | **Stripe 결제 (ntfy Pro 수익 엔진)** |
| `metrics/` | 2 | **Prometheus `/metrics`** |
| `ban/` | 4 | 악성 IP 차단 피드 |
| `action/` | 2 | 🔘 **알림 안의 버튼** (`view` / `http` / `broadcast`) |
| `template/gotext/` | 10 | Go `text/template` 포크 내장 + sprig 함수 |
| `util/` | 47 | 유틸 + sprig 템플릿 함수 |
| `log/` | 4 | 구조화 로깅 (text/json, 레벨 오버라이드) |

### 3.4 UI / 문서 / 예제

| 폴더 | 파일수 | 내용 |
|---|---|---|
| `web/` | 164 | **React 19 + MUI 9 + Vite 8 PWA**. Dexie(IndexedDB), i18next(다국어), react-remark |
| `docs/` | 158 | MkDocs 공식 문서 (`publish.md`, `install.md`, `config.md`, `subscribe/*`, `faq.md` 등) |
| `examples/` | 12 | `publish-{go,php,python}` · `subscribe-{go,php,python}` · SSE HTML · WebSocket HTML · **`ssh-login-alert`**(PAM 훅) · **`grafana-dashboard`** · `linux-desktop-notifications` |
| `tools/` | 10 | `loadtest`(부하테스트) · `pgimport`(SQLite→PG 마이그레이션) |

### 3.5 빌드 / 배포 / CI

| 파일 | 내용 |
|---|---|
| `Makefile` (18KB) | 빌드·테스트·릴리스 전 과정 |
| `Dockerfile`, `Dockerfile-arm`, `Dockerfile-build`, `docker-compose.yml` | 컨테이너 배포 |
| `.goreleaser.yml` (6KB) | 멀티플랫폼 바이너리 + deb/rpm 자동 생성 |
| `.github/` (16) | GitHub Actions CI (test, lint, codecov) |
| `server/ntfy.service`, `ntfy.openrc` | systemd / OpenRC 서비스 등록 |
| `CLAUDE.md` | 🔸 원본에 없는 파일. 이 포크에서 Curator-Agent가 자동 생성한 한국어 가이드 |

### 3.6 조직도 비유

```
🏢 ntfy
├─ 🚪 main.go ──────────── 안내 데스크
├─ 📞 cmd/ ─────────────── 콜센터 (CLI 명령)
├─ 🧠 server/ ─────────── 본사 (직원 58명)
│   ├─ server.go ········· CEO (84KB)
│   ├─ topic.go ·········· 사서함 관리과
│   ├─ visitor.go ········ 경비팀 (속도제한)
│   ├─ server_account.go · 회원관리부
│   ├─ server_payments.go  💰 결제팀 (Stripe)
│   ├─ server_firebase.go  📮 안드로이드 배송팀
│   ├─ smtp_server.go ···· ✉️ 이메일 접수창구
│   └─ server_twilio.go ·· 📞 전화 발신팀
├─ 💾 message/, db/ ───── 창고 (메시지 보관)
├─ 👤 user/ ───────────── 인사부 (계정·토큰·권한)
├─ 📎 attachment/, s3/ ── 택배 보관소
├─ 🔘 action/ ─────────── 알림 버튼 설계팀
├─ 📝 template/ ───────── 문구 디자인팀
├─ 📊 metrics/ ────────── 통계실
├─ 💻 web/ ───────────── 웹사이트 팀 (React)
├─ 📚 docs/ ──────────── 매뉴얼 제작실
└─ 🎓 examples/ ──────── 교육자료실
```

---

## 4. API 전수조사 (코드에서 직접 추출)

### 4.1 발행 (Publish)

```
POST /<topic>                                 ← 가장 기본
PUT  /<topic>
GET  /<topic>/publish | /send | /trigger      ← 브라우저 주소창에서도 가능
POST /                                        ← JSON 바디에 topic 포함
```

### 4.2 구독 (Subscribe) — 4가지 방식

```
GET /<topic>/json    ← JSON 스트림 (한 줄에 메시지 하나)
GET /<topic>/sse     ← Server-Sent Events (브라우저 EventSource)
GET /<topic>/raw     ← 순수 텍스트
GET /<topic>/ws      ← WebSocket
```

여러 토픽 동시 구독: `/topic1,topic2,topic3/json`

### 4.3 메시지 수정/삭제

```
PUT    /<topic>/<id>                ← 메시지 수정
POST   /<topic>/<id>/read | clear   ← 읽음 처리
DELETE /<topic>/<id>/delete         ← 삭제
```

`X-Sequence-ID`로 **같은 알림 덮어쓰기** 가능 (진행률 0% → 50% → 100%)

### 4.4 관리 API

```
/v1/health  /v1/version  /v1/config  /v1/stats  /v1/tiers
/v1/account  /v1/account/{login,token,password,settings,subscription,
             reservation,phone,email,billing/*}
/v1/users  /v1/users/access
/v1/webpush
/_matrix/push/v1/notify     ← Matrix 메신저 게이트웨이
/metrics                    ← Prometheus
```

### 4.5 메시지 전체 옵션 (`server/types.go`)

| 헤더 | JSON 필드 | 설명 |
|---|---|---|
| `X-Title` / `Title` / `t` | `title` | 알림 제목 |
| `X-Priority` / `prio` / `p` | `priority` | **1=min, 2=low, 3=default, 4=high, 5=max/urgent** |
| `X-Tags` / `tag` | `tags` | 이모지 태그 (`warning`→⚠️, `skull`→💀) |
| `X-Click` | `click` | 알림 탭 시 열릴 URL |
| `X-Icon` | `icon` | 알림 아이콘 |
| `X-Actions` | `actions` | 🔘 버튼 최대 3개 (`view`/`http`/`broadcast`) |
| `X-Attach` / `X-Filename` | `attach`/`filename` | 📎 첨부파일 |
| `X-Markdown` | `markdown` | 마크다운 렌더링 |
| `X-Email` | `email` | 📧 이메일 동시 발송 |
| `X-Call` | `call` | 📞 전화 걸기 |
| `X-Delay` | `delay` | ⏰ 예약 발송 (`30min`, `tomorrow 10am` — 자연어 파싱, `olebedev/when`) |
| `X-Cache` | `cache` | 서버 저장 여부 |
| `X-Firebase` | `firebase` | FCM 전송 여부 |
| `X-Sequence-ID` | `sequence_id` | 알림 덮어쓰기용 ID |

---

## 5. 언제 쓰는가 — 실전 시나리오

| 상황 | 명령/방법 |
|---|---|
| 긴 작업 완료 알림 | `./빌드.sh && curl -d "빌드 완료 ✅" ntfy.sh/mytopic` |
| **SSH 침입 감지** | PAM 훅 등록 (`examples/ssh-login-alert`) |
| 서버 백업 cron | `0 3 * * * /backup.sh && curl -d "백업 성공" ntfy.sh/...` |
| 디스크 full 경고 | `df` 체크 스크립트 + `-H "Priority: urgent"` |
| Grafana/Prometheus 알럿 | Webhook URL을 ntfy로 지정 (`server/templates/grafana.yml` 내장) |
| GitHub Actions 실패 | 워크플로에 curl 한 줄 (`templates/github.yml` 내장) |
| 주식/코인 가격 도달 | 가격 체크 Python 루프 |
| 택배·크롤링 변동 알림 | 크롤러에서 변동 시 발행 |
| **Claude Code 작업 완료** | Stop 훅에 curl |
| 홈서버 UPS 정전 | NUT 스크립트 연동 |

---

## 6. 설치 및 사용법

### 6.1 설치 없이 쓰기 (대부분 이걸로 충분)

```bash
curl -d "메시지" ntfy.sh/내가정한-긴-토픽이름-x7k9m2
```
+ 폰에 ntfy 앱 설치 (Play스토어 / App Store / F-Droid)
+ 또는 https://ntfy.sh/app 에서 토픽 구독 (PWA 설치 가능)

### 6.2 CLI 설치

```bash
# macOS
brew install ntfy

# Debian/Ubuntu
curl -fsSL https://archive.ntfy.sh/apt/keyring.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/ntfy.gpg
echo "deb [signed-by=/etc/apt/keyrings/ntfy.gpg] https://archive.ntfy.sh/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/ntfy.list
sudo apt update && sudo apt install ntfy

# 바이너리 직접 (Linux amd64)
wget https://github.com/binwiederhier/ntfy/releases/latest/download/ntfy_linux_amd64.tar.gz
tar zxvf ntfy_*.tar.gz && sudo cp ntfy_*/ntfy /usr/local/bin/
```
그 외 Arch / Fedora / FreeBSD / NixOS / Windows(Scoop·Chocolatey) 모두 지원.

```bash
ntfy publish mytopic "메시지"
ntfy publish -t "제목" -p high --tags warning mytopic "본문"
ntfy subscribe mytopic                   # 터미널 실시간 수신
ntfy subscribe mytopic '내스크립트.sh'    # 알림 수신 시 스크립트 실행
```

### 6.3 자가호스팅

```bash
docker run -p 80:80 -itd binwiederhier/ntfy serve   # 가장 빠름
docker compose up -d                                 # 저장소에 compose 포함
make build                                           # 소스 빌드 (Go 1.25+)
```

`/etc/ntfy/server.yml` 예시 (전체 주석 예시는 `server/server.yml` 26KB 참고):

```yaml
base-url: "https://ntfy.example.com"
listen-http: ":80"

cache-file: "/var/cache/ntfy/cache.db"
cache-duration: "12h"

auth-file: "/var/lib/ntfy/user.db"
auth-default-access: "deny-all"      # 비공개 서버라면 필수

attachment-cache-dir: "/var/cache/ntfy/attachments"
attachment-total-size-limit: "5G"

# 대규모 운영
# database-url: "postgres://user:pass@host:5432/ntfy?pool_max_conns=50"

enable-metrics: true
```

```bash
sudo systemctl enable ntfy && sudo systemctl start ntfy
```

Kubernetes / Kustomize / Helm 설치법은 `docs/install.md` 참고.

### 6.4 사용자/권한/토큰 관리

```bash
ntfy user add --role=admin philipp
ntfy user add bmshin
ntfy access bmshin "myserver-*" rw      # 와일드카드 권한
ntfy token add --expires=30d bmshin     # tk_... 토큰 발급
ntfy tier add --name="Pro" starter      # 요금제 티어
```

### 6.5 알림 꾸미기 & 버튼

```bash
# 꾸미기
curl -H "Title: 🚨 디스크 경고" -H "Priority: urgent" \
     -H "Tags: warning,skull" -H "Click: https://my-dashboard.com" \
     -d "서버 디스크 95% 사용 중!" ntfy.sh/mytopic

# 알림에 버튼 달기 (침대에서 서버 재시작!)
curl -H "Actions: http, 서버 재시작, https://api.내서버.com/restart, method=POST; \
                  view, 로그 보기, https://logs.내서버.com" \
     -d "서버 응답 없음. 어떻게 할까요?" ntfy.sh/mytopic
```

---

## 7. 플러그인? 스킬? MCP?

### 결론: **셋 다 아닙니다. "독립 실행 서비스(완제품)" 입니다.**

| 구분 | 해당 | 설명 |
|---|---|---|
| 플러그인 | ❌ | 다른 프로그램에 끼워넣는 모듈 아님 |
| 스킬(Claude Skill) | ❌ | Claude가 읽는 지침서 파일 아님 |
| MCP 서버 | ❌ | MCP 프로토콜 구현체 **아님** |
| **독립 서버 + CLI + 모바일앱** | ✅ | **정답.** HTTP REST API를 가진 메시지 브로커 |

언어·플랫폼 무관하게 `curl`만 되면 전부 연동 — 그게 이 프로젝트의 철학.

### 하지만 셋 다로 **감쌀 수 있음** (여기가 기회)

**① Claude Code 훅 — 가장 쉽고 즉시 유용 ⭐⭐⭐**

```json
// .claude/settings.json
{
  "hooks": {
    "Stop": [{
      "hooks": [{ "type": "command",
        "command": "curl -s -H 'Title: ✅ Claude 완료' -H 'Tags: robot' -d \"$(pwd) 작업 종료\" ntfy.sh/내토픽" }]
    }],
    "Notification": [{
      "hooks": [{ "type": "command",
        "command": "curl -s -H 'Title: ❓ Claude가 질문함' -H 'Priority: high' -d '승인 대기 중' ntfy.sh/내토픽" }]
    }]
  }
}
```

**② MCP 서버로 래핑 ⭐⭐** — `ntfy_publish` / `ntfy_subscribe` 툴 노출. Python/TS 50~100줄
**③ Claude Skill ⭐** — `SKILL.md`에 "알림 보낼 땐 이렇게" 규칙 정리

> ntfy는 **"재료"**, 플러그인/스킬/MCP는 **"요리법"**.

---

## 8. API 토큰이 필요한가?

| 상황 | 토큰 필요 |
|---|---|
| ntfy.sh 공개 토픽 발행 | ❌ 불필요 |
| ntfy.sh 공개 토픽 구독 | ❌ 불필요 |
| ntfy.sh **예약 토픽**(Pro 기능) | ✅ 필요 |
| 자가호스팅 + `auth-file` 설정 | ✅ 필요 |
| 자가호스팅 + 인증 없음 | ❌ 불필요 |
| **프로덕션/업무용** | ✅ **필수로 쓸 것** |

### 인증 2가지 (`server/server_auth.go`)

```bash
# 1) Basic 인증
curl -u "bmshin:mypassword" -d "메시지" https://my-ntfy.com/mytopic

# 2) Bearer 토큰 (권장, tk_ 접두사 — user/manager.go:32)
ntfy token add --expires=90d bmshin
# → tk_AgQdq7mVBoFD37zQVN29RhuMzNIz2
curl -H "Authorization: Bearer tk_AgQdq..." -d "메시지" https://my-ntfy.com/mytopic
```

### 권한 4단계 (`user/types.go:222`)

```
deny-all (0) → read (1) → write (2) → read-write (3)
```

```bash
ntfy access ci-bot "deploy-*" write     # 쓰기만, deploy-* 토픽만
ntfy access viewer "logs-*"  read       # 읽기만
```

### 실무 권장

1. 비밀번호보다 **토큰** 사용 (유출 시 해당 토큰만 폐기)
2. **만료 설정** (`--expires=30d`)
3. **최소 권한** (발행 봇에게는 `write`만)
4. **환경변수 관리** — 코드 하드코딩 금지
5. 공개 ntfy.sh 사용 시 **민감 정보 전송 금지** (토픽 이름이 유일한 보안)

---

## 9. AI 에이전트 구축에 도움이 되는가? → ⭐⭐⭐⭐⭐

AI 에이전트의 가장 큰 구조적 약점:

> **"사람이 화면을 보고 있지 않으면 아무 일도 못 한다"**

ntfy가 메우는 게 정확히 이 구멍이다.

| # | 활용 | 설명 |
|---|---|---|
| ① | **작업 완료 알림** | 긴 리팩토링/빌드 종료 시 폰 알림 |
| ② | 🔥 **Human-in-the-Loop 승인** | Action 버튼이 HTTP 요청 → **폰 버튼 하나로 에이전트의 위험 작업 승인/거부** |
| ③ | **에이전트 간 메시지 버스** | `/ws`, `/sse`, `/json` 스트림으로 Redis/RabbitMQ 없이 멀티에이전트 통신 |
| ④ | **에이전트에게 "사람 호출" 도구 부여** | MCP 래핑 → LLM이 직접 판단해 사람 호출 |
| ⑤ | **장시간 자율 실행 모니터링** | `X-Sequence-ID`로 같은 알림 덮어쓰며 진행률 갱신 |
| ⑥ | **에러 즉시 에스컬레이션** | 3회 재시도 실패 → `Priority: urgent` + `X-Call`로 전화까지 |

### ② 승인 워크플로 실제 예시

```bash
curl \
  -H "Title: ⚠️ 프로덕션 DB 마이그레이션 승인 요청" \
  -H "Priority: urgent" \
  -H "Actions: http, ✅ 승인, https://my-agent.com/approve/req-123, method=POST; \
               http, ❌ 거부, https://my-agent.com/deny/req-123, method=POST" \
  -d "에이전트가 users 테이블 컬럼 추가를 요청. 영향 행: 2.4M" \
  ntfy.sh/agent-approval-x9k2
```

> **결론: 비동기·장시간·자율 실행 에이전트에게 ntfy는 "있으면 좋은" 게 아니라 사실상 필수 인프라.**

---

## 10. React / PHP로 만들 수 있는가?

### ✅ ① 클라이언트/연동 — 완전히 가능, 아주 쉬움 (난이도 ⭐)

**PHP 발행** (저장소 `examples/publish-php/publish.php`에 이미 존재):

```php
<?php
file_get_contents('https://ntfy.sh/mytopic', false, stream_context_create([
    'http' => [
        'method' => 'POST',
        'header' => "Content-Type: text/plain\r\n" .
                    "Title: 주문 접수\r\n" .
                    "Priority: high\r\n" .
                    "Tags: moneybag",
        'content' => '새 주문 1건 (45,000원)'
    ]
]));
```

**Laravel:**
```php
Http::withHeaders(['Title' => '배포 완료', 'Priority' => 'high'])
    ->withToken($ntfyToken)
    ->post('https://ntfy.example.com/deploy', '프로덕션 배포 성공');
```

**React 구독 (SSE):**
```jsx
import { useEffect, useState } from "react";

export function useNtfy(topic) {
  const [messages, setMessages] = useState([]);
  useEffect(() => {
    const es = new EventSource(`https://ntfy.sh/${topic}/sse`);
    es.onmessage = (e) => setMessages((p) => [JSON.parse(e.data), ...p]);
    return () => es.close();
  }, [topic]);
  return messages;
}
```

**React 발행:**
```jsx
await fetch("https://ntfy.sh/mytopic", {
  method: "POST",
  headers: { Title: "제목", Priority: "high", Tags: "tada" },
  body: "본문",
});
```

> 🔸 `web/` 폴더의 공식 웹앱이 **이미 React 19 + MUI 9 + Vite 8** → 그 코드를 바로 참고·개조 가능.

### ⚠️ ② 서버를 React/PHP로 재구현 — 가능하나 비권장 (난이도 ⭐⭐⭐⭐⭐)

| 기능 | PHP로 가능 | 현실 |
|---|---|---|
| 발행 API | ✅ 쉬움 | 문제없음 |
| 메시지 저장 | ✅ | MySQL/SQLite |
| **수만 개 동시 롱커넥션 (SSE/WS)** | ❌ **치명적** | PHP-FPM은 요청당 프로세스 → 1만 연결 = 1만 프로세스 = 메모리 폭발 |
| 해결책 | ⚠️ | Swoole / ReactPHP / Workerman 등 비동기 런타임 필수 |

Go를 쓴 이유가 바로 이것. goroutine은 커넥션당 수 KB → 단일 서버 수십만 동시 연결.
React는 브라우저 언어라 애초에 서버 역할이 아님.

### ✅ ③ 현실적 최선 — 하이브리드 (강력 추천)

```
┌──────────────────────────────────┐
│ React 프론트엔드 (직접 개발)        │  ← 대시보드, 토픽 관리, 통계, 한글 UI
└────────────┬─────────────────────┘
             │ REST
┌────────────▼─────────────────────┐
│ PHP(Laravel) 비즈니스 백엔드        │  ← 회원, 국내 결제, 팀/조직, 템플릿
└────────────┬─────────────────────┘
             │ HTTP (Bearer tk_...)
┌────────────▼─────────────────────┐
│ ntfy (Go) — 그대로 사용 🔥         │  ← 실시간 전송 엔진만 담당
└──────────────────────────────────┘
```

개발 기간 재구현의 1/10, 안정성 10배.

---

## 11. 유튜브 강의 영상 제작 가능성 → ⭐⭐⭐⭐⭐

### 왜 좋은 소재인가

| 성공 요소 | 평가 |
|---|---|
| 즉각적 "와!" 순간 | ⭐⭐⭐⭐⭐ curl 치자마자 폰 띠링 — 영상 10초 안에 임팩트 |
| 진입장벽 | ⭐⭐⭐⭐⭐ 가입·결제·설정 없음 → 이탈 최소 |
| 실용성 | ⭐⭐⭐⭐⭐ "내 서버 알림" 수요는 영구적 |
| 썸네일 임팩트 | ⭐⭐⭐⭐⭐ "터미널 한 줄 → 폰 알림" |
| 한국어 콘텐츠 | ⭐⭐⭐⭐⭐ 거의 없음 = **블루오션** |
| 시리즈 확장성 | ⭐⭐⭐⭐⭐ 입문→자가호스팅→연동→AI |
| 제작 난이도 | ⭐⭐⭐⭐ 화면 녹화 + 폰 화면만 필요 |

### 추천 커리큘럼 12편

**🔰 입문부 (유입용)**

| # | 제목 | 길이 |
|---|---|---|
| 1 | **"터미널 한 줄로 내 폰에 알림 보내기 (10초면 됩니다)"** | 5분 |
| 2 | "알림 예쁘게 꾸미기 — 제목·이모지·긴급도·클릭" | 8분 |
| 3 | **"알림 버튼 눌러서 서버 재시작하기"** | 10분 |
| 4 | "오래 걸리는 명령 끝나면 알림받기 (개발자 필수 꿀팁)" | 6분 |

**⚙️ 실전부**

| # | 제목 | 길이 |
|---|---|---|
| 5 | **"SSH 침입 감지 알림 만들기"** (PAM 훅) | 12분 |
| 6 | "서버 디스크·CPU·메모리 감시 자동화" | 15분 |
| 7 | "Docker로 나만의 알림 서버 10분에 세우기" | 15분 |
| 8 | "Grafana / Prometheus / GitHub Actions 연동" | 18분 |

**🤖 고급·차별화부 (조회수 폭발 구간)**

| # | 제목 | 길이 |
|---|---|---|
| 9 | **"Claude Code가 코딩 끝내면 폰으로 알려주기"** 🔥 | 12분 |
| 10 | **"AI 에이전트의 위험한 작업을 폰 버튼으로 승인하기"** (국내 최초급) | 18분 |
| 11 | "React로 나만의 실시간 알림 대시보드 만들기" | 25분 |
| 12 | **"24000 스타 Go 프로젝트 코드 해부"** | 30분 |

### 제작 팁

**✅ 하세요**
- **0~5초에 결과부터** (폰 알림 띠링 장면을 맨 앞)
- **화면 2분할** — 터미널 / 폰 화면 (OBS + scrcpy)
- 복붙 가능한 명령어를 **설명란에 전부**
- GitHub 예제 저장소 만들어 링크
- "무료입니다" 강조

**❌ 피하세요**
- 영상에 **실제 토픽 이름 노출 금지** (전 세계가 내 알림 구독 가능) → 더미 토픽 + 모자이크
- 토큰·비밀번호 노출 금지
- 1편부터 자가호스팅/설정 파일 설명 (시청자 이탈)

### 수익 경로

```
유튜브 광고 → 멤버십/슈퍼챗 → 인프런·클래스101 강의(2~5만원)
→ 전자책 "개발자 알림 자동화 레시피 50선" → 기업 출강/컨설팅
→ SaaS 아이디어로 연결 (유튜브가 최고의 무료 마케팅)
```

> 라이선스(Apache 2.0)상 영상·강의 제작에 법적 문제 없음. 매너상 원저작자(Philipp C. Heckel)와 GitHub 링크 표기 권장.

---

## 12. 수익화 아이디어 상세

### 12.1 먼저 알아야 할 3가지 전제

**① 라이선스 — 합법적으로 돈 벌 수 있는가?**

```
Apache License 2.0 + GPLv2 듀얼
```

| 가능 | 주의 |
|---|---|
| ✅ 상업적 이용 | ⚠️ GPLv2 조건 배포 시 소스 공개 의무 발생 |
| ✅ 수정·개조 | ⚠️ **SaaS 형태(서버에서 서비스만 제공)는 소스 공개 의무 없음** ← 핵심 |
| ✅ 재배포·사설 사용 | ⚠️ 저작권·라이선스 사본 유지 필수 |
| ✅ 특허 사용권 부여 | ⚠️ 상표("ntfy" 이름) 무단 사용은 별개 → **다른 브랜드명 사용** |

> **SaaS 운영이 라이선스상 가장 깔끔.**

**② 모델이 이미 검증됨** — 원작자가 ntfy Pro($5~/월) + GitHub Sponsors + Liberapay로 실제 수익 중.

**③ 🔥 과금 시스템이 이미 코드에 있음**

```
payments/                     Stripe 연동
server/server_payments.go     (24KB) 구독 생성·업그레이드·웹훅·청구포털
cmd/tier.go                   (14KB) 요금제 티어 CRUD
/v1/tiers, /v1/account/billing/*   API
user/manager.go               티어별 한도(메시지·대역폭·첨부 용량)
```

→ **SaaS 창업의 2~3개월치 작업(결제·구독·한도관리)이 완성된 상태.** 최대 자산.

---

### 12.2 수익화 아이디어 10선

#### 🥇 1. 한국형 매니지드 알림 서비스 (SaaS)

| 항목 | 내용 |
|---|---|
| 💡 핵심 | ntfy를 엔진으로, **한국 시장에 없는 것**을 붙여 판매 |
| 🎯 타깃 | 국내 1인 개발자, 스타트업 개발팀, 쇼핑몰 운영자, 홈서버 운영자 |
| 🔧 차별화 | ① 완전 한글 UI/문서 ② **카카오톡 알림톡 / SMS / 네이버웍스 연동** ③ **국내 결제**(토스·카카오페이·계좌이체) ④ 국내 리전(낮은 지연, 데이터 국내 보관) ⑤ **세금계산서 발행** ⑥ 한국어 고객지원 |
| 💵 가격 | 무료(일 100건) / 베이직 ₩4,900 / 프로 ₩14,900 / 팀 ₩49,000 / 기업 별도 |
| ⏱ 개발 | 4~8주 |
| 📈 난이도 | ⭐⭐⭐ |
| ⚠️ 리스크 | ntfy.sh 무료 플랜과 경쟁 → "한국어·카톡·국내결제·세금계산서"가 방어선 |

**왜 되는가:** 국내 기업은 "해외 서비스, 영어 문서, 해외 결제, 세금계산서 불가" 때문에 좋은 도구를 못 쓴다. 그 틈이 비즈니스.

#### 🥈 2. AI 에이전트 알림·승인 플랫폼 🔥 (최우선 추천)

| 항목 | 내용 |
|---|---|
| 💡 핵심 | ntfy의 **Action 버튼**을 AI 에이전트 승인 워크플로로 전문화 |
| 🎯 타깃 | AI 에이전트 개발자, 자동화 운영팀, AI 스타트업 |
| 🔧 기능 | ① **MCP 서버 제공** ② 승인 요청 → 폰 버튼 → 결과 반환 전체 흐름 ③ **감사 추적(Audit Trail)** — 기업 필수 ④ 다단계 승인(팀장→임원) ⑤ 타임아웃 기본 동작 ⑥ Claude Code / LangChain / CrewAI / n8n 플러그인 |
| 💵 가격 | 승인 건당 + 월정액 하이브리드. $19 / $79 / 기업 $499+ |
| ⏱ 개발 | 8~12주 |
| 📈 난이도 | ⭐⭐⭐⭐ |
| 🚀 시장성 | **⭐⭐⭐⭐⭐ 최상** — 가장 뜨거운 미해결 문제, 경쟁자 거의 없음 |

**왜 되는가:** 기업이 AI 에이전트를 프로덕션에 못 올리는 1순위 이유 = **통제 불가**. "AI가 위험한 일을 하려면 반드시 사람 승인 + 기록"을 제품화하면 기업이 돈을 낸다. ntfy에 Action 버튼 + 감사 가능한 메시지 저장이 이미 있음.

#### 🥉 3. 교육 콘텐츠 사업 (가장 먼저 시작할 것)

| 항목 | 내용 |
|---|---|
| 💵 수익원 | ① 유튜브 광고·멤버십 ② 인프런/클래스101 강의(₩29,000~69,000) ③ 전자책 "개발자 알림 자동화 레시피 50선"(₩15,000) ④ 노션 템플릿·스크립트 묶음 ⑤ 기업 출강(회당 50~200만) ⑥ 1:1 컨설팅 |
| ⏱ 개발 | **2~4주**로 첫 수익 |
| 💸 초기비용 | **거의 0원** |
| 📈 난이도 | ⭐⭐ (가장 쉬움) |
| 🚀 시장성 | ⭐⭐⭐⭐ 한국어 콘텐츠 공백 = 선점 |

**왜 먼저 해야 하나:** ① 리스크 0 ② **시장 검증**(댓글로 수요 파악) ③ SaaS 출시 시 **이미 구축된 청중 = 무료 마케팅 채널**

#### 4. 업종 특화 수직 SaaS (Vertical SaaS)

| 업종 | 제품명 예 | 알림 내용 | 왜 돈을 내나 |
|---|---|---|---|
| 🛒 쇼핑몰 | "오더알림" | 신규주문·품절임박·CS·정산 | 주문 놓치면 매출 손실 |
| 🏥 병원/클리닉 | "예약알림" | 노쇼 예측·예약변경·장비이상 | 노쇼 1건 = 수만원 |
| 🏭 제조/설비 | "설비파수꾼" | 센서 임계치·가동중단·예방정비 | 설비 1시간 정지 = 수백만원 |
| 🍜 외식/프랜차이즈 | "매장알림" | 배달앱 주문·냉장고 온도·POS 오류 | 식자재 폐기 방지 |
| 🏠 부동산/임대 | "매물알림" | 신규매물·계약만료·관리비 연체 | 좋은 매물은 선착순 |
| 🚚 물류 | "배송알림" | 배송지연·온도이탈·차량이상 | 클레임 방지 |

가격: 범용보다 **5~20배** (월 5~50만원) / 개발 6~10주 / 난이도 ⭐⭐⭐⭐ / 시장성 ⭐⭐⭐⭐⭐

**왜 비싸게 받나:** "푸시 알림 서비스"는 월 5천원도 아깝지만, "설비 고장 1시간 전 알려주는 시스템"은 월 50만원도 싸다. 같은 ntfy인데 포장이 다름.

#### 5. 자가호스팅 설치·운영 대행

| 항목 | 내용 |
|---|---|
| 🎯 타깃 | 보안·규제로 **내부망 필수**인 금융·의료·공공·방산·대기업 |
| 💵 가격 | 초기 구축 **300~1,500만원** + 연간 유지보수 **500~2,000만원** |
| 🔧 제공 | 설계·설치·HA(PostgreSQL 복제)·SSO/LDAP·모니터링·교육·SLA |
| ⏱ 개발 | **0주** (제품 개발 없음 — 바로 영업 가능) ⭐ |
| 📈 난이도 | ⭐⭐⭐ (기술보다 영업이 어려움) |

**매력:** 제품을 만들 필요가 없음. 1인 사업으로 가장 빨리 현금이 도는 모델.

#### 6. 마켓플레이스 / 플러그인 생태계

| 제품 | 비고 |
|---|---|
| **WordPress 플러그인** | 전 세계 웹사이트 43%, 유료화 생태계 성숙 → **가장 확실** |
| Shopify 앱 / n8n·Make·Zapier 노드 / Home Assistant 통합 | |
| **MCP 서버** / Chrome 확장 / Grafana 플러그인 | |

가격: 무료 + 프로($5~29) / 개발 2~4주/개 / 난이도 ⭐⭐ / 여러 개 쌓으면 월 수백만원

#### 7. 모니터링 통합 SaaS

Uptime 모니터링 + 알림 결합 (UptimeRobot + PagerDuty의 저가 대안).
웹사이트/API/SSL/도메인만료 감시 + 온콜 로테이션 + 에스컬레이션 + 상태 페이지.
월 $9 / $29 / $99 (PagerDuty는 좌석당 $21~41). 개발 10~16주 / 난이도 ⭐⭐⭐⭐ / 경쟁 치열.

#### 8. 알림 템플릿·레시피 마켓

ntfy에 **이미 템플릿 엔진 존재** (`template/gotext`, Go template + sprig) → 업종별 템플릿 팩 판매.
"쇼핑몰 알림 템플릿 30종", "DevOps 알림 팩", "홈서버 감시 레시피 50선".
₩9,900~49,000 / 개발 **1~2주** (가장 빠름) / 난이도 ⭐ / **부업·첫걸음으로 최적**

#### 9. 하드웨어 번들

ntfy를 받아 물리적으로 반응하는 기기 (경광등·부저·E-ink).
타깃: 공장·주방·창고·서버실 — **폰을 못 보는 환경**.
ESP32 + 경광등 + ntfy 구독 펌웨어. 기기 8~30만원 + 월 구독 ₩9,900.
개발 12~20주 / 난이도 ⭐⭐⭐⭐⭐ (재고·불량·배송 리스크) / 경쟁 거의 없음.

#### 10. 오픈소스 기여 → 개인 브랜딩

⭐24,000 프로젝트에 의미 있는 기여로 커리어 자산화.
방법: ① **한국어 문서 번역**(가장 쉬운 진입) ② 버그 수정 PR ③ MCP 서버 공개 ④ WordPress/n8n 플러그인 공개 ⑤ 기술 블로그 연재.
수익: 간접 — 이직 시 연봉 1,000~3,000만원 상승, 프리랜서 단가 상승, 강연, 스폰서십.
**ROI는 사실 가장 높을 수 있음.**

---

### 12.3 전체 비교표

| # | 아이디어 | 개발기간 | 난이도 | 초기비용 | 수익규모 | 시장성 | 추천도 |
|---|---|---|---|---|---|---|---|
| 1 | 한국형 매니지드 SaaS | 4~8주 | ⭐⭐⭐ | 중 | 중~대 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 |
| 2 | **AI 에이전트 승인 플랫폼** | 8~12주 | ⭐⭐⭐⭐ | 중 | **대** | **⭐⭐⭐⭐⭐** | 🔥🔥🔥🔥🔥 |
| 3 | **교육 콘텐츠** | **2~4주** | ⭐⭐ | **0** | 소~중 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 |
| 4 | 업종 특화 SaaS | 6~10주 | ⭐⭐⭐⭐ | 중 | **대** | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 |
| 5 | **설치·운영 대행** | **0주** | ⭐⭐⭐ | **0** | 중~대 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 |
| 6 | 플러그인 생태계 | 2~4주 | ⭐⭐ | 저 | 소~중 | ⭐⭐⭐ | 🔥🔥🔥 |
| 7 | 모니터링 SaaS | 10~16주 | ⭐⭐⭐⭐ | 중 | 중~대 | ⭐⭐⭐ | 🔥🔥 |
| 8 | 템플릿 마켓 | **1~2주** | ⭐ | **0** | 소 | ⭐⭐ | 🔥🔥 |
| 9 | 하드웨어 번들 | 12~20주 | ⭐⭐⭐⭐⭐ | **고** | 중 | ⭐⭐⭐ | 🔥 |
| 10 | 오픈소스 브랜딩 | 2주~ | ⭐⭐ | **0** | 간접 | ⭐⭐⭐⭐ | 🔥🔥🔥 |

---

### 12.4 실행 로드맵

```
[0~1개월]  ③ 교육 콘텐츠 시작  +  ⑩ 한국어 문서 번역 기여
           → 비용 0원, 시장 반응 수집, 청중·신뢰 구축
           → 부수효과: ntfy를 완전히 꿰뚫게 됨

[1~3개월]  ⑧ 템플릿 팩 판매  +  ⑥ WordPress 플러그인 출시
           → 첫 매출. "진짜 돈을 내는가" 검증

[2~4개월]  ⑤ 설치·운영 대행 영업 시작
           → 제품 개발 없이 큰 금액. 현금 흐름 확보
           → 부수효과: 기업 고객의 진짜 요구사항 파악

[4~9개월]  ② AI 에이전트 승인 플랫폼  또는  ④ 업종 특화 SaaS
           → 청중 + 자금 + 고객 인사이트로 본게임
           → 유튜브 청중이 그대로 초기 사용자
```

**핵심 전략: "콘텐츠로 청중을 모으고 → 작은 제품으로 검증하고 → 큰 SaaS로 확장"**
처음부터 2번으로 뛰어들면 "만들었는데 아무도 모름" 함정에 빠진다.

---

### 12.5 리스크 체크리스트

| 리스크 | 대응 |
|---|---|
| ntfy.sh 무료 플랜과 경쟁 | **차별화 레이어**에 집중 (한국어·카톡·결제·업종특화·승인워크플로) |
| 원작자가 같은 기능 추가 | 상류 기능이 아닌 **현지화·업종·AI** 영역에 집중 |
| 상표권 | **"ntfy"를 제품명에 쓰지 말 것.** "Powered by ntfy" 수준 표기 |
| GPLv2 조항 | SaaS 운영은 안전. **수정본 바이너리 배포 시** 라이선스 검토 필수 |
| 알림 서비스 신뢰성 | 다운되면 신뢰 즉사 → **HA 구성, SLA, 상태 페이지 필수** |
| 스팸·어뷰징 악용 | `visitor.go`(rate limit), `ban/` 활용 + 자체 정책 |
| 한국 법규 | 정보통신망법, 개인정보보호법, **광고성 정보 전송 시 수신동의** 의무 확인 |

---

## 13. 최종 결론

### 이 저장소가 당신에게 주는 3가지 가치

**① 즉시 실용 — "내 자동화에 알림 날개"**
터미널에서 `curl -d "테스트" ntfy.sh/아무이름` 하면 끝.
세상에서 푸시 알림을 붙이는 **가장 빠른 방법.** FCM 인증서·APNs 키·앱 심사 전부 생략.

**② 학습 교재 — "1인 개발 Go 백엔드의 교과서"**
Pub/Sub 아키텍처 · Long-polling/SSE/WebSocket 3종 비교 · SQLite↔PostgreSQL 추상화 + 읽기 복제본 ·
Rate limiting & 어뷰징 방지 · **Stripe 결제 + 티어 요금제 전체 구현** · FCM/WebPush/APNs/SMTP/Twilio 통합 ·
Prometheus 메트릭 · goreleaser 멀티플랫폼 배포 · 압도적 테스트 커버리지(테스트가 본 코드보다 큼).

> **"오픈소스로 월 구독 수익을 만드는 구조"를 소스코드 레벨에서 볼 수 있는 몇 안 되는 저장소.**

**③ 사업 기반 — "검증된 모델 + 완성된 과금 엔진"**
Apache 2.0으로 상업적 이용 가능 + 원작자가 이미 수익화로 모델 검증 + Stripe/티어 코드 완비.

### 지금 바로 할 일 3가지

```
1. 폰에 ntfy 앱 설치 → curl 한 줄로 알림 받아보기 (5분)
2. Claude Code Stop 훅에 curl 등록 → AI 작업 완료 알림 (10분)
3. 유튜브 1편 기획 "터미널 한 줄로 내 폰에 알림 보내기" (주말)
```

---

## 부록 A. 빠른 참조 치트시트

```bash
# 가장 기본
curl -d "메시지" ntfy.sh/mytopic

# 전체 옵션
curl \
  -H "Title: 제목" \
  -H "Priority: urgent" \          # min|low|default|high|urgent (1~5)
  -H "Tags: warning,skull" \       # 이모지 태그
  -H "Click: https://example.com" \
  -H "Icon: https://example.com/icon.png" \
  -H "Attach: https://example.com/file.zip" \
  -H "Markdown: yes" \
  -H "Email: me@example.com" \
  -H "Delay: 30min" \              # 또는 "tomorrow 10am"
  -H "Actions: view, 열기, https://example.com" \
  -H "Authorization: Bearer tk_xxx" \
  -d "본문" \
  ntfy.sh/mytopic

# JSON 방식
curl ntfy.sh -d '{"topic":"mytopic","title":"제목","message":"본문","priority":4,"tags":["warning"]}'

# 구독
curl -s ntfy.sh/mytopic/json          # JSON 스트림
curl -s ntfy.sh/mytopic/sse           # SSE
curl -s ntfy.sh/mytopic/raw           # 텍스트
curl -s "ntfy.sh/mytopic/json?poll=1" # 지난 메시지만 1회 조회
curl -s "ntfy.sh/mytopic/json?since=10m"  # 10분 전부터

# CLI
ntfy publish -t "제목" -p high --tags warning mytopic "본문"
ntfy subscribe mytopic
ntfy subscribe mytopic '/path/script.sh'
ntfy serve                             # 서버 실행
```

## 부록 B. 참고 문서 위치 (저장소 내부)

| 문서 | 경로 |
|---|---|
| 시작하기 | `docs/index.md` |
| **발행 API 전체** | `docs/publish.md` |
| 템플릿 함수 | `docs/publish/template-functions.md` |
| 구독 (폰/웹/CLI/API/PWA) | `docs/subscribe/{phone,web,cli,api,pwa}.md` |
| 설치 (전 플랫폼) | `docs/install.md` |
| **서버 설정 전체** | `docs/config.md` + `server/server.yml` |
| 예제 모음 | `docs/examples.md` |
| 연동 목록 | `docs/integrations.md` |
| FAQ / 트러블슈팅 | `docs/faq.md`, `docs/troubleshooting.md` |
| 개발/빌드 | `docs/develop.md`, `docs/contributing.md` |

---

*이 문서는 `bmshin94/ntfy` 저장소 전체를 전수조사하여 작성한 한국어 분석/활용 가이드입니다.*
*원본 프로젝트: https://github.com/binwiederhier/ntfy (Apache 2.0 + GPLv2, © Philipp C. Heckel)*
