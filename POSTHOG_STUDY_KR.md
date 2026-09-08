# 🦔 PostHog 분석 & 수익화 스터디 노트 (한국어)

> 이 레포가 뭔지, 어떻게 쓰는지, 그리고 이걸로 뭘 해볼 수 있는지 정리한 문서입니다.
> 작성일: 2026-09-08

## 🔗 관련 링크

| 구분 | 주소 |
|---|---|
| **내 저장소 (fork)** | https://github.com/bmshin94/posthog |
| **원본 저장소 (upstream)** | https://github.com/PostHog/posthog |
| 공식 사이트 | https://posthog.com |
| 공식 문서 | https://posthog.com/docs |
| 무료 가입 (US) | https://us.posthog.com/signup |
| 무료 가입 (EU) | https://eu.posthog.com/signup |
| 셀프호스팅 문서 | https://posthog.com/docs/self-host |
| MCP 연동 | https://posthog.com/mcp |
| 기여 가이드 | https://github.com/PostHog/posthog/blob/master/CONTRIBUTING.md |

---

## 1. 📌 PostHog가 뭐야?

**한 문장 요약: "내 웹사이트/앱에 온 사람들이 뭘 하는지 다 보여주는 오픈소스 도구 모음"**

원래는 따로따로 돈 내고 사야 하는 도구들을 하나로 묶어놨습니다.

| PostHog 기능 | 대체하는 유명 서비스 |
|---|---|
| Product / Web Analytics | Google Analytics, Amplitude, Mixpanel |
| Session Replay (화면 녹화 재생) | Hotjar, FullStory |
| Feature Flags (기능 켜고 끄기) | LaunchDarkly |
| Experiments (A/B 테스트) | Optimizely |
| Error Tracking | Sentry |
| Surveys (설문) | Typeform |
| Data Warehouse / CDP | Segment, Fivetran |
| LLM Observability | LangSmith, Helicone |

여기에 요즘 밀고 있는 **Self-driving mode**가 붙습니다.
에러 / rage click 같은 신호를 AI가 분석해서 **리포트와 PR까지 만들어주는** 기능입니다.

### 💡 킬러 기능: Autocapture
코드를 따로 안 짜도 **모든 클릭, 페이지 이동, 폼 제출**을 자동으로 기록해줍니다.

---

## 2. 📂 저장소 구조 분석

**규모: 파일 48,870개 / 726MB** — 회사 제품 전체가 통째로 들어있는 초대형 모노레포입니다.

```
posthog/       → Python/Django 백엔드 (API, 인증, 쿼리엔진)   .py   20,218개
frontend/      → React + TypeScript UI (Kea 상태관리)        .tsx   7,681개
products/      → 기능별 모듈 97개
                 (experiments, surveys, error_tracking,
                  llm_analytics, feature_flags, cdp ...)
rust/          → 고속 데이터 수집기                          .rs    1,472개
                 (capture, ingestion, feature-flags)
nodejs/        → 이벤트 처리 파이프라인 (plugin-server, CDP)
livestream/    → Go 기반 실시간 이벤트 스트림
common/hogql_parser → HogQL (자체 SQL 방언, ANTLR 문법 직접 정의)
docker-compose.*    → ClickHouse + Postgres + Kafka + Redis
```

### 아키텍처 포인트
- **용도별로 언어를 나눠 씀** — 트래픽이 몰리는 수집 구간은 Rust, 비즈니스 로직은 Python, UI는 React, 실시간은 Go.
  폴리글랏 아키텍처의 교과서적 사례.
- **ClickHouse**가 메인 분석 DB (수십억 이벤트 집계용), Postgres는 메타데이터용.
- `products/` 폴더가 기능당 1개씩 딱 떨어져 있어 구조 파악이 쉬움.

### 💎 숨은 보석: `.claude/` 폴더
```
.claude/
├── skills/   → 93개 (django-migrations, writing-tests,
│                debugging-ci-failures, clickhouse-migrations ...)
├── agents/   → 전문 서브에이전트 (code-reviewer, test-writer ...)
├── hooks/    → 세션 시작 자동 세팅
└── rules/    → 코딩 컨벤션
AGENTS.md      → 46KB 짜리 AI 개발 가이드
```
실무 팀이 AI 에이전트를 어떻게 굴리는지 볼 수 있는 레퍼런스. 이런 규모로 공개된 사례는 드묾.

### 📋 라이선스
- 대부분 **MIT** → 상업적 이용 가능
- 단, **`ee/` 폴더는 상용 라이선스** → 그대로 재판매 불가
- PostHog **상표**는 사용 불가

### ℹ️ 내 fork 상태
`bmshin94/posthog`는 `PostHog/posthog`의 fork이며, shallow clone(커밋 50개만) 상태입니다.
원본과의 차이는 `CLAUDE.md` 하나뿐입니다.

