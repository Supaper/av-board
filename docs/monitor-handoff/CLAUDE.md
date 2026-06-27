# Board Monitor — Project Context for Claude Code

> 이 파일은 **신규 레포(board-monitor)** 의 루트에 둘 `CLAUDE.md` 초안이다.
> `av-board`의 CLAUDE.md 구조를 그대로 차용했다. 실제 값(레포명/브랜치/Firebase
> 프로젝트/도메인)은 셋업하면서 확정해 채운다.

## ⚠️ Claude Code 필수 행동 규칙

> 매 세션 시작마다 읽고, 기능 작업이 끝날 때마다 반드시 실행한다.

### 기능 구현 완료 즉시 (선언 전에) 해야 할 일
기능 작업이 끝났다고 말하기 **전에** 아래를 직접 실행한다:

1. **`CLAUDE.md`** (이 파일)
   - `## 완료된 기능` 섹션에 새 항목 추가
   - `## 다음 개발 우선순위 (Backlog)` 에서 해당 항목 제거 및 번호 재정렬
2. **`git add` → `git commit` → `git push`**
   - 배포 브랜치(GitHub Pages 원본)에 push 즉시 반영

---

## 프로젝트 개요

특정 게시판(`thelifechurch.kr`)을 모니터링하여 **지정 작성자의 신규 글을 카테고리별로
취합**하고, **일별 알림 / 월별 통계**를 Gmail로 보고하는 시스템.
장기적으로 취합 결과를 **한글(.hwpx/.hwp) 양식**에 자동으로 채워 문서화한다.

기존 Google Sheets + Apps Script 워크플로를 **av-board 스택(Firebase + GitHub Pages +
Google 로그인)** 으로 이관한 프로젝트다.

**참고 레퍼런스:** `Supaper/av-board`
**상세 명세:** `docs/PRD.md`

---

## 아키텍처 핵심 규칙

### 역할 분리 (GitHub Pages는 정적 → 서버 크롤링 불가)
```
GitHub Actions(cron)  →  Firebase RTDB  →  GitHub Pages 대시보드
   crawler/                (저장소)           index.html (정적, Google 로그인)
   collect.mjs → 크롤·파싱·신규감지·분류·적재
   report.mjs  → 일별/월별 Gmail 발송 (nodemailer + Gmail SMTP)
```

- **크롤링·쓰기는 GitHub Actions(Node + firebase-admin)만** 수행 — 서비스 계정은 GitHub Secret.
- **대시보드(index.html)는 read-only** — av-board와 동일하게 클라이언트에서 Firebase에 직접 붙음.
- 단일 파일 구조: 대시보드는 `index.html` 한 파일에 `<style>` + `<script type="module">`.
- 외부 라이브러리는 CDN, 빌드 도구 없음.

### Firebase 데이터 구조
```
config/{boards,authors,categories,recipients}
posts/{authorName}/{postId}/   title, url, postedAt, category, collectedAt
baseline/{authorName}/         lastPostId, lastPostedAt
stats/monthly/{yyyy-mm}/{authorName}/  categoryCounts, total, rate
runs/{ts}/                     newCount, status, error?
profiles/{uid}/                name, email, admin
allowedEmails/{sanitized}      true
```

### 신규 감지 / 카테고리 (Apps Script 이관 규칙)
- Apps Script의 `PropertiesService` 기준점 → Firebase `baseline/{author}` 로 이관.
- 검색 URL 패턴: `{baseUrl}?boardID=...&Mode=list&keyfield=name&key={encodeURIComponent(작성자)}`.
- 한글 페이지 인코딩은 **EUC-KR 추정** → `iconv-lite`로 디코딩 후 `cheerio` 파싱.
- 카테고리는 제목 키워드 매핑(`categories/{id}.keywords`)으로 결정 — 하드코딩 금지.
  (예: 제목에 `큐티나눔` 포함 → `큐티` 카테고리)
- 시간대는 **KST(Asia/Seoul)** 고정.

### 보안 (av-board와 동일 철학)
- Firebase config 키는 클라이언트에 임베드(공개 키, 비밀 아님).
- 진짜 보안 = **Google 로그인 + `allowedEmails` 화이트리스트 + Security Rules**.
- 편집/관리 권한: `profiles/{uid}/admin: true` (Firebase 콘솔 수동 설정).
- 크리덴셜(`FIREBASE_SERVICE_ACCOUNT`, `GMAIL_USER`, `GMAIL_APP_PASSWORD`)은 **GitHub Secrets만**.

### window.* 온클릭 패턴 (대시보드)
- 인라인 `onclick="fn()"`은 `window.fn = ...`로 노출해야 동작 (모듈 스코프 직접 접근 불가).

---

## 레포 구조
```
index.html                    # 대시보드 (정적, GitHub Pages)
crawler/
  collect.mjs                 # 크롤·파싱·신규감지·분류·Firebase 적재
  report.mjs                  # 일별/월별 Gmail 리포트
  lib/parse.mjs               # 게시판 HTML 파서 (원본 Apps Script 로직 이식)
  package.json
.github/workflows/
  daily.yml                   # cron: 수집 + 일별 알림
  monthly.yml                 # cron: 월별 통계 리포트
firebase-rules.json           # Security Rules
docs/PRD.md                   # 제품 요구사항
README.md                     # 셋업 가이드
CLAUDE.md                     # (이 파일)
```

---

## 완료된 기능
- (아직 없음 — M1 스캐폴딩부터 시작)

## 다음 개발 우선순위 (Backlog)
1. **M1 스캐폴딩** — 레포 구조 + Firebase/Secrets/Pages 셋업 가이드
2. **M2 수집기 MVP** — 크롤→파싱→신규감지→적재 (원본 Apps Script 파서 이식)
3. **M3 일별 알림 메일**
4. **M4 월별 통계 메일** (= 기존 Apps Script 대체 완료)
5. **M5 대시보드** — Google 로그인 + 현황 + 통계 차트
6. **M6 설정 화면** — 작성자/카테고리/수신자 관리
7. **M7 HWPX 문서 자동화** — 지정 양식 채우기 (양식 샘플 확보 후)

---

## 세션 시작 시 확인
```bash
git fetch origin && git status && git log --oneline -5
```
- 배포 브랜치에서 작업 중인지 확인, 작업 후 push까지 완료해야 Pages 반영.

## 커밋 컨벤션
```
feat: 새 기능   fix: 버그 수정   style: UI 변경   refactor: 정리   chore: 설정/빌드
```
