# 영어학원 월간 학습레포트 자동화

## 1. 프로젝트 배경

아버지가 운영하는 영어학원에서는 매달 학생별 학습 기록을 확인한 뒤, 과목별 학습 범위와 교재 진행 상황을 정리하고 학부모에게 전달할 월간 학습레포트를 수작업으로 작성해 왔다. 학생마다 달력 형태의 학습 기록을 확인하고, 단어·작문·듣기·독해의 학습 범위를 정리한 뒤, 교재 정보와 총평을 작성하여 정해진 레포트 양식에 옮기는 작업이 반복적으로 발생했다.

본 프로젝트의 목표는 **Google Drive에 학생 데이터 파일이 등록되면 학습 내용을 자동으로 분석하고, 생성형 AI를 이용해 총평을 작성한 뒤, Google Slides 템플릿을 기반으로 월간 학습레포트를 자동 생성하는 것**이다.

---

# 프로젝트 1. 자동화 도구 비교 구현

## 2. 비교 대상

동일한 자동화 업무를 다음 두 가지 방식으로 구현했다.

1. **Make**
2. **Activepieces + ChatGPT Work**

두 구현 모두 최종적으로 동일한 학생 데이터와 동일한 Google Slides 레포트 템플릿을 사용한다.

---

## 3. 공통 워크플로우

```text
Google Drive에 새 학생 데이터 파일 등록
                ↓
Google Sheets 데이터 읽기
                ↓
과목별 학습기록 분석
                ↓
학습시작 / 학습완료 / test 구분
                ↓
단어·작문·듣기·독해 월간 학습범위 생성
                ↓
학생정보·교재정보·학습횟수·달력 데이터 읽기
                ↓
Gemini로 월간 총평 생성
                ↓
Google Slides 템플릿의 placeholder 치환
                ↓
학생별 월간 학습레포트 생성
```

---

## 4. Make 구현

### 워크플로우 구성 화면

![Make 전체 워크플로우](assets/make-workflow.png)

Make에서는 Google Drive Trigger 이후 Google Sheets, Tools, Array Aggregator, Text Parser 등을 조합하여 데이터를 단계적으로 처리했다.

특히 달력에 기록된 학습 데이터를 과목별로 분리하고 다음 값을 계산하도록 구성했다.

- 단어·작문·듣기·독해의 시작 단원과 마지막 단원
- `학습시작`
- `학습완료`
- 작문 등의 `test` 횟수
- 기존 교재 종료 후 새 교재 시작 여부

이후 데이터를 다시 하나의 bundle로 집계한 뒤 Gemini를 한 번만 실행하도록 구성했다. 마지막 단계에서는 Google Slides 템플릿의 학생정보, 과목별 횟수, 과목별 학습범위, 교재정보, AI 총평, 달력 데이터(D1~D42)를 실제 값으로 치환했다.

---

## 5. Activepieces 구현

### 워크플로우 구성 화면

![Activepieces 전체 워크플로우](assets/activepieces-workflow.png)

Activepieces에서는 다음과 같은 구조로 구현했다.

```text
1. Google Drive - New File
2. Router
   ├─ Branch 1 → 정상 처리
   └─ Otherwise → Stop Flow
3. Google Sheets - Read Data Range
4. Google Sheets - Read Data Range
5. Code
6. Google Sheets - Read Data Range
7. Chat Gemini
8. Google Slides - Generate from Template
9. Google Drive - Move File
```

Make에서 여러 개의 Aggregator, Parser, Variable 모듈로 나누어 처리했던 데이터 가공 부분은 Activepieces에서 **Code Step**으로 묶어 구현했다. 그 결과 전체 단계 수를 줄이면서 동일한 결과를 생성할 수 있었다.

---

## 6. Activepieces의 Router / Stop Flow 분기

Activepieces 구현에는 Trigger 바로 다음에 **Router**를 배치했다.

Router는 입력된 파일을 두 경로로 나눈다.

- **Branch 1**: 자동화가 실제로 처리해야 하는 입력은 이후 Google Sheets 읽기 단계로 진행한다.
- **Otherwise**: Branch 1 조건을 만족하지 않는 입력은 `Stop Flow`로 전달한다.

