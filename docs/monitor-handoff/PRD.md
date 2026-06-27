# PRD — 게시판 작성자 모니터링 & 리포트 시스템

> 코드네임(임시): **board-monitor**
> 참고 레퍼런스: `Supaper/av-board` (Firebase + GitHub Pages + Google 로그인 보안 스택)
> 대체 대상: 기존 Google Sheets + Apps Script + Gmail 모니터링 워크플로

---

## 1. 목적 (Why)

특정 게시판(`thelifechurch.kr`)을 주기적으로 모니터링하여,
**지정된 작성자들의 신규 게시글을 카테고리별로 자동 취합**하고,

- **일별** — 신규 게시물 알림(Gmail)
- **월별** — 작성자/카테고리별 통계 리포트(Gmail)
- **(장기)** — 취합 결과를 **한글 문서(.hwpx/.hwp)** 양식에 자동 채워 문서화

하는 것을 목표로 한다.

현재는 Google Sheets + Apps Script로 운영 중이며, 이를 **av-board와 동일한 스택
(Firebase + GitHub Pages + Google 로그인)** 으로 이관하여 데이터 영속화·웹 대시보드·
확장성을 확보한다.

---

## 2. 현재 상태 (As-Is)

기존 Apps Script(`monitorDailyCollectionOnly`, `sendMonthlyQTReport`, `resetMemory`)의 동작:

| 항목 | 내용 |
|---|---|
| 데이터 소스 | `http://www.thelifechurch.kr/main/sub.html?boardID=www56&Mode=list&keyfield=name&key={작성자명}` |
| 검색 방식 | 게시판의 **작성자명(`keyfield=name`) 검색** URL에 이름을 URL-encode해서 요청 |
| 신규 감지 | `PropertiesService`(스크립트 속성)에 작성자별 "마지막 수집 기준점" 저장 → 다음 실행 시 그보다 새 글만 수집 |
| 저장 | 작성자 이름과 동일한 시트(`getSheetByName(name)`)에 행 단위 기록 |
| 일별 알림 | 신규 글이 1건 이상이면 `MailApp.sendEmail`로 HTML 목록 메일 발송 (수신: 실행 계정 본인) |
| 월별 통계 | 지난달 데이터에서 제목에 **`큐티나눔`** 포함 글을 작성자별로 카운트 → 완주 횟수/달성률(%) 표를 메일 발송 |
| 기준점 초기화 | `resetMemory()` = 모든 스크립트 속성 삭제 |
| 시간대 | `Session.getScriptTimeZone()` (KST 기준), 날짜 포맷 `yyyy-MM-dd` |
| 대상 그룹 | 일별/월별이 **서로 다른 작성자 목록 + 서로 다른 스프레드시트**를 사용 (그룹이 둘 이상 존재) |

> ⚠️ **중요**: 사용자가 전달한 Apps Script는 게시판 HTML을 파싱하는 핵심 부분이
> `코드` placeholder로 생략돼 있다. **신규 세션은 사용자에게 원본 전체 Apps Script
> (HTML 파싱/정규식 부분 포함)를 요청해서, 실제 게시판의 행/제목링크/날짜 추출
> 로직을 그대로 이식해야 한다.** (게시판이 봇 직접 접근을 403으로 막으므로
> 원본 파싱 로직이 가장 신뢰할 수 있는 명세다.)

### As-Is의 한계
- 데이터가 Google Sheets에 분산(작성자별 시트) → 통합 조회/시각화 어려움
- 기준점이 Apps Script 속성에만 존재 → 이력 추적/감사 어려움
- 카테고리 분류가 코드에 하드코딩(`큐티나눔` 등)
- 웹 대시보드 없음, 문서화(HWP) 자동화 없음

---

## 3. 목표 상태 (To-Be) — 아키텍처

GitHub Pages는 정적 호스팅이라 서버 크롤링이 불가능하다. 그래서 역할을 분리한다
(av-board가 클라이언트 전용인 것과 동일한 제약).

```
┌─────────────────────────┐   크롤링/파싱/신규감지/분류   ┌──────────────────┐
│ GitHub Actions (cron)   │ ───────────────────────────▶ │ Firebase RTDB    │
│  crawler/collect.mjs    │                              │ (posts/stats/…)  │
│  - 작성자별 검색 URL fetch │                              └────────┬─────────┘
│  - EUC-KR 디코드 → 파싱   │   변동/신규 → Gmail              읽기   │ (read-only)
│  - 카테고리 분류          │ ──────────────────┐                    ▼
│  - baseline 비교(신규만)  │                   ▼            ┌──────────────────┐
└─────────────────────────┘            ┌──────────────┐     │ GitHub Pages     │
                                        │ Gmail (SMTP) │     │ index.html 대시보드 │
   일별 알림 / 월별 통계 메일  ◀──────────│ nodemailer   │     │ - Google 로그인    │
                                        └──────────────┘     │ - 허용 계정만 접근  │
                                                             └──────────────────┘
```