---

## 3. 🛠 설치 및 사용법

목적에 따라 방법이 완전히 다릅니다.

```
"내 사이트 분석하고 싶다"      → 방법 1 ✅ (5분)
"데이터를 내 서버에 두고 싶다"  → 방법 2   (30분 + 서버비)
"코드를 뜯어보고 고치고 싶다"   → 방법 3   (반나절 + 고사양 PC)
```

### 🥇 방법 1. 클라우드 (추천, 설치 불필요)

1. https://us.posthog.com/signup 가입 (카드 등록 불필요)
   - 월 이벤트 100만 건 / 세션 녹화 5,000개 무료
2. 발급받은 스니펫을 `<head>`에 붙여넣기

**React / Next.js**
```bash
npm install posthog-js
```
```javascript
import posthog from 'posthog-js'

posthog.init('phc_YOUR_KEY', {
  api_host: 'https://us.i.posthog.com',
})
```

**PHP (공식 SDK 있음)**
```bash
composer require posthog/posthog-php
```
```php
PostHog\PostHog::init('phc_YOUR_KEY', ['host' => 'https://us.i.posthog.com']);

PostHog\PostHog::capture([
    'distinctId' => $customerId,
    'event'      => '장바구니_이탈',
    'properties' => ['금액' => 45000, '상품' => '원피스'],
]);
```

**Python**
```bash
pip install posthog
```
```python
from posthog import Posthog
posthog = Posthog('phc_YOUR_KEY', host='https://us.i.posthog.com')
```

3. 확인: 대시보드 **Activity** 메뉴에서 실시간 이벤트가 뜨면 성공

**직접 이벤트 남기기**
```javascript
posthog.capture('결제_완료', { 금액: 15000, 상품: '아메리카노' })
```

**Feature flag 사용 (배포 없이 기능 on/off)**
```javascript
if (posthog.isFeatureEnabled('new-design')) {
  // 새 디자인 노출
}
```

### 🥈 방법 2. 셀프호스팅 (hobby deploy)

**요구사항**: 리눅스 서버 / **RAM 8GB 이상** / **실제 도메인(A 레코드 필요, IP 불가)** / sudo

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/posthog/posthog/HEAD/bin/deploy-hobby)"
```

실행하면 버전 → 도메인 → sudo 비밀번호를 물어보고, Docker로 전체 스택을 띄운 뒤 HTTPS 인증서까지 자동 설정합니다.
(15~20분 소요)

> ⚠️ 공식 고객지원 없음. 월 10만 이벤트 수준까지만 권장.

### 🥉 방법 3. 로컬 개발환경 (기여자용)

**요구사항**: RAM 16GB 권장 / 디스크 20GB+ / Docker 실행 중

```bash
# 0) 원본 클론 (fork는 shallow라 개발에 불편)
git clone https://github.com/PostHog/posthog.git
cd posthog

# 1) 개발 도구 (Flox가 Python 3.13.13, Node v24.13.0 등 자동 설치)
brew install flox
flox activate
#    수동으로 할 경우: uv sync && pnpm install

# 2) 실행 — 이거 하나면 끝
hogli start
#    → Docker 기동 + 마이그레이션 + 전체 서비스 실행
#    → http://localhost:8010 자동 오픈

