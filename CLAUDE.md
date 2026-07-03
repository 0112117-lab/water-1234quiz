# CLAUDE.md — 정수퀴즈 프로젝트 작업 지침

> 이 문서는 Claude(및 다른 AI/개발자)가 이 프로젝트를 이어서 작업할 때 반드시 먼저 읽는 기준 문서입니다.
> **환각 방지 원칙:** 이 문서에 없는 경로/기능/수치를 지어내지 말 것. 불확실하면 실제 `index.html`을 열어 확인할 것.

---

## 1. 프로젝트 개요

- **이름:** 정수퀴즈 (정수시설운영관리사 기출·적중예상 학습 웹앱)
- **형태:** **단일 HTML 파일 PWA** (`index.html` 하나에 HTML+CSS+JS 전부 포함)
- **빌드 도구 없음:** 프레임워크 없음, 번들러 없음, npm/package.json 없음. 파일 하나가 곧 앱.
- **백엔드 없음:** 서버·DB·API 없음. 100% 클라이언트 사이드.
- **배포:** GitHub Pages
  - 저장소: `0112117-lab/water-1234quiz`
  - 라이브 URL: `https://0112117-lab.github.io/water-1234quiz/`
- **산출물 파일:** `index.html` (약 1.14 MB, 문항 데이터 내장)

## 2. 실행·배포 방법

- **로컬 확인:** `index.html`을 브라우저로 열면 즉시 동작(빌드 불필요).
- **배포:** `index.html`을 저장소 루트에 **덮어쓰기 업로드 → Commit** 하면 1~2분 내 라이브 반영.
- **중요:** 홈 화면 추가·PWA 설치·매니페스트는 **https 주소에서만** 동작한다. 다운로드한 로컬 파일(`file://`, `nt://`)에서는 설치 버튼이 비활성화된다. 반드시 라이브 URL로 접속해 테스트할 것.

## 3. 코드 구조 (단일 파일 내부)

```
index.html
├── <head>  메타·PWA 매니페스트(data URI)·apple-touch-icon·<title>
├── <style> 네온 다크 테마 (CSS 변수 :root 기반)
├── <body>  #app 컨테이너 + header + #toast + #aiModal
└── <script>
     ├── const QUESTIONS = [ ...문항... /*__DATA__*/ ];   // 문항 데이터 배열
     ├── 상태(S) 로드/저장  (store.load/save, persist)
     ├── 화면 렌더 (goHome, renderQ, showResult, showStats, showSettings)
     ├── 퀴즈 로직 (startQuiz, nav, pick, setAns, toggleMark, startWrong, startMarked)
     └── 부가 기능 (askAI, openAI, addHome, exportData, importData, resetAll)
```

## 4. 데이터 모델 (문항 1개)

```js
{
  g:  "ex31_2",              // group key (회차/장 식별자)
  gt: "제31회 2급 기출문제 (2022)", // group title (화면 표시명)
  s:  "수처리공정",           // subject (과목)
  n:  1,                      // 문항 번호 (회차 내 1~80)
  q:  "문제 지문...",         // 문제
  c:  ["보기1","보기2","보기3","보기4"], // 보기 4개 (배열 길이 4 고정)
  a:  2,                      // 정답 번호 (1~4)
  e:  "해설..."               // 해설
}
```

- **문자열 내 큰따옴표는 `\"` 로 이스케이프.**
- **새 문항은 반드시 `/*__DATA__*/` 마커 앞에 삽입.**
- **과목(s) 매핑 규칙** (기출 회차 기준, 문항번호별):
  - q1–20 = 수처리공정
  - q21–40 = 수질분석및관리
  - q41–60 = 설비운영
  - q61–80 = 수리학

## 5. 그룹(콘텐츠) 체계

| 접두 | 의미 |
|---|---|
| `b1c*` | 1권 적중예상문제 (과목1·2) |
| `b2c*` | 2권 적중예상문제 (3과목 설비운영) |
| `b3c*` | 3권 적중예상문제 |
| `ex##_2` | 제##회 **2급** 기출문제 |
| `ex##_3` | 제##회 **3급** 기출문제 |

