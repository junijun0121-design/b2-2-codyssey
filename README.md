# 📰 뉴스 요약 자동화 워크플로우 (News Summary Automation Workflow)

n8n과 Google Gemini API, Notion을 연동하여 매일 AI 및 IT 관련 주요 뉴스를 자동으로 수집, 필터링, 요약하여 Notion 데이터베이스에 적재하는 자동화 시스템입니다. 본 시스템은 일시적인 네트워크 오류 및 예외 상황에 대응할 수 있도록 강력한 안정성 설계가 반영되어 있습니다.

## 🔗 관련 링크
* **GitHub Repository**: [Codyssey Team B2-2](https://github.com/junijun0121-design/b2-2-codyssey)
* **n8n Workflow**: [공유 링크](https://j55080634.app.n8n.cloud/workflow/6gmSMwuXa0xEldjN)

---

## 🛠️ 기술 스택 (Tech Stack)
* **Automation**: n8n
* **LLM API**: Google Gemini
* **Data Sources**: RSS Feeds (ITWorld Korea, TechCrunch, AI Times)
* **Storage / Target**: Notion Database, Google Sheets (데이터 조회)

---

## ⚙️ 시스템 아키텍처 및 워크플로우 (Workflow Architecture)

전체 워크플로우는 **[스케줄 트리거] ➔ [뉴스 크롤링] ➔ [시간 필터링] ➔ [필드 추출] ➔ [AI 필터링 및 중복 방지] ➔ [Gemini AI 요약 및 파싱] ➔ [Notion 적재]** 순으로 유기적으로 진행됩니다.

### 1. 스케줄 트리거 및 뉴스 크롤링 (팀원 1 전준)
* **실행 주기**: 매일 자정 (12:00 AM) 1회 자동 실행
* **수집 대상 (AI/IT 관련 뉴스 3종 RSS)**:
  1. **ITWorld Korea**: AI, 클라우드, 개발, 보안, 인프라 등 실무형 기사 수집
  2. **TechCrunch**: 글로벌 테크 스타트업, 벤처 캐피털(VC) 투자 동향 수집
  3. **AI Times**: AI 산업, 연구, 정책 및 딥러닝 트렌드 특화 뉴스 수집
* **데이터 병합**: 수집된 3개의 RSS 피드 데이터를 `Merge` 노드를 통해 하나로 병합

### 2. 필터링 및 중복 방지 처리 (팀원 2 김지현)
* **AI 기사 선별**: n8n `If` 노드를 활용하여 제목(`title`)에 "AI"가 포함된 기사만 필터링 (예: 전체 35건 중 25건 선별)
* **대표 기사 추출**: 전달 데이터를 최소화하기 위해 `Code (JavaScript)` 노드를 통해 필터링된 AI 기사 중 첫 번째 기사 1건만 선택
* **중복 방지 설계**: 기사 원문 URL(`link`)을 기준으로 기존 데이터와 비교하여 동일 URL 존재 시 제외하고 신규 URL만 다음 단계로 전달

### 3. AI 뉴스 요약 및 최종 데이터 적재 (팀원 3 정지태)
* **Google Gemini 노드 연동**: 영어 기사의 '한국어 제목 번역'과 '본문 3줄 요약'을 동시에 수행
  * *프롬프트 규칙*: 제목과 요약 사이에 구분자(`||`) 삽입, 요약문 문장 사이에 줄바꿈 기호(`\n`) 삽입
* **자바스크립트 Expression 데이터 파싱**: 단일 텍스트로 출력된 Gemini의 답변을 n8n 수식으로 분리
  * *기사 제목 추출*: `.split('||')[0].trim()` (구분자 앞의 번역 제목만 추출)
  * *요약문 추출*: `.split('||')[1].trim()` (구분자 뒤의 요약 내용만 추출)
* **Notion 데이터베이스 최종 적재**: 정제된 데이터를 Notion에 최종 저장

---

## 🛡️ 시스템 안정성 및 예외 처리 (Stability & Error Handling) - 팀원 4 김혜민

워크플로우의 중단 없는 운영과 효율적인 예외 처리를 위해 다음과 같이 안정성 축을 설계했습니다.

### 1) 재시도 로직 (Retry Logic)
외부 API 및 네트워크의 일시적 에러에 대응하기 위해 주요 노드에 자동 재시도 기능을 적용했습니다.
* **적용 설정**: `Retry on Fail` 기능 활성화
* **재시도 횟수**: 최대 2회
* **재시도 간격**: 5초 (지연 현상을 빠르게 극복하고 연속성 있는 프로세스 진행)

### 2) 에러 처리 정책 (Error Handling Policy)
| 예외 상황 (Scenario) | 대응 정책 (Policy) | 설계 목적 (Objective) |
| :--- | :--- | :--- |
| **뉴스 기사 미수집** <br>(RSS Read 실패) | 워크플로우 즉시 종료 <br>+ 알림 발송 | 불필요한 후속 작업 실행 방지 및 <br>서버 자원 낭비 차단 |
| **AI 뉴스 요약 실패** <br>(Gemini API 오류) | AI 요약문 없이 <br>원문 데이터 우선 저장 | AI 모델 오류로 인한 <br>정상 뉴스 데이터 누락 최소화 |
| **Notion 저장 실패** <br>(최종 적재 오류) | 에러 로그 기록 <br>+ 관리자 알림 발송 | 오류 상황 신속 파악 및 <br>데이터 즉각 재처리(Recovery) 보장 |

---

## 📊 데이터베이스 스키마 (Notion 적재 필드)

Notion 데이터베이스에 최종적으로 기록되는 필드 구조는 다음과 같습니다.

| 필드명 (Field) | 데이터 설명 (Description) |
| :--- | :--- |
| **title** | 번역 및 공백 정제가 완료된 한국어 기사 제목 |
| **link** | 뉴스 원문 URL |
| **description** | 가독성 있게 줄바꿈 처리된 한글 3줄 요약문 |
| **isoDate** | 뉴스 최초 발생 일시 |

---

## 💡 참고 자료
* [n8n 및 AI 활용 관련 유튜브 가이드](https://www.youtube.com/watch?v=IRnqZRRVgtg)
* [OpencoreAI 기술 문서](https://opencoreai.org/157/)