`Stop Flow`는 해당 실행을 그 지점에서 종료시키는 역할을 한다. 따라서 처리 대상이 아닌 입력이 들어왔을 때 뒤쪽의 Google Sheets, Gemini, Google Slides 단계까지 불필요하게 실행되는 것을 막을 수 있다.

이 구조를 사용한 이유는 다음과 같다.

1. 잘못된 입력으로 인한 오류 방지
2. 불필요한 AI 호출 방지
3. 불필요한 실행량 및 credit 사용 방지
4. Trigger는 유지하면서 실제 처리 대상을 조건에 따라 제한

> **참고:** 업로드한 전체 Flow 화면에서는 Router의 Branch 1 조건식 자체는 펼쳐져 있지 않으므로, 본 문서에서는 화면에서 확인 가능한 구조와 역할을 중심으로 설명했다.

---

## 7. 실행 결과

두 자동화의 최종 출력 형식은 동일한 Google Slides 월간 학습레포트이다.

### 결과물 1페이지

![월간 학습레포트 1페이지](assets/monthly-report-page1.png)

1페이지에는 다음 정보가 자동으로 반영된다.

- 학생 이름
- 학교 및 학년
- 학습 영역 수
- 이번 달 총 학습 횟수
- 단어·듣기·작문·독해별 학습 횟수
- 각 과목의 월간 학습 범위

### 결과물 2페이지

![월간 학습레포트 2페이지](assets/monthly-report-page2.png)

2페이지에는 다음 정보가 자동으로 반영된다.

- 학생 이름 및 학교/학년
- 월간 학습 달력
- 학습 교재 정보
- Gemini가 생성한 월간 총평

달력은 Google Sheets의 6주 × 7일 데이터를 `D1~D42` placeholder와 대응시켜 자동으로 입력하도록 구성했다.

---

## 8. 실행 화면을 별도로 첨부하지 않은 이유

본 문서에서는 **워크플로우 구성 화면과 실제로 생성된 최종 레포트 결과물**을 증빙으로 사용했다.

개발 과정에서 이미 여러 차례 실제 실행과 수정 테스트를 수행했으며, 추가적인 재실행은 Make 및 AI 관련 credit을 추가로 소모할 수 있기 때문에 제출용 캡처를 얻기 위한 목적만으로 다시 실행하지 않았다.

다만 과제의 “각 도구별 실행 결과 화면” 항목을 매우 엄격하게 평가하는 경우에는, 새로 실행하지 않고 **기존 Run History / 실행 이력 화면을 열어 과거 성공 실행을 캡처하는 방식**으로 보완할 수 있다. 이 경우 추가 자동화 실행이나 AI 호출이 필요하지 않다.

---

## 9. Make vs Activepieces 비교

| 비교 항목 | Make | Activepieces + ChatGPT Work |
|---|---|---|
| UI/UX | 모듈을 가로로 연결하는 시각적 캔버스 | 세로형 Step 기반 Flow Builder |
| 구현 방식 | Parser, Tools, Aggregator, Variable 등을 세밀하게 조합 | 복잡한 가공을 Code Step에 집중 가능 |
| 초기 설정 난이도 | bundle, map, aggregator 개념을 이해해야 해 학습 곡선이 큼 | 전체 구조는 단순하지만 Code Step 사용 시 코드 이해가 필요 |
| 구축 속도 | 대부분 직접 설정하여 시간이 많이 소요됨 | ChatGPT Work를 이용해 Make 구조를 참고하면서 빠르게 재현 |
| 디버깅 | 각 모듈의 input/output과 bundle을 매우 세밀하게 확인 가능 | step별 실행 결과 확인 가능하지만 Make보다 구조가 단순함 |
| 조건 분기 | Filter/Router를 시각적으로 구성하기 편함 | Router + Stop Flow로 비대상 입력을 조기에 종료 가능 |
| 데이터 가공 | no-code 함수 조합으로 세밀한 처리 가능 | Code Step으로 복잡한 파싱 로직을 한 곳에 모을 수 있음 |
| Google 연동 | Drive, Sheets, Slides 관련 세부 설정이 풍부함 | 이번 프로젝트에 필요한 Drive, Sheets, Slides 연동 가능 |
| AI 연동 | Gemini 모듈을 연결하고 입출력을 직접 확인하기 쉬움 | Chat Gemini Step으로 동일한 총평 생성 가능 |
| 긴 워크플로우 관리 | 모듈과 bundle 수가 많아지면 캔버스와 credit 관리가 복잡해짐 | Step 수를 줄이고 로직을 묶기 쉬움 |
| 유지보수 | 각 세부 로직을 개별 모듈에서 수정하기 편함 | 중앙 Code Step을 수정하면 여러 로직을 한 번에 변경 가능 |

