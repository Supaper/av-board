# AV Board — Project Context for Claude Code

## ⚠️ Claude Code 필수 행동 규칙

> 이 규칙은 매 세션 시작마다 읽고, 기능 작업이 끝날 때마다 반드시 실행한다.

### 기능 구현 완료 즉시 (선언 전에) 해야 할 일

기능 작업이 끝났다고 말하기 **전에** 아래를 직접 실행한다:

1. **`CLAUDE.md`** (이 파일)
   - `## 완료된 기능` 섹션에 새 항목 추가
   - `## 다음 개발 우선순위 (Backlog)` 에서 해당 항목 제거 및 번호 재정렬

2. **`CHANGELOG.md`**
   - 해당 버전 항목에 구현 내용 기록 (Added / Changed / Fixed)
   - 버전 규칙: 기능 추가·변경 → `v1.X` 올림 / 버그픽스만 → `v1.X.Y` 올림

3. **`git add` → `git commit` → `git push`**
   - 브랜치: `feat/initial-board` (현재 개발 브랜치)
   - GitHub Pages가 이 브랜치를 기준으로 배포되므로 push 즉시 반영됨

> 업데이트를 빠뜨리면 다음 세션의 Claude가 이미 완성된 기능을 "아직 할 일"로 잘못 안내한다.
> CHANGELOG 없이 인프라 설정(Firebase 룰 등)을 변경하면 사고 발생 시 복구 근거가 없어진다.

---

## 프로젝트 개요

Firebase + Vanilla HTML/CSS/JS 기반의 **주간 업무 보드 (AV Board)**.
서울AV 팀의 프로젝트별 주간 업무 진행 현황을 관리하고, 대시보드에서 통계·카테고리·담당자별 현황을 한눈에 볼 수 있다.

**GitHub:** `https://github.com/Supaper/av-board`
**배포 URL:** `https://supaper.github.io/av-board/` (GitHub Pages, `feat/initial-board` 브랜치)
**개발자:** 서울AV (`soojhann@seoulav.co.kr`)

---

## 기술 스택

| 역할 | 사용 기술 |
|---|---|
| 프론트엔드 | Vanilla HTML / CSS / JavaScript (ES Modules) |
| 데이터베이스 | Firebase Realtime Database (modular SDK v10.12.0) |
| 인증 | Firebase Auth (`signInWithPopup` — Google) |
| 호스팅 | GitHub Pages (`feat/initial-board` 브랜치) |
| 빌드 도구 | 없음 (단일 `index.html`) |

---

## 아키텍처 핵심 규칙

### 단일 파일 구조
- 모든 코드는 `index.html` 한 파일에 집중
- `<style>` 태그에 CSS, `<script type="module">` 에 JS
- 외부 라이브러리는 CDN (`<script src>`) 사용
- 컴포넌트 분리 없음 — 함수 단위로 화면을 렌더링