- 홈 화면은 4개 파트(1권/2권/3권/과년도 기출)로 접이식 목록 표시(`togglePart`).
- **현재 총 문항: 2,038** (검증 기준값). 기출 회차는 각 1~80번 완전.

## 6. 상태(State) 구조 — localStorage

- **키:** `jsq_v1` (JSON 직렬화)
- **구조:**
  ```js
  S = {
    exams: { [groupKey]: { answers: { [n]: 선택번호 } } }, // 회차별 응답
    marks: [ "group-n", ... ],   // 북마크(별표)한 문항 qid 목록
    wrongHist: { [qid]: 오답횟수 },
    best: 최고 연속정답,
    last: { key: 마지막에 푼 groupKey }, // 이어풀기 기준
    openParts: { [partKey]: 펼침여부 }
  }
  ```
- `qid(q)` = `q.g + '-' + q.n`

## 7. 주요 기능 (실제 구현됨)

- **이어풀기:** 마지막 푼 장에 미응답이 있으면 그 장을 이어서, **다 풀었으면 다음 미완료 장으로 자동 이동**.
- **풀이 모드:** 전체 이어풀기 / 과목별(`subject`) / 셔플(`shuffle`) / 처음부터(`restart`).
- **오답노트**(`startWrong`) · **북마크**(`startMarked`, 별표) · **통계**(`showStats`).
- **백업/복원:** JSON을 클립보드로 내보내기(`exportData`) / 붙여넣어 복원(`importData`).
- **AI 질문하기:** 문제·보기·정답을 정해진 프롬프트로 클립보드 복사 후, 클로드/ChatGPT로 이동.
  - 모바일: 설치된 앱으로 직접 이동(Android intent / iOS 유니버설 링크), 미설치 시 웹 폴백.
  - GPT는 지정 공유 링크(`chatgpt.com/share/6a3b8f68-...`)로 연결.
- **홈 화면에 추가:** `beforeinstallprompt` 캡처 + 수동 안내(`addHome`). **https에서만 동작.**

## 8. 작업 규칙 (반드시 준수)

1. **파일 편집은 file-tool(Read/Grep/Edit)이 기준.** 이 환경의 bash 마운트는 캐시가 밀려 옛 파일을 보여줄 때가 있으니, bash 검증 결과가 이상하면 file-tool로 재확인.
2. **문항 추가는 `/*__DATA__*/` 마커 앞에.** 삽입 후 반드시 검증: 각 회차 1~80 누락/중복 없음, `a`는 1~4, 필드(g/gt/s/n/q/c/e) 결손 0.
3. **정답이 불확실한 항목**(법규·고시 수치, 환산계수 등)은 교재 정답표 번호를 기록하되 해설에 `(교재 정답표 기준)` 표기. 임의로 지어내지 말 것.
4. **수치·계산 문항**(표면부하율, CT값, 손실수두, 동력 등)은 직접 계산해 정답표와 대조 후 기록.
5. 큰 변경 후에는 `<script>` 구문 검사 + 데이터 무결성 검사를 돌린다.

## 9. 알려진 제약

- **iOS Safari:** 웹페이지/홈화면 웹앱에서 외부 앱(클로드/ChatGPT) 강제 실행 불가(Apple 플랫폼 제약). 아이폰에서는 웹으로 열림.
- **로컬 파일 실행 시:** 홈 화면 추가·PWA 설치 불가(https 필요).
- **버전 문자열 미내장:** 현재 코드에 앱 버전 표기가 없음. → `FUTURE_ROADMAP.md` 참고(버전 표기 추가 권장).

## 10. 관련 문서

- `docs/APP_SPEC.md` — 기능/화면/데이터 상세 명세
- `docs/RELEASE_CHECKLIST.md` — 배포 전 점검표
- `docs/BUGLOG.md` — 버그 이력
- `docs/SECURITY_PRIVACY_REVIEW.md` — 보안·개인정보 검토
- `docs/FUTURE_ROADMAP.md` — 향후 개선 로드맵