---

## 10. 장단점

### Make의 장점

- 데이터가 이동하는 과정을 모듈별로 매우 자세히 확인할 수 있다.
- Filter, Text Parser, Array Aggregator 등 세부적인 no-code 기능이 강력하다.
- 복잡한 오류가 발생했을 때 어느 단계에서 문제가 생겼는지 찾기 쉽다.
- Google 서비스와의 세밀한 mapping이 편리하다.

### Make의 단점

- 복잡한 자동화에서는 모듈 수가 빠르게 증가한다.
- bundle 수가 증가하면 뒤쪽 Action이 여러 번 실행될 수 있다.
- 초보자가 bundle, aggregator, map 함수 등을 이해하는 데 시간이 필요하다.
- 이번 프로젝트처럼 복잡한 데이터 처리 로직을 처음 만드는 데 상당한 시간이 필요했다.

### Activepieces의 장점

- 전체 Flow가 비교적 단순하게 보인다.
- Code Step으로 복잡한 로직을 하나의 단계에 묶을 수 있다.
- Router와 Stop Flow를 이용해 필요 없는 실행을 조기에 종료할 수 있다.
- ChatGPT Work를 이용하여 이미 완성된 Make 워크플로우를 참고하면서 재구현했기 때문에 구축 시간을 크게 줄일 수 있었다.

### Activepieces의 단점

- 복잡한 데이터 가공을 Code Step에 집중하면 no-code만으로 구조를 이해하기 어려울 수 있다.
- 각 데이터 처리 과정을 시각적으로 세분화해서 확인하는 기능은 Make가 더 편리했다.
- 외부 계정 연결과 OAuth 설정 과정에서 별도 확인이 필요했다.

---

## 11. 어떤 상황에서 적합한가

### Make가 적합한 경우

- 데이터 흐름이 복잡한 경우
- 중간 데이터를 계속 확인하며 디버깅해야 하는 경우
- 개발자가 아닌 사용자가 시각적으로 세부 로직을 관리해야 하는 경우
- Filter, Router, Aggregator를 많이 사용하는 자동화

### Activepieces가 적합한 경우

- 이미 업무 규칙이 정리되어 있고 빠르게 자동화로 옮기고 싶은 경우
- 복잡한 로직을 Code Step으로 묶어 전체 Flow를 간단하게 유지하고 싶은 경우
- AI의 도움을 받아 자동화를 빠르게 구축하거나 다른 도구에서 이전하려는 경우

### 최종 의견

이번 프로젝트에서는 **Make는 자동화 로직을 처음 설계하고 디버깅하는 과정에서 강점**이 컸고, **Activepieces는 이미 검증된 로직을 더 단순한 Flow로 재구현하는 과정에서 강점**이 컸다.

즉,

> **설계·검증에는 Make, 빠른 재구현과 단순한 운영에는 Activepieces가 유리했다.**

---

# 프로젝트 2. 자유 주제 자동화 설계 및 구현

## 12. 자동화할 반복 업무 정의

자동화 대상은 영어학원에서 매달 반복되는 **학생별 월간 학습레포트 작성 업무**이다.

기존 방식에서는 다음 작업을 사람이 반복했다.

1. 학생별 학습 달력 확인
2. 단어·작문·듣기·독해 학습범위 정리
3. 교재 변경 및 진도 확인
4. 한 달 학습량 집계
5. 학부모용 총평 작성
6. 정해진 레포트 템플릿에 내용 입력

이를 하나의 자동화로 연결해 반복적인 정리와 복사 작업을 줄이고자 했다.

---

## 13. 프로젝트 2 선정 도구

**Make**를 선정했다.

선정 이유는 다음과 같다.

- Google Drive, Google Sheets, Google Slides 연동이 가능하다.
- 복잡한 학습 데이터를 단계별로 확인하며 가공하기 편하다.
- Filter와 Aggregator를 시각적으로 확인할 수 있다.
- 실제 운영 과정에서 데이터 형식이 달라졌을 때 특정 단계만 수정하기 쉽다.
- 이번 프로젝트의 복잡한 데이터 파싱 로직을 처음 설계하고 검증하는 데 가장 적합했다.

