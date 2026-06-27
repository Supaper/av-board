# board-monitor

특정 게시판을 모니터링하여 **지정 작성자의 신규 글을 카테고리별로 취합**하고,
**일별 알림 / 월별 통계**를 Gmail로 보고하는 시스템. (장기: 한글 .hwpx 양식 자동 문서화)

`Supaper/av-board`의 스택(Firebase + GitHub Pages + Google 로그인)을 차용했다.

## 아키텍처
```
GitHub Actions(cron)  →  Firebase RTDB  →  GitHub Pages 대시보드
   crawler/                                 (Google 로그인, 허용 계정만)
   ├ collect.mjs  크롤·파싱·신규감지·분류·적재
   └ report.mjs   일별/월별 Gmail 발송
```
- 크롤링/쓰기는 GitHub Actions(Node + firebase-admin)만 수행.
- 대시보드는 정적 `index.html`, Firebase read-only.

## 셋업

### 1) Firebase
1. Firebase 프로젝트 생성 → **Realtime Database**, **Authentication(Google)** 활성화.
2. 웹 앱 등록 → `firebaseConfig`를 `index.html`에 임베드(공개 키).
3. **Security Rules** 적용(`firebase-rules.json`): 로그인 + `allowedEmails`만 `.read`,
   클라이언트 `.write` 금지.
4. 서비스 계정 키 발급(설정 → 서비스 계정 → 새 비공개 키) → JSON 확보.

### 2) GitHub Secrets (Settings → Secrets and variables → Actions)
| Secret | 용도 |
|---|---|
| `FIREBASE_SERVICE_ACCOUNT` | Admin SDK JSON 전체 (크롤러 쓰기) |
| `GMAIL_USER` | 발신 Gmail 주소 |
| `GMAIL_APP_PASSWORD` | Gmail 2단계 인증 후 발급한 **앱 비밀번호** |

> 비밀값은 **절대 코드/커밋에 넣지 않는다.** 워크플로에서 `secrets.*`로만 주입.

### 3) GitHub Pages
- Settings → Pages → 배포 브랜치 지정 → `index.html` 정적 서빙.

### 4) 설정 데이터 입력 (Firebase `config/`)
- `authors` (작성자 목록, 그룹), `categories` (키워드→카테고리), `recipients` (수신자),
  `boards` (게시판 URL/charset).

## 실행
- 일별 수집+알림: `.github/workflows/daily.yml` (cron + 수동 `workflow_dispatch`).
- 월별 통계: `.github/workflows/monthly.yml`.
- 로컬 테스트: `cd crawler && npm i && node collect.mjs`
  (`GOOGLE_APPLICATION_CREDENTIALS`로 서비스 계정 지정).

## 문서
- 제품 요구사항: `docs/PRD.md`
- 개발 가이드/규칙: `CLAUDE.md`