- **수집기(crawler)** = GitHub Actions 스케줄 워크플로 (Node.js). Apps Script 트리거를 대체.
- **저장소** = Firebase Realtime Database (단일 진실 공급원).
- **리포트** = GitHub Actions에서 `nodemailer` + Gmail SMTP(앱 비밀번호).
- **대시보드** = `index.html` 정적 페이지 (av-board와 동일 패턴), Firebase read-only.
- **보안** = Google 로그인 + 허용 이메일 화이트리스트 + Firebase Security Rules + GitHub Secrets.

---

## 4. 기능 요구사항

### Phase 1 — MVP (기존 Apps Script 기능 동등 이관) 🎯 최우선
- **F1. 작성자별 크롤링**: 설정된 작성자 목록에 대해 검색 URL을 순회 요청, 게시판 목록 파싱
  (제목, 게시글 URL, 작성일, 작성자). 한글 인코딩(**EUC-KR 추정**) 디코딩 처리.
- **F2. 신규 감지**: 작성자별 baseline(마지막 수집 글 식별자)을 Firebase에 저장,
  다음 실행 시 그보다 새 글만 신규로 처리.
- **F3. 카테고리 분류**: 제목 키워드 매핑으로 카테고리 부여(예: 제목에 `큐티나눔` → `큐티` 카테고리).
  키워드→카테고리 매핑은 **설정/Firebase에서 관리**(하드코딩 금지).
- **F4. Firebase 적재**: 신규 글을 `posts/{작성자}/{postId}`에 기록.
- **F5. 일별 알림 메일**: 당일 신규 글이 있으면 작성자/카테고리별로 묶은 HTML 메일을
  지정 수신자에게 발송. (Apps Script의 일별 메일 대체)
- **F6. 월별 통계 메일**: 지난달 데이터로 작성자×카테고리 횟수/달성률 표 생성 → 메일 발송.
  (Apps Script의 `sendMonthlyQTReport` 대체, 카테고리 일반화)
- **F7. baseline 리셋**: 기준점 초기화 수단(`resetMemory` 대체 — 수동 워크플로 또는 스크립트).

### Phase 2 — 웹 대시보드 (av-board 패턴)
- **F8. Google 로그인** (`signInWithPopup`) + 허용 계정만 접근.
- **F9. 현황 보드**: 작성자별 / 카테고리별 / 기간별 게시글 목록·필터.
- **F10. 통계 대시보드**: 월별 추이, 카테고리 도넛 차트, 작성자별 달성률 카드 (av-board 대시보드 차용).
- **F11. 설정 화면**: 작성자 목록·카테고리 키워드·수신자·크롤 대상 게시판을 화면에서 관리.

### Phase 3 — 한글 문서 자동화 (장기)
- **F12. HWPX 자동 생성**: 월별 취합 결과를 **지정된 한글 양식**에 채워 `.hwpx` 파일 생성.
- **F13. 산출물 전달**: 생성 문서를 메일 첨부 또는 다운로드 제공.

> HWP/HWPX 기술 메모: `.hwpx`는 ZIP+XML(OWPML) 구조라 **템플릿 XML 치환 방식으로 생성 가능**.
> `.hwp`(바이너리 OLE 복합문서)는 난이도가 매우 높음 → **hwpx 우선 권장**, 필요 시 한컴
> 변환 도구로 hwp 변환. Phase 3 착수 전에 "지정된 양식" 샘플 파일을 확보해야 함.

---

## 5. 비기능 요구사항

- **보안 (3계층)**
  1. 대시보드: Google 로그인 + 허용 Gmail 화이트리스트(또는 도메인 제한).
  2. Firebase Security Rules: `posts/stats/...`는 로그인+허용 계정만 `.read`,
     클라이언트 `.write` 금지. 쓰기는 크롤러(Admin SDK)만.
  3. 크리덴셜은 전부 **GitHub Secrets**: `FIREBASE_SERVICE_ACCOUNT`, `GMAIL_USER`,
     `GMAIL_APP_PASSWORD`. 코드/클라이언트에 절대 임베드 금지.
