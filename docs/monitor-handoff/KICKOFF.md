# 새 세션 킥오프 프롬프트 (그대로 붙여넣기)

아래 블록 전체를 새 Claude Code 세션(새 레포 `board-monitor`)에 붙여넣으세요.
함께 둔 `PRD.md`, `CLAUDE.md`, `README.md` 도 새 레포에 같이 올리면 더 정확합니다.

---

```
나는 기존에 운영하던 `Supaper/av-board` 프로젝트의 스택을 그대로 차용해서
새 프로그램을 만들려고 한다. av-board의 핵심 패턴:
- Vanilla HTML/CSS/JS 단일 index.html (빌드 없음, ES Modules + CDN)
- Firebase Realtime Database (modular SDK v10.12.0)
- Firebase Auth + Google signInWithPopup 로그인
- GitHub Pages 배포
- 보안 = Google 로그인 + 허용 Gmail 화이트리스트 + Firebase Security Rules,
  Firebase config 키는 공개 키라 클라이언트 임베드 OK

## 만들 것
게시판(thelifechurch.kr)을 주기적으로 모니터링해서, 지정된 작성자들의 신규 글을
카테고리별로 취합하고 ▷ 일별 신규글 알림(Gmail) ▷ 월별 작성자/카테고리 통계(Gmail)
를 보낸다. 장기적으로는 취합 결과를 한글(.hwpx/.hwp) 양식에 자동으로 채워 문서화한다.

## 현재(As-Is)는 Google Sheets + Apps Script로 운영 중이며, 핵심 로직은 다음과 같다
- 검색 URL: http://www.thelifechurch.kr/main/sub.html?boardID=www56&Mode=list&keyfield=name&key={작성자명}
  (작성자명을 URL-encode 해서 게시판 "이름 검색" 결과 목록을 파싱)
- 신규 감지: PropertiesService(스크립트 속성)에 작성자별 마지막 수집 기준점 저장 →
  다음 실행 때 그보다 새 글만 수집
- 저장: 작성자 이름과 같은 시트에 행으로 기록
- 일별: 신규 글 ≥1건이면 MailApp.sendEmail로 HTML 목록 메일 발송
- 월별: 지난달 데이터에서 제목에 "큐티나눔"이 포함된 글을 작성자별로 카운트해
  완주 횟수/달성률(%) 표를 메일 발송 (sendMonthlyQTReport)
- resetMemory()로 기준점 초기화, 시간대는 KST, 날짜 포맷 yyyy-MM-dd
- 일별/월별이 서로 다른 작성자 명단과 다른 스프레드시트를 사용(그룹이 둘 이상)

## 아키텍처 (GitHub Pages는 정적이라 서버 크롤링 불가 → 역할 분리)
- 수집기 = GitHub Actions cron(Node.js): 크롤→파싱→신규감지→카테고리분류→Firebase 적재
  → 변동 시 Gmail 발송. cheerio + iconv-lite(EUC-KR 추정) + firebase-admin 사용.
- 저장소 = Firebase RTDB.
- 리포트 = nodemailer + Gmail SMTP(앱 비밀번호).
- 대시보드 = index.html(정적, GitHub Pages), Google 로그인 + 허용 계정만, Firebase read-only.
- 비밀값은 전부 GitHub Secrets: FIREBASE_SERVICE_ACCOUNT, GMAIL_USER, GMAIL_APP_PASSWORD.

## 데이터 모델 (Firebase RTDB)
config/{boards,authors,categories,recipients}
posts/{작성자}/{postId}/  title,url,postedAt,category,collectedAt
baseline/{작성자}/  lastPostId,lastPostedAt
stats/monthly/{yyyy-mm}/{작성자}/  categoryCounts,total,rate
runs/{ts}/  newCount,status,error
profiles/{uid}, allowedEmails/{email}

## 작업 순서
1. 레포 스캐폴딩: index.html / crawler/{collect.mjs,report.mjs,lib/parse.mjs,package.json}
   / .github/workflows/{daily.yml,monthly.yml} / firebase-rules.json / README.md / CLAUDE.md
   (CLAUDE.md, README.md, PRD.md는 내가 함께 제공한 초안을 사용)
2. 수집기 MVP: 단일 작성자로 크롤·파싱 검증 후 전체 명단 순회. 신규만 적재.
3. 일별 알림 메일 → 4. 월별 통계 메일 (여기까지가 기존 Apps Script 대체)
5. 대시보드(로그인+현황+통계 차트) → 6. 설정 화면 → 7. HWPX 문서 자동화

## 시작 전에 나한테 먼저 물어봐야 할 것 (이게 정확도를 좌우한다)
- 원본 Apps Script "전체" 코드 (게시판 HTML을 파싱하는 정규식/DOM 추출 부분 포함).
  내가 준 코드엔 그 부분이 생략돼 있다. 그 파싱 로직을 lib/parse.mjs로 그대로 이식해야 한다.
- 게시판 페이지 실제 charset (EUC-KR 맞는지)
- 일별/월별 작성자 명단·그룹 정의
- 카테고리(키워드→카테고리) 전체 매핑
- 일별/월별 메일 수신자
- 크롤 주기(cron)
- 대시보드 접근 허용 Gmail 계정

먼저 위 항목들을 나에게 질문해서 채운 뒤, 1번 스캐폴딩부터 시작해줘.
비밀값은 절대 커밋하지 말고, 게시판 robots.txt/이용약관 범위 내에서 정중하게 크롤링해줘.
```
