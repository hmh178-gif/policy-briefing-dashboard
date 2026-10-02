# 정책브리핑 보도자료 수집 · 부처별 현황 대시보드

대한민국 정책브리핑(korea.kr) 보도자료를 **날짜별로 실시간 수집**하여 부처별 발표 현황과 기사 목록을 시각화한 반응형 웹 대시보드입니다.

🔗 **[웹앱 바로가기](https://script.google.com/macros/s/AKfycby1b9qY14lXayc3v1IGLcZPQaQSDfVdqBWT8Ok5Ueo0s6fkSqC39_lihxT2kzsNXPim/exec)**

---

## 1. 프로젝트 요약

| 항목 | 내용 |
|---|---|
| 수집 대상 | `korea.kr/briefing/pressReleaseList.do` (정적 페이지) |
| 수집 항목 | 제목 · 날짜 · 부처명 · 요약 · 원문링크 · newsId |
| 초기 구현 | Python(BeautifulSoup) 크롤링 → 정적 HTML 대시보드 |
| 최종 구현 | Google Apps Script 웹앱 (날짜 선택 시 서버가 실시간 수집) |
| 기간 | 2026.08 |

---

## 2. 핵심 과제 — 정적 HTML은 왜 부족했는가

처음에는 Colab에서 크롤링한 데이터를 `IPython.display.HTML`로 출력하고, 이를 단일 HTML 파일로 저장했습니다. 이 파일은 차트 표시·부처 필터·반응형 레이아웃까지 정상 동작했지만, **"다른 날짜를 새로 수집"하는 기능만은 구현할 수 없었습니다.**

원인은 세 가지였습니다.

1. 저장된 HTML에는 Python 실행 환경이 없음
2. 브라우저에서 korea.kr로 직접 요청 시 **CORS 정책에 차단됨**
3. "반응형"은 화면 크기 대응을 뜻할 뿐, 데이터 재수집과 무관함

→ 단순 HTML 파일이 아니라 **서버 역할을 하는 구성 요소**가 필요하다고 판단, Google Apps Script 웹앱으로 전환했습니다.

### 전환 후 구조

```
사용자 브라우저
    ↓  날짜 선택 / 새로 수집
Index.html  (화면 · 상호작용)
    ↓  google.script.run
Code.gs     (서버 · 크롤링 · 정제 · 집계)
    ↓  UrlFetchApp
정책브리핑 보도자료 목록
    ↓
Index.html 대시보드 갱신
```

브라우저가 korea.kr에 직접 접근하지 않고 **Apps Script 서버가 대신 요청**하므로 CORS 제약을 받지 않습니다.

---

## 3. 크롤링 설계

### 선택자 확정 과정

목록 컨테이너를 찾는 데 시행착오가 있었습니다.

- `ul li` → 252개 (메뉴·푸터까지 포함, 범위 과다)
- `ul.lst` → 첫 항목이 "홈으로" (브레드크럼 메뉴였음)
- **상세 링크(`pressReleaseView.do`)가 20개 안팎 반복되는 지점**을 기준으로 역추적
- → `div.list_type li` 로 확정

### 최종 선택자

| 항목 | 선택자 |
|---|---|
| 목록 항목(부모) | `div.list_type li` |
| 제목 | `strong` |
| 날짜 | `span.source > span:nth-of-type(1)` |
| 부처명 | `span.source > span:nth-of-type(2)` |
| 요약 | `span.lead` |
| 상세링크 | `a[href*='pressReleaseView.do']` → `urljoin`으로 절대경로 변환 |

> 💡 날짜·부처가 상세페이지에만 있는 줄 알았으나 **목록의 `span.source`에 함께 존재**했습니다. 덕분에 상세페이지를 40회 방문할 필요 없이 목록 요청만으로 수집이 가능해졌습니다.

### 정제 처리

- **중복 제거**: `newsId` 기준
- **요약 잡음 제거**: 대표 기사만 요약 앞에 `보도자료 보도시점…배포…` 머리말이 붙어 서식이 달랐음 → 요약 내에서 제목이 시작되는 위치를 찾아 앞부분만 잘라냄 (행 삭제 없이 서식 통일)
- **서버 부하 방지**: 페이지 간 `time.sleep` 적용

---

## 4. 주요 기능

- 한국 시간 기준 **오늘 보도자료 자동 수집**
- 날짜 직접 선택 / 이전·다음 날짜 이동 / 선택 날짜 재수집
- KPI 카드 — 전체 보도자료 건수, 발표 부처 수
- Chart.js 기반 **부처별 가로 막대 차트 + 도넛 차트**
- 차트 클릭 또는 부처 버튼으로 **기사 목록 필터링**
- 제목·요약·원문 링크 기사 카드
- 모바일 / PC 반응형 레이아웃

---

## 5. 보안 처리

- 수집한 제목·요약은 HTML 문자열로 직접 삽입하지 않고 **`textContent`로 출력** (XSS 방지)
- 원문 링크는 **`http`·`https` 프로토콜만 허용**

---

## 6. 기술 스택

`Google Apps Script` `JavaScript` `HTML/CSS` `Chart.js`
`Python` `requests` `BeautifulSoup` `pandas` (초기 프로토타입)

---

## 7. 파일 구조

```
.
├── Code.gs                      # 웹앱 서버 — doGet, 크롤링, 정제, 집계
├── Index.html                   # 대시보드 화면 · 차트 · 필터
├── prototype/
│   ├── crawling.ipynb           # Colab 초기 크롤링 코드
│   └── press_dashboard.html     # 정적 HTML 대시보드 (전환 전 버전)
└── README.md
```

---

## 8. 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| 선택자가 252개 매칭 | 범위가 너무 넓음 | 상세 링크 기준 역추적 → `div.list_type li` |
| 날짜가 0번 행만 채워짐 | 요약 텍스트에서만 날짜 추출 시도 | `span.source`에서 직접 추출 |
| 저장한 HTML이 0바이트 | 저장 시 잘못된 변수명 사용 | 변수 수정 + `os.path.getsize`로 크기 검증 |
| Colab에서 차트 미표시 | Colab이 동적 스크립트 로딩 차단 | Chart.js를 `<head>`에 직접 삽입한 완전 문서로 저장 |
| 새 날짜 수집 불가 | 브라우저 CORS + Python 환경 부재 | Apps Script `UrlFetchApp`을 서버 역할로 사용 |

---

## 9. 한계 및 개선 방향

- **외부 요청 할당량** — 공개 웹앱이라 '새로 수집'을 반복 클릭하면 Apps Script 할당량이 소진됨. 날짜별 결과 캐시와 중복 요청 방지가 필요
- **구조 변경 취약성** — korea.kr의 HTML 구조가 바뀌면 선택자가 깨짐. 변경 감지·알림 로직 미구현
- **오류 로그 미수집** — 수집 실패 시 사용자 안내만 하고 기록은 남기지 않음
