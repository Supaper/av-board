# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## ⚠️ Claude Code 필수 행동 규칙

기능 작업이 끝났다고 말하기 **전에** 아래를 직접 실행한다:

1. **`CLAUDE.md`** — `## 완료된 기능` 추가, `## Backlog` 에서 해당 항목 제거
2. **`CHANGELOG.md`** — 아래 버전 규칙에 따라 버전 올리고 변경 내용 기록
3. **`README.md`** — 기능 소개·파일 구조·트러블슈팅 중 바뀐 내용 있으면 반영
4. **`git add` → `git commit` → `git push`** (브랜치: `feat/initial-board`)

> Firebase 룰을 콘솔에서 바꾼 경우에도 `database.rules.json`을 함께 커밋한다.

### CHANGELOG 버전 규칙

```
v1.X   — 새 기능 추가 또는 기존 기능 변경 (마이너 버전 올림)
v1.X.Y — 버그 픽스만 있을 때 (패치 버전 올림)
```

**작성 형식:**

```markdown
## [v1.X] — YYYY-MM-DD

### Added
- 새로 추가된 기능

### Changed
- 기존 동작이 바뀐 내용 (UI 개선, 로직 변경 등)

### Fixed
- 수정된 버그
```

**규칙:**
- 날짜는 오늘 날짜(`currentDate` 기준)로 기록
- 섹션은 해당하는 것만 포함 (Added만 있으면 Changed·Fixed 생략)
- 항목은 사용자 관점에서 한 줄로 간결하게
- 인프라·설정 변경(Firebase 룰, .gitignore 등)도 기록

---

## 프로젝트 개요

Firebase + Vanilla HTML/CSS/JS 기반 **주간 업무 보드 (AV Board)**.
서울AV 팀의 프로젝트별 주간 업무 진행 현황을 관리하고, 대시보드에서 통계·카테고리·담당자별 현황을 한눈에 볼 수 있다.

- **GitHub:** `https://github.com/Supaper/av-board`
- **배포 URL:** `https://supaper.github.io/av-board/` (GitHub Pages, `feat/initial-board` 브랜치)

---

## 빌드·테스트·개발 명령

**빌드 도구 없음.** npm, 번들러, 테스트 프레임워크, 린터 모두 없다.

- 개발 확인: 파일 수정 후 브라우저에서 직접 열거나 `push` → GitHub Pages URL 확인
- 로컬 미리보기 필요 시: `python -m http.server 8080` (또는 VS Code Live Server)
- `git push origin feat/initial-board` → 수 초 내 자동 배포

---

## 파일 구조

```
index.html          — 메인 앱 (CSS + JS 전부 포함, ~2000줄)
admin.html          — 사용자 관리 전용 페이지 (관리자만 접근)
database.rules.json — Firebase Security Rules (git 추적 필수)
CHANGELOG.md        — 버전별 변경 이력
```

---

## 아키텍처

### 단일 파일 구조 (`index.html`)
- `<style>` 에 CSS 전부, `<script type="module">` 에 JS 전부
- CDN으로 로드: Firebase SDK v10.12.0, SheetJS xlsx-0.20.3
- 컴포넌트 분리 없음 — 함수 단위로 DOM 문자열을 생성해 `innerHTML`로 주입

### `admin.html` 역할
- `users/` 노드의 CRUD 전용 페이지 (사용자 추가·수정·삭제)
- Firebase Auth 로그인 후 `users/{uid}.admin === true` 인지 확인
- 관리 가능 필드: `name, email, title, color, order, admin(bool)`
- `order` 값으로 탭 순서 결정 (`renderTabs`에서 `sort`)

### 인증 흐름
1. Google OAuth → `signInWithPopup` → `onAuthStateChanged` → `handleUser(user)`
2. `handleUser`가 `users` 테이블을 이메일로 쿼리 (`orderByChild('email').equalTo(user.email)`)
3. 레코드 없으면 → `accessDenied` 화면 (Firebase Auth 계정이 있어도 앱 접근 불가)
4. 접근 허용 = Firebase `users/` 노드에 이메일이 등록된 사용자