### Firebase 데이터 구조
```
users/
  {uid}/
    name, email, title, color, admin (bool)
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

### window.* 온클릭 패턴
- HTML 인라인 `onclick="funcName()"` 핸들러는 `window.funcName`으로 노출해야 동작
- 모듈 스코프 함수는 `window.openProjHistPopup = ...` 형태로 등록
- `<script type="module">` 내부 함수는 인라인 onclick에서 직접 접근 불가

### 주차 레이블 정규화
- 엑셀 임포트 데이터에 날짜 범위가 붙어 있음: `"6월 4주차(6/22~6/26일)"`
- `taskWeekLabel(t)` 함수가 정규식으로 제거: `.replace(/(\d+)월\s*(\d+)주차.*/, '$1월 $2주차').trim()`
- 앱 내 주차 계산: `Math.ceil((day + firstDayOfMonth) / 7)` 공식 사용

### 관리자 권한
- `users/{uid}/admin: true` 를 Firebase 콘솔에서 수동 설정
- 편집 권한 체크: `const canEdit = ownerUid === currentProfile?.id || currentProfile?.admin === true`

---

## 주요 함수 레퍼런스

| 함수 | 역할 |
|---|---|
| `taskWeekLabel(t)` | 업무의 주차 레이블 정규화 반환 |
| `currentWeekLabel()` | 이번 주 레이블 (`N월 M주차`) |
| `nextWeekLabel()` | 다음 주 레이블 |
| `latestPerProj(uid)` | uid별 프로젝트당 최신 업무 한 건씩 반환 |
| `isTaskArchived(t)` | `.archived` 플래그 또는 60일 자동 아카이브 여부 |
| `amtColor(v)` | 금액 색상 (≥1억 → green, <1억 → blue) |
| `fmtAmount(v)` | 금액 포맷팅 (억/만 단위) |
| `openProjHistPopup(uid, projName)` | 프로젝트 히스토리 팝업 열기 (uid='' 이면 전체) |
| `toggleProjHistEdit(tid)` | 히스토리 팝업 내 인라인 편집 토글 |
| `saveProjHistEntry(ownerUid, tid)` | 히스토리 팝업 내 편집 저장 (Firebase update) |
| `makePie(data, total)` | 도넛 차트 SVG 생성 |
| `renderBoard()` | 주차별 보드 뷰 전체 렌더링 |
| `renderDashboard()` | 대시보드 전체 렌더링 |

### 상태 변수
- `collapsedWeeks` — Set: 이번주/다음주 중 사용자가 닫은 주차
- `manuallyOpenedWeeks` — Set: 과거 주차 중 사용자가 연 주차
- `_phpCtx = {uid, projName}` — 히스토리 팝업 컨텍스트 (편집 후 새로고침용)
- `currentProfile` — 현재 로그인한 사용자 프로필 객체

---

## CSS 레이아웃 규칙

- `#board`: `max-width: 1600px; margin: 0 auto; padding: clamp(20px, 3vw, 48px)`
- `.stat-all`: 기본 4컬럼 grid, 1100px 이상에서 7컬럼
- `.dash-charts`: 기본 1컬럼, 1200px 이상에서 `2fr 1fr`
- 모바일(≤599px): `stat-all` 2컬럼, `dash-charts` 1컬럼, `dash-people` 1컬럼
- 주차 헤더 색상:
  - 이번 주 `.wh-current`: `border-left: 4px solid #34a853; background: #f6fff8`
  - 다음 주 `.wh-next`: `border-left: 4px solid #f59e0b; background: #fffdf0`
  - 과거 `.wh-past`: `border-left: 4px solid #e0e0e0; opacity: 0.85`

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

## 인프라 & 배포

### GitHub Pages
- **배포 브랜치:** `feat/initial-board`
- **배포 URL:** `https://supaper.github.io/av-board/`
- push 즉시 자동 배포됨 (빌드 없음)

### PR 워크플로
- 대규모 기능은 별도 브랜치 → PR → `feat/initial-board` merge
- PR은 브라우저에서 직접 생성: `https://github.com/Supaper/av-board/pull/new/{branch}`
- `gh` CLI 사용 시 GitHub token 노출 주의 — 브라우저 방식 권장

### Firebase

**⚠️ Firebase 룰은 반드시 `database.rules.json`으로 git 추적한다.**

- Firebase config 키는 클라이언트 코드에 임베드 (공개 키이므로 비밀이 아님)
- 보안은 Firebase Security Rules로 처리 (콘솔에서 관리)
- **룰 변경 시 절차:**
  1. 콘솔에서 룰 수정 → 게시
  2. `database.rules.json` 동일하게 업데이트
  3. 커밋에 포함 (`git add database.rules.json`)
- **룰 복구 시:** `database.rules.json` 내용을 콘솔 Rules 탭에 붙여넣고 게시
- 현재 룰 요약: 로그인한 사용자만 읽기·쓰기 / `users` 경로에 `email` 인덱스 필요

---

## 개발 환경

- **OS:** Windows 11 Pro
- **Shell:** PowerShell (primary) / Git Bash (POSIX 스크립트용)
- **Git 주의:** PowerShell에서 git이 안 되면 PATH에 `C:\Program Files\Git\cmd` 추가

### Claude Code 세션 시작 시 확인 사항
```bash
git fetch origin
git status
git log --oneline -5
```
- `feat/initial-board` 브랜치에서 작업 중인지 확인
- 작업 후 push까지 완료해야 GitHub Pages에 반영됨

---

## Git 워크플로

### 브랜치 전략
- `main` — 공식 릴리즈 브랜치
- `feat/initial-board` — 현재 활성 개발 브랜치 (GitHub Pages 배포 원본)
- 기능 브랜치 → `feat/initial-board` PR merge 패턴

### 커밋 컨벤션
```
feat: 새 기능
fix: 버그 수정
style: CSS/UI 변경 (기능 변경 없음)
refactor: 코드 정리
```