# 3) 테스트 데이터
hogli dev:demo-data
```

**자주 쓰는 명령어**
```bash
hogli quickstart        # 시작 가이드
hogli --help            # 전체 명령어
hogli dev:setup         # 실행할 서비스 선택 (가볍게 돌릴 때 필수)
hogli dev:reset         # 전체 초기화
hogli stop              # 중지
hogli format            # 코드 포맷팅
hogli lint              # 코드 검사
hogli test:python <경로> # 파이썬 테스트
hogli test:js <경로>     # JS 테스트
hogli db:migrate        # DB 마이그레이션
```

> 💡 전부 띄우면 무거우므로 `hogli dev:setup`으로 필요한 서비스만 켜는 걸 권장.

---

## 4. 🕵️ 발견한 시장 빈틈 (중요!)

레포를 직접 뒤져서 확인한 사실입니다.

### 외부 서비스 커넥터: **1,343개** 보유
`products/warehouse_sources/backend/temporal/data_imports/sources/`
Stripe, Shopify, Salesforce, HubSpot, Notion, Linear 등 전 세계 서비스가 다 있음.

### 그런데 한국 서비스는 **0개**
```
kakao        ❌     naver      ❌     toss         ❌
cafe24       ❌     imweb      ❌     channeltalk  ❌
coupang      ❌     포트원      ❌     이니시스      ❌
```

### 내보내기(destination)도 **35개뿐**, 한국 서비스 없음
`nodejs/src/cdp/templates/_destinations/`
```
slack, twilio, whatsapp, hubspot, meta_ads, google_ads,
linear, github, gitlab, webhook, email, push ... (35개)
→ 카카오 알림톡 ❌  네이버웍스 ❌  채널톡 ❌
```

**결론: 한국 시장이 통째로 비어 있음.**

---

## 5. 💰 수익화 아이디어 (현실성 순)

### 🥇 ① PostHog 한국 도입 / 셀프호스팅 구축 대행 ⭐⭐⭐⭐⭐
- **왜 되나**: 데이터 해외 반출이 어려운 곳이 많음(금융·의료·공공·게임).
  GA4는 데이터가 구글로 가고 Amplitude는 비쌈. PostHog는 자체 서버 설치 가능.
  그런데 셀프호스팅이 어려움 → **그래서 대행이 돈이 됨**
- **수익 모델**
  | 항목 | 가격대 |
  |---|---|
  | 초기 구축 (서버 세팅 + 이관) | 300~800만원 |
  | 이벤트 설계 컨설팅 | 200~500만원 |
  | 월 운영/모니터링 | 월 50~150만원 |
- **필요한 것**: 레퍼런스 1개. 첫 고객만 뚫으면 그 다음부터 굴러감
- **주의**: 기술보다 **영업이 8할**

### 🥈 ② 한국 커넥터 오픈소스 기여 ⭐⭐⭐⭐
직접 수익은 없지만 **영업 자산**이 됨.
- 만들 후보: 카카오 알림톡(destination), 채널톡 / 카페24 / 아임웹 / 포트원(source)
- 레포에 가이드 있음:
  - `.claude/skills/implementing-warehouse-sources/`
  - `.claude/skills/documenting-warehouse-sources/`
- 경로: 오픈소스 머지 → "PostHog 컨트리뷰터" 타이틀 → ①번 영업력 상승

### 🥉 ③ 쇼핑몰용 마이크로 SaaS ⭐⭐⭐⭐ (스택 궁합 최고)
- **컨셉**: 카페24 / 아임웹 / 식스샵 사장님은 개발을 못 함
  → **버튼 하나로 설치되는 앱**을 만들어 판매
- **동작**: 앱스토어 설치 → 자동 연동 → 매일 아침 **카톡으로 리포트**
  > "어제 방문 342명 / 장바구니 담고 안 산 사람 28명 / 이탈 1위: 배송비 페이지"
- **가격**: 월 19,900원 → 100명이면 월 200만원
- **핵심**: 사장님은 "PostHog"를 몰라도 됨. **"카톡으로 매출 알려주는 앱"**으로 팔면 됨

### ④ AI 분석 비서 (MCP 활용) ⭐⭐⭐
PostHog MCP로 AI가 데이터를 직접 조회 → 슬랙/카톡 봇으로 질의응답.
- **주의**: PostHog가 Self-driving mode로 직접 미는 영역이라 정면승부는 위험.
  **한국어 + 슬랙/카톡 특화**로 좁혀야 승산 있음

### ⑤ 한국어 콘텐츠 / 교육 ⭐⭐⭐
한국어 자료가 거의 없어 선점 효과가 큼. 블로그/유튜브/전자책/강의.
진짜 목적은 **①번 컨설팅으로 가는 깔때기**.

### ⑥ AI 개발 세팅 컨설팅 ⭐⭐⭐⭐ (숨은 보석)
`.claude/` 스킬 93개 + `AGENTS.md` 46KB = 실무 AI 에이전트 운영 노하우 교재.
- AI 개발환경 세팅 컨설팅 300~500만원 / 사내 교육 100~200만원 / 템플릿 판매
- PostHog와 무관하게도 수익화 가능

### 🎯 추천 로드맵
```
[1~2개월]  ⑤ 한국어 콘텐츠 + ② 커넥터 기여   ← 신뢰 자산 쌓기
[3개월~]   ① 컨설팅으로 현금화
[6개월~]   ③ 마이크로 SaaS로 자동 수익
```

### ⚠️ 하지 말아야 할 것
1. PostHog를 그대로 복사해 리브랜딩 재판매 → `ee/` 라이선스 + 상표 문제 + 승산 없음
2. 얇은 껍데기(thin wrapper) 제품 → 다음 업데이트에 소멸. **"한국"이라는 해자**가 있어야 함

---

## 6. ⚛️🐘 React / PHP로 가능한가?

**결론: 두 가지 세계가 있고, 돈 되는 건 대부분 🅱️입니다.**

```
🅰️ PostHog "안쪽"을 고치는 일    → React/PHP ❌
🅱️ PostHog "위에" 만드는 일      → React/PHP ✅
```

| 아이디어 | React | PHP | 비고 |
|---|:---:|:---:|---|
| ① 도입/구축 컨설팅 | — | — | 언어 무관, Docker/리눅스 필요 |
| ② 커넥터 기여 | ❌ | ❌ | **Python 필요** (내보내기만 Hog 가능) |
| ③ 쇼핑몰 SaaS | ✅ | ✅ | **궁합 최고** |
| ④ AI 분석 봇 | ✅ | ✅ | API 호출만 하면 됨 |
| ⑤ 콘텐츠/교육 | — | — | 언어 무관 |
| ⑥ AI 세팅 컨설팅 | — | — | 언어 무관 |

### ❌ 커넥터(source)는 Python
```
sources/stripe/
├── source.py
├── stripe.py
├── settings.py
└── tests/test_stripe_source.py     ← 전부 .py
```

### 🤏 내보내기(destination)는 TypeScript + Hog — 배울 만함
`nodejs/src/cdp/templates/_destinations/slack/slack.template.ts` 실제 코드:
```javascript
let body := {
  'channel': inputs.channel,
  'text': inputs.text
};

