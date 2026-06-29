# AV Board

서울AV 팀의 **주간 업무 진행 현황 보드**.
프로젝트별·주차별 업무를 관리하고 대시보드에서 통계·카테고리·담당자 현황을 한눈에 확인할 수 있다.

**배포 URL:** https://supaper.github.io/av-board/
**배포 브랜치:** `feat/initial-board` (push 즉시 반영)

---

## 기술 스택

| 역할 | 기술 |
|---|---|
| 프론트엔드 | Vanilla HTML / CSS / JavaScript (ES Modules) |
| 데이터베이스 | Firebase Realtime Database (SDK v10.12.0) |
| 인증 | Firebase Auth (Google signInWithPopup) |
| 호스팅 | GitHub Pages |
| 빌드 | 없음 — `index.html` 단일 파일 |

---

## Firebase 설정

### Realtime Database 보안 룰

**⚠️ 룰은 반드시 `database.rules.json`에 기록하고 git으로 관리한다.**
Firebase 콘솔에서 룰을 변경한 뒤에는 이 파일도 함께 업데이트해야 한다.

현재 적용 룰 (`database.rules.json`):

```json
{
  "rules": {
    "users": {
      ".read": "auth != null",
      ".write": "auth != null",
      ".indexOn": ["email"]
    },
    "tasks": {
      ".read": "auth != null",
      "$uid": {
        ".write": "auth != null"
      }
    },
    "sites": {
      ".read": "auth != null",
      ".write": "auth != null"
    }
  }
}
```

룰 적용 방법:
1. Firebase 콘솔 → Realtime Database → Rules 탭
2. `database.rules.json` 내용을 붙여넣고 게시(Publish)
3. 변경 사항을 이 파일에도 반영 후 commit

### 데이터 경로 구조

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

### 관리자 권한 부여

Firebase 콘솔 → Realtime Database → `users/{uid}/admin` 값을 `true`로 수동 설정.
관리자는 자신 외 다른 사람의 업무도 편집할 수 있다.

---

## 로컬 실행

별도 빌드 없이 파일을 브라우저에서 직접 열거나 정적 서버로 서빙한다.

```bash
# Python 간이 서버 (포트 8000)
python -m http.server 8000

# 또는 VS Code Live Server 확장 사용
```

---

## 배포 워크플로

```bash
git checkout feat/initial-board
# ... 파일 수정 ...
git add .
git commit -m "feat: 설명"
git push origin feat/initial-board
# GitHub Pages 자동 반영 (1~2분 소요)
```

---

## 파일 구조

```
av-board/
├── index.html           # 앱 전체 (HTML + CSS + JS 단일 파일)
├── admin.html           # 관리자 도구 페이지
├── database.rules.json  # Firebase RTDB 보안 룰 (git 추적)
├── CHANGELOG.md         # 버전별 변경 이력
├── CLAUDE.md            # Claude Code 작업 가이드
└── README.md            # 이 파일
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| 로그인 후 접근 거부 화면 | `users/{uid}` 레코드 없음 | Firebase 콘솔에서 사용자 레코드 직접 추가 |
| DB 쿼리 콘솔 경고 | `users` 인덱스 누락 | `database.rules.json`의 `.indexOn: ["email"]` 적용 확인 |
| 룰 날아감 | 콘솔에서 실수로 덮어씀 | `database.rules.json` → 콘솔에 붙여넣기 후 게시 |
| GitHub Pages 미반영 | `main` 브랜치에만 push | `feat/initial-board` 브랜치에 push |