- **정중한 크롤링**: 요청 간 지연(throttle), User-Agent 명시, 실패 재시도(backoff).
  대상 사이트 `robots.txt`/이용약관 준수. (현재 운영 중인 수집과 동일 범위 유지)
- **시간대**: 모든 일/월 경계는 **KST(Asia/Seoul)** 기준.
- **멱등성**: 같은 글을 중복 수집/중복 알림하지 않음(postId 해시 + baseline).
- **비용**: Firebase Spark(무료) + GitHub Actions 무료 한도 내 동작 목표.

---

## 6. 데이터 모델 (Firebase RTDB)

```
config/
  boards/{boardId}/         { name, baseUrl, keyfield, domain, encoding:"euc-kr" }
  authors/{authorId}/       { name, group, active }          # 일별/월별 그룹 구분
  categories/{categoryId}/  { label, keywords:[...] }         # 예: 큐티 → ["큐티나눔"]
  recipients/{type}/        { daily:[...], monthly:[...] }     # 메일 수신자
posts/
  {authorName}/
    {postId}/               { title, url, postedAt, category, collectedAt, boardId }
baseline/
  {authorName}/             { lastPostId, lastPostedAt }       # 신규 감지 기준점
stats/
  monthly/{yyyy-mm}/
    {authorName}/           { categoryCounts:{...}, total, rate }
runs/
  {ts}/                     { newCount, durationMs, status, error? }   # 실행 로그/감사
profiles/{uid}/             { name, email, admin }             # av-board와 동일
allowedEmails/{sanitized}/  true                               # 접근 화이트리스트
```

---

## 7. 기술 스택

| 역할 | 기술 |
|---|---|
| 수집기 | Node.js (ESM) + `cheerio`(정적 HTML 파싱) + `iconv-lite`(EUC-KR 디코딩) + `firebase-admin` |
| 스케줄 | GitHub Actions `schedule: cron` + `workflow_dispatch`(수동 실행) |
| 메일 | `nodemailer` + Gmail SMTP(앱 비밀번호) |
| DB | Firebase Realtime Database (modular SDK v10.12.0, 대시보드 측) |
| 인증 | Firebase Auth — Google `signInWithPopup` |
| 대시보드 | Vanilla HTML/CSS/JS 단일 `index.html` (빌드 없음, av-board 패턴) |
| 호스팅 | GitHub Pages |
| 문서화(P3) | hwpx(OWPML) 템플릿 치환 |

> 게시판이 **JS 렌더링 없이 서버에서 HTML을 그리는 전통적 게시판**으로 보이므로
> `cheerio`로 충분할 가능성이 높다. 만약 JS 렌더링이 필요하면 `playwright`로 전환.
> (이 환경엔 Chromium이 `/opt/pw-browsers`에 사전 설치돼 있음.)

---

## 8. 마일스톤

1. **M1 — 스캐폴딩**: 레포 구조 + `CLAUDE.md` + `README` + Firebase/Secrets 셋업 가이드.
2. **M2 — 수집기 MVP**: 크롤→파싱→신규감지→Firebase 적재 (단일 작성자 검증 후 전체).
3. **M3 — 일별 알림 메일** (Gmail) 동작.
4. **M4 — 월별 통계 메일** 동작 → **기존 Apps Script 대체 완료**.
5. **M5 — 대시보드**(로그인+현황+통계 차트) 배포.
6. **M6 — 설정 화면**(작성자/카테고리/수신자 관리).
7. **M7 — HWPX 문서 자동화**(양식 확보 후 착수).

---

## 9. 오픈 이슈 / 사용자 확인 필요

- [ ] **원본 Apps Script 전체 코드**(HTML 파싱/정규식 부분 포함) 제공 → F1 파서 이식의 근거.
- [ ] 게시판 페이지 **실제 charset**(EUC-KR 확정 여부).
- [ ] 일별/월별 **작성자 목록과 그룹 정의**(현재 두 함수의 명단이 다름) 정리.
- [ ] 카테고리 정의(키워드→카테고리 매핑) 전체 목록.
- [ ] 리포트 **수신자 이메일** 목록(일별/월별 각각).
- [ ] 크롤 **주기**(cron) — 예: 매일 1회 vs 시간별.
- [ ] 대시보드 **접근 허용 계정** 목록.
- [ ] (P3) **지정된 한글 양식** 샘플 `.hwpx`/`.hwp` 파일.
- [ ] 새 레포 vs 기존 레포 — 신규 레포 권장(목적·스택이 av-board와 다름).