let res := fetch('https://slack.com/api/chat.postMessage', {
  'body': body,
  'method': 'POST',
  'headers': { 'Authorization': f'Bearer {inputs.token}' }
});

if (res.status != 200) {
  throw Error(f'Failed: {res.status}');
}
```
JavaScript와 거의 동일 (`=` 대신 `:=`). 슬랙 템플릿 전체가 60줄 수준.
→ **카카오 알림톡 destination은 React 경험자도 충분히 만들 수 있음.**

### ✅ ③번 마이크로 SaaS 아키텍처 (React + PHP)
```
┌─────────────────────────────────┐
│  ⚛️ React — 사장님이 보는 화면     │
│  · 설치 마법사 · 대시보드 · 설정   │
└──────────────┬──────────────────┘
               │ REST API
┌──────────────▼──────────────────┐
│  🐘 PHP (Laravel) — 두뇌         │
│  · 카페24 OAuth 로그인            │
│  · 주문 웹훅 수신 → 이벤트 전송     │
│  · 매일 새벽 집계 → 알림톡 발송     │
└───┬──────────────────────┬──────┘
    │                      │
┌───▼────────┐      ┌──────▼────────┐
│ 🦔 PostHog │      │ 💬 알림톡      │
│  데이터저장 │      │  솔라피/알리고  │
│   · 분석   │      └───────────────┘
└────────────┘
```

**장점: 별도 DB 서버가 불필요** (데이터는 PostHog가 저장) → 서버비 최소화.
무거운 ClickHouse / Kafka는 신경 쓸 필요 없음.

**PHP에서 데이터 조회 (Query API + HogQL)**
```php
$response = Http::withToken($apiKey)->post(
    "https://us.posthog.com/api/projects/{$projectId}/query/",
    ['query' => [
        'kind'  => 'HogQLQuery',
        'query' => "SELECT count() FROM events
                    WHERE event = '장바구니_이탈'
                      AND timestamp > now() - INTERVAL 1 DAY"
    ]]
);
```

### ⚠️ 미리 알아둘 함정
1. **카카오 알림톡은 직접 발송 불가** — 공식 대행사를 거쳐야 함 (솔라피/알리고 추천, 둘 다 PHP 예제 있음).
   템플릿 사전 승인 1~3일 소요.
2. **카페24 앱 심사 존재** — 처음엔 심사 없이 개별 쇼핑몰에 직접 설치해주는 방식으로 3~5곳 검증 후 앱스토어 등록이 안전.

---

## 7. 🎬 첫 주 실행 계획 (③번 기준)

```
Day 1-2   PostHog 가입 → PHP SDK로 이벤트 1건 전송 성공
Day 3-4   카페24 개발자센터 가입 → OAuth 연동 테스트 (샘플 쇼핑몰 무료 생성 가능)
Day 5-6   Query API로 "어제 방문자 수" 조회 → React로 그래프 1개 렌더
Day 7     솔라피 가입 → 내 폰으로 알림톡 1건 발송 성공
```
이 7일이면 MVP 뼈대 완성.

---

## 8. ✅ 다음에 정할 것

- [ ] PHP는 순수 PHP인지 Laravel인지
- [ ] 타겟이 카페24 / 아임웹 / 자체 사이트 중 어디인지
- [ ] React는 Next.js인지 Vite인지
- [ ] PostHog 공식 파트너 프로그램 존재 여부 확인 (있으면 ①번이 훨씬 수월)

---

_이 문서는 저장소 실제 내용을 직접 확인해서 작성되었습니다._
_원본: https://github.com/PostHog/posthog · 내 fork: https://github.com/bmshin94/posthog_