### Firebase 데이터 구조
```
users/
  {uid}/
    name, email, title, color, order, admin (bool)
tasks/
  {uid}/
    {taskId}/
      projectName, category, amount, progressRate,
      schedule, partner, notes, weekLabel,
      createdAt, updatedAt, archived
sites/
  {siteId}/
    name, locations
```

### 리얼타임 리스너 (`startListeners`)
`onValue` 3개가 각각 `allUsers`, `allTasks`, `allSites` 를 채우고 `renderBoard()` 를 호출.
모든 렌더는 이 전역 변수를 읽어 동기적으로 HTML 문자열을 생성한다.

### 보드 렌더링 모드
`viewMode: 'project' | 'week'` 전역 변수로 제어.
- `'project'` (기본): `renderProjectView(uid)` — 프로젝트명별 최신 업무 카드
- `'week'`: 주차 아코디언 그룹 — 이번주(기본 열림) / 다음주(기본 열림) / 과거(기본 닫힘)

### window.* 온클릭 패턴
`<script type="module">` 내부 함수는 인라인 `onclick`에서 접근 불가.
새 핸들러 함수는 반드시 `window.funcName = ...` 으로 노출해야 한다.

### Excel 가져오기 흐름
1. `triggerImport()` → `<input type="file">` 클릭 → SheetJS로 `.xlsx` 파싱
2. 고정 컬럼 위치 기반으로 필드 추출 (헤더 자동 감지 아님)
3. 중복 제거: 동일 프로젝트+주차 중 내용이 변경된 시점만 저장 (마지막 스냅샷 방식)
4. Firebase `tasks/{uid}/` 에 `push()` 로 저장

### 주차 레이블
- 저장 형식: `"6월 4주차"` (날짜 범위 제거 후 정규화)
- `taskWeekLabel(t)`: `.replace(/(\d+)월\s*(\d+)주차.*/, '$1월 $2주차').trim()`
- 주차 번호 계산: `Math.ceil((day + firstDayOfMonth) / 7)`
- `isTaskArchived(t)`: `.archived` 플래그 또는 현재 날짜로부터 60일 초과 경과

---

## 주요 함수 레퍼런스

| 함수 | 역할 |
|---|---|
| `handleUser(user)` | 로그인 후 사용자 조회·권한 확인·리스너 시작 |
| `startListeners()` | 3개 `onValue` 구독 시작 |
| `renderBoard()` | 탭 선택에 따라 대시보드 또는 유저 뷰 렌더 |
| `renderUser(uid)` | `viewMode`에 따라 프로젝트별/주차별 뷰 선택 |
| `renderDashboard()` | DASHBOARD 탭 전체 HTML 반환 |
| `renderProjectView(uid, isOwn)` | 프로젝트별 카드 뷰 HTML 반환 |
| `latestPerProj(uid)` | 프로젝트당 최신 업무 1건씩 반환 (히스토리 뷰용) |
| `taskWeekLabel(t)` | 업무의 주차 레이블 정규화 반환 |
| `currentWeekLabel()` / `nextWeekLabel()` | 이번 주 / 다음 주 레이블 |
| `isTaskArchived(t)` | archived 플래그 또는 60일 자동 아카이브 여부 |
| `openProjHistPopup(uid, projName)` | 프로젝트 히스토리 팝업 열기 (uid='' → 전체) |
| `saveProjHistEntry(ownerUid, tid)` | 히스토리 팝업 내 편집 Firebase 저장 |
| `makePie(data, total)` | 도넛 차트 SVG 문자열 생성 |
| `fmtAmount(v)` | 금액 포맷 (원화) |
| `amtColor(v)` | ≥1억 → green(`#059669`), <1억 → blue(`#3b82f6`) |
| `esc(s)` | HTML 이스케이프 |

