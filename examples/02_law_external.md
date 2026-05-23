# 예시 2: 외부망 단독 — 법령 변경 자동 분석 보고서

## 사용자 요청 (가상)
> 매주 우리 사업과 관련된 법령의 변경사항을 법제처 API에서 가져와 AI로 요약한 보고서를 만들고, 외부 컴플라이언스 위탁사에 메일로 전달하고 싶어요.

## 페르소나 협업 흐름

### [Orchestrator] 요구사항 분해
- 데이터 소스: 법제처 API (외부)
- 처리: AI 요약 (외부 LLM)
- 결과 수신자: **외부 위탁사** (회사 외부)
- 개인정보 포함 여부: 없음 (법령 본문은 공개 데이터)
- 추가 확인: 사용할 AI 모델은? 위탁사 메일 주소는 외부망에 등록 가능한가?

### [Architect] 결정 트리
- Q1 (개인정보): 아니오
- Q3 (외부 의존): 예 — 법제처 API + AI
- Q4 (수신자 = 내부 직원): 아니오 — 외부 위탁사
- **결정: 패턴 B — 외부망 단독**

컴포넌트:
| 컴포넌트 | 위치 | 기술 |
|---------|------|------|
| 주간 트리거 | 외부망 | Power Automate (Schedule) |
| 법제처 API 호출 | 외부망 | Power Automate (HTTP) |
| AI 요약 | 외부망 | AI Builder 또는 Azure OpenAI (회사 승인 모델) |
| 보고서 생성 | 외부망 | Power Automate (Word 템플릿) |
| 외부 메일 발송 | 외부망 | Power Automate (Outlook 외부망) |

### [Developer] 플로우 명세

| 순번 | 단계 | 액션 | 커넥터 | 입력 |
|-----|------|------|--------|------|
| 1 | 트리거 | Recurrence | Schedule | Frequency=Week, On=Monday, At=09:00 KST |
| 2 | API 호출 | HTTP | HTTP (Premium) | GET https://www.law.go.kr/...api... |
| 3 | JSON 파싱 | Parse JSON | Data Operations | body, schema |
| 4 | 필터 | Filter array | Built-in | 관심 법령만 추출 |
| 5 | AI 요약 | Create text with GPT | AI Builder (Premium) | 모델=회사 승인, prompt |
| 6 | Word 보고서 생성 | Populate Word template | Word Online | 템플릿 파일, 데이터 |
| 7 | 메일 발송 | Send email | Outlook | 위탁사 주소, 첨부 |

DLP 영향: HTTP, AI Builder는 Premium. 외부망 환경 라이선스 확인 필요.

### [Security] 체크리스트
- A1 외부망 환경이므로 외부 호출 허용: ✅
- A5 내부망 → 외부망 전송 없음: ✅
- B1 개인정보 미처리: ✅
- F1 회사 승인 AI 모델 사용: ⚠️ 사용자 확인 필요 — 어떤 모델을 쓰는지 명시 요청
- F2 입력 데이터에 개인정보 없음: ✅ 법령 본문만
- F3 학습 미사용 설정 확인 필요: ⚠️

**판정: 조건부 통과** — F1, F3을 명시하면 통과.

### [Orchestrator] 합의
- 패턴 B 외부망 단독 확정
- 조건: AI 모델 설정에서 "입력이 학습에 사용되지 않음" 옵션 확인, 모델명을 설계서에 명시
- Documentation으로 진행

---

## 학습 포인트

- **외부 데이터 + 외부 수신자** → 외부망 단독의 전형
- AI 사용 시 Security는 모델 자체보다 "학습 사용 여부"와 "입력 데이터 분류"를 본다
- 위탁사 주소는 외부망 Outlook 연락처에 등록 가능 (직원 정보가 아니므로 개인정보 아님)