---

## 14. 프로젝트 2 워크플로우

```text
[Google Drive]
학생 데이터 파일 등록
        ↓
[Trigger]
새 파일 감지
        ↓
[Google Sheets]
설정/달력/정보 데이터 읽기
        ↓
[Filter]
처리 대상 데이터만 통과
        ↓
[Parser / Aggregator / Variables]
과목별 학습기록 분석 및 요약
        ↓
[Gemini]
월간 총평 자동 생성
        ↓
[Google Slides]
Template Placeholder 치환
        ↓
[Result]
학생별 월간 학습레포트 자동 생성
```

---

## 15. 요구사항 충족 여부

| 요구사항 | 구현 내용 |
|---|---|
| Trigger 1개 이상 | Google Drive 새 파일 감지 |
| Action 2개 이상 | Google Sheets 읽기, 데이터 가공, Gemini, Google Slides 등 다수 |
| 조건 분기 1개 이상 | Make Filter / Activepieces Router |
| 자동 실행 | 학생 데이터 파일이 등록되면 Trigger가 실행되는 구조 |
| 실제 동작하는 Workflow | 최종 Google Slides 학습레포트 생성 확인 |
| 생성형 AI Action | Gemini를 이용한 월간 총평 자동 생성 |

---

## 16. 보너스 과제

### 보너스 1 – AI 연동 Action

**구현 완료**

Gemini를 워크플로우에 연결하여 학생의 학습 데이터와 교재 진행 상황을 입력으로 전달하고 월간 총평을 자동 생성하도록 했다.

AI가 지나치게 과장된 표현을 생성하지 않도록 기존 학원의 총평 문체를 반영한 few-shot prompt와 문체 규칙도 함께 적용했다.

### 보너스 2 – 실패 알림 및 재시도

이번 제출에서는 별도의 실패 알림 Action을 실제 구현하지 않았다.

향후 다음과 같은 방식으로 확장할 수 있다.

```text
자동화 실행 실패
      ↓
오류 정보 수집
      ↓
Gmail로 관리자에게 실패 알림
      ↓
실패한 학생 파일 ID를 별도 Google Sheet에 기록
      ↓
수정 후 재실행
```

---

## 17. 보안 및 비용 관리

- 제출 화면에는 API Key, 비밀번호, 인증 토큰을 노출하지 않는다.
- Google 계정 이메일이나 파일/폴더 ID가 노출될 경우 마스킹한다.
- Make에서는 bundle 수를 줄여 뒤쪽 Action의 불필요한 반복 실행을 방지했다.
- Gemini는 학생 1명당 한 번만 호출하도록 구성했다.
- Activepieces에서는 Router + Stop Flow를 통해 비대상 입력이 AI/Slides 단계까지 진행하지 않도록 설계했다.
- 제출용 화면을 얻기 위한 불필요한 추가 실행은 credit을 소비하므로 가능한 한 기존 구현 화면과 실제 생성 결과를 사용했다.

---

# 18. 결론

본 프로젝트에서는 실제 영어학원에서 반복되는 학생별 월간 학습레포트 작성 업무를 자동화했다.

자동화 도입 전에는 사람이 학생의 학습 기록을 확인하고, 과목별 진도를 정리하고, 교재 정보를 입력하고, 총평을 작성한 뒤 레포트 양식으로 옮겨야 했다. 자동화 후에는 학생 데이터 파일이 등록되면 데이터 분석부터 AI 총평 작성, Google Slides 레포트 생성까지 하나의 Workflow에서 처리할 수 있게 되었다.

또한 동일한 업무를 Make와 Activepieces 두 도구에서 구현하면서 자동화 플랫폼별 특징을 직접 비교했다.

- **Make:** 세밀한 시각적 설계와 디버깅에 강점
- **Activepieces:** Code Step과 AI 보조를 활용한 빠른 재구현과 단순한 Flow 구성에 강점

이번 프로젝트를 통해 단순히 자동화 도구를 사용하는 것뿐 아니라, **실제 반복 업무를 분석하고 데이터 구조를 설계한 뒤 AI까지 연결하여 실사용 가능한 자동화 시스템으로 만드는 과정**을 경험할 수 있었다.