### 전역 상태 변수
- `currentProfile` — 로그인 사용자 프로필 `{ id, name, color, admin, ... }`
- `allUsers` / `allTasks` / `allSites` — Firebase `onValue` 로 채워지는 맵
- `activeTab` — 현재 선택된 탭 uid (`'all'` = 대시보드)
- `viewMode` — `'project'` | `'week'`
- `collapsedWeeks` — Set: 이번주/다음주 중 사용자가 닫은 주차
- `manuallyOpenedWeeks` — Set: 과거 주차 중 사용자가 연 주차
- `openedArchived` — Set: 이전 프로젝트 섹션이 열린 uid
- `_phpCtx = {uid, projName}` — 히스토리 팝업 컨텍스트

---

## CSS 레이아웃 규칙

- `#board`: `max-width: 1600px; margin: 0 auto; padding: clamp(20px, 3vw, 48px)`
- `.stat-all`: 기본 4컬럼 grid, 1100px 이상에서 7컬럼
- `.dash-charts`: 기본 1컬럼, 1200px 이상에서 `2fr 1fr`
- 모바일(≤599px): `stat-all` 2컬럼, `dash-charts` 1컬럼, `dash-people` 1컬럼
- 카테고리 배지 색: `CAT_BG` / `CAT_FG` 상수 테이블 (9개 카테고리)
- 주차 헤더 색상:
  - 이번 주 `.wh-current`: `border-left: 4px solid #34a853; background: #f6fff8`
  - 다음 주 `.wh-next`: `border-left: 4px solid #f59e0b; background: #fffdf0`
  - 과거 `.wh-past`: `border-left: 4px solid #e0e0e0; opacity: 0.85`

---

## Firebase 인프라

**⚠️ Firebase 룰 변경 시 `database.rules.json` 도 반드시 같이 커밋.**

- 현재 룰: 로그인한 사용자만 읽기·쓰기 / `users` 경로에 `email` 인덱스
- 룰 복구: `database.rules.json` 내용을 Firebase 콘솔 Rules 탭에 붙여넣고 게시
- 새 사용자 추가: `admin.html` 에서 추가 (또는 Firebase 콘솔에서 `users/` 직접 편집)
- 관리자 권한 부여: 콘솔에서 `users/{uid}/admin: true` 수동 설정

---

## 완료된 기능

- Firebase Auth 로그인 (Google `signInWithPopup`)
- 업무 등록 / 수정 / 삭제 (CRUD)
- 엑셀 대량 임포트 (중복 제거 — 내용 변경 시점만 유지)
- 주차별 뷰: 이번주/다음주/과거 그룹 (과거는 기본 닫힘)
- 프로젝트별 뷰: 프로젝트 이름 클릭 → 히스토리 팝업
- 주차별 뷰에서 업무 클릭 → 프로젝트 히스토리 팝업
- 대시보드: 통계 7카드, 카테고리 도넛 차트, 담당자별 카드
- 대시보드 팝업에서 프로젝트 히스토리 팝업 연결
- 히스토리 팝업 내 인라인 편집 (자신 업무 + 관리자 권한 시 타인 편집 가능)
- [이번 주] / [다음 주] 배지 컬러 코드 헤더
- 사업장 관리 (추가/편집)
- 반응형 레이아웃 (모바일 1컬럼, 데스크탑 멀티컬럼)
- 도넛 차트 개선: 168px SVG, 2컬럼 레전드 그리드

---

## 다음 개발 우선순위 (Backlog)

1. **알림 기능** — 업무 마감일 또는 일정 기반 푸시 알림
2. **PDF/엑셀 리포트 내보내기** — 주간 보고서 자동 생성
3. **프로젝트 마스터 관리** — 프로젝트 목록 별도 관리 화면
4. **다크 모드** — CSS 변수 기반 테마 전환

---

## Git 워크플로

- `main` — 공식 릴리즈 브랜치
- `feat/initial-board` — 현재 활성 개발 브랜치 (GitHub Pages 배포 원본)
- 대규모 기능: 별도 브랜치 → PR → `feat/initial-board` merge
- PR 생성: 브라우저에서 직접 (`https://github.com/Supaper/av-board/pull/new/{branch}`)
- `gh` CLI 사용 시 GitHub token 노출 주의 — 브라우저 방식 권장
