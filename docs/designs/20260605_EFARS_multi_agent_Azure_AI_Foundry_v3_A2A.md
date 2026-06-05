# EFARS 멀티에이전트 — Azure AI Foundry **A2A 전용** Low-Code 설계서 v3.0

> 코드 없이 **Foundry 포털 UI** + **A2A Tool**만으로 구축. Workflow 사용 안 함.
> 작성일: 2026-06-05 (KST) · 설계 버전: v3.0 (A2A Edition)
> 이전 버전: [v2.0 Workflow Edition](./20260605_EFARS_multi_agent_Azure_AI_Foundry_v2_LowCode.md) (참고용으로 유지)

---

## 0. v3.0이 v2.0과 다른 점 (요약)

| 항목 | v2.0 | **v3.0 (본 문서)** |
| --- | --- | --- |
| 에이전트 간 통신 | Foundry Workflow 비주얼 디자이너 (결정적 sequential 흐름) | **A2A Tool**만 사용 (orchestrator의 LLM이 자율 라우팅) |
| Orchestrator 정체 | Workflow 객체 (코드 없음) | **별도 Prompt Agent** (Instructions로 라우팅 정책 명시) |
| 전문 에이전트 노출 방식 | Workflow 노드로 invoke | 각 에이전트가 **Incoming A2A endpoint**로 노출 |
| 결정권 | Workflow가 노드 순서 강제 | Orchestrator가 계속 컨트롤 유지 (*Agent A keeps control*) |
| Code Interpreter 설명 | 간략 언급 | **§4에서 동작 원리·자동 코드 예시·에이전트별 필요 여부** 명시 |
| Instructions 구조 | 자유 형식 | **6개 표준 섹션**으로 통일 (§3) |

> A2A 패턴 핵심: *"When Agent A calls Agent B through the A2A tool, Agent B's answer goes back to Agent A. Agent A then summarizes the answer and generates a response for the user. **Agent A keeps control** and continues to handle future user input."* — [Connect to an A2A agent endpoint](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/agent-to-agent)

---

## 1. 전체 토폴로지 (A2A 전용)

```mermaid
flowchart TB
    User([👤 사용자]) -->|채팅| Orch
    subgraph FoundryProject["🏭 Foundry Project"]
        Orch[["🎯 orchestrator<br/>(Prompt Agent)<br/>+ A2A Tool x 4<br/>+ File Search (사고 케이스 컨텍스트)"]]
        IC[["🔍 incident-classifier<br/>(Prompt Agent)<br/>Incoming A2A endpoint"]]
        RS[["📚 regulation-search<br/>(Prompt Agent)<br/>Incoming A2A endpoint"]]
        DM[["⏱ deadline-manager<br/>(Prompt Agent)<br/>Incoming A2A endpoint<br/>+ Code Interpreter"]]
        RW[["📝 report-writer<br/>(Prompt Agent)<br/>Incoming A2A endpoint<br/>+ Code Interpreter"]]

        Orch -- A2A call --> IC
        Orch -- A2A call --> RS
        Orch -- A2A call --> DM
        Orch -- A2A call --> RW
        IC -. A2A 응답 .-> Orch
        RS -. A2A 응답 .-> Orch
        DM -. A2A 응답 .-> Orch
        RW -. A2A 응답 .-> Orch
    end
```

**호출 규칙**: orchestrator가 사용자 입력을 받고 → Instructions에 명시된 순서/조건에 따라 → 4개 A2A Tool 중 필요한 것만 호출 → 각 응답을 받아 다음 호출에 사용 → 마지막에 최종 보고서를 사용자에게 반환.

---

## 2. 사전 준비

| 항목 | 값 |
| --- | --- |
| 포털 | <https://ai.azure.com> (New Foundry 토글 ON) |
| 필요 권한 | 프로젝트 **Contributor** 이상, **Foundry User** 롤 |
| 모델 배포 | `gpt-4.1` 또는 `gpt-5-mini` |
| Application Insights | 프로젝트에 연결 (자동 트레이싱) |
| 리전 | Code Interpreter 가능 리전 선택 ([지원 리전](https://learn.microsoft.com/azure/foundry/reference/region-support)) |

---

## 3. Instructions 6섹션 표준 (모든 에이전트 공통 구조)

본 설계의 모든 에이전트는 다음 6섹션으로 Instructions를 작성합니다. 섹션 헤더(`# 1. ROLE` 등)는 그대로 사용하세요. 모델이 섹션 단위로 일관되게 추론합니다.

| 섹션 | 의미 | 작성 원칙 |
| --- | --- | --- |
| `# 1. ROLE` | 한 줄로 정체성 정의 | "당신은 ~ 전문가다" 한 문장 |
| `# 2. INPUT` | 받게 되는 데이터의 형식·필드 | A2A 호출 시 orchestrator가 전달하는 메시지 구조 명시 |
| `# 3. TOOLS` | 사용 가능 도구와 각 도구를 언제·어떻게 쓸지 | 도구별 발화 트리거 명시 (예: "기한 계산 시 Code Interpreter로 datetime 코드 작성") |
| `# 4. PROCEDURE` | 단계별 절차 | 번호 목록 1→2→3, 분기 조건 명시 |
| `# 5. OUTPUT` | 출력 형식 (반드시 JSON 스키마) | 추가 텍스트 금지 명시 |
| `# 6. CONSTRAINTS` | 금지·예외 처리·신뢰도 하한 | "X 외의 외부 지식 금지", "근거 못 찾으면 confidence 낮추기" |

이 구조를 따르면 §7 검증 단계에서 Evaluator가 일관된 평가를 할 수 있습니다.

---

## 4. Code Interpreter — 어떤 코드를 넣는가?

### 4-1. 핵심 원리 (오해 방지)

| 흔한 오해 | 실제 동작 |
| --- | --- |
| 사용자가 미리 Python 코드를 등록해 둔다 | ❌ |
| **모델이 런타임에 필요한 Python 코드를 자동 작성·실행한다** | ✅ |

Foundry Code Interpreter는 **Microsoft 관리형 sandbox**(Azure Container Apps dynamic sessions)에서 모델이 작성한 코드를 격리 실행합니다. Hyper-V 경계 isolation. 출처: [Code Interpreter — Sandboxed execution environment](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/code-interpreter#sandboxed-execution-environment)

**한 줄 요약**: *"The agent's Foundry model writes and executes code for data analysis, chart generation, and iterative problem-solving tasks."*

### 4-2. 동작 특성

| 특성 | 값 | 영향 |
| --- | --- | --- |
| 언어 | Python | numpy, pandas, matplotlib, python-docx 등 사전 설치 |
| 세션 수명 | 최대 1시간 (idle 30분) | 장기 작업 시 청크 분할 |
| 네트워크 | **outbound 차단** | 외부 API 호출 불가 → orchestrator를 거쳐야 함 |
| 격리 | 대화별 별도 세션 | 동시 대화 = 동시 세션 |
| 파일 | 첨부 파일 읽기 가능, 차트·docx 생성 후 다운로드 링크 반환 | 본 설계에서 report-writer가 활용 |
| 추가 비용 | 토큰비 외 별도 과금 | Code Interpreter 활성 에이전트만 켜기 |

### 4-3. 본 설계 에이전트별 Code Interpreter 필요 여부

| 에이전트 | Code Interpreter | 이유 |
| --- | --- | --- |
| orchestrator | ❌ | 라우팅·요약은 텍스트 추론으로 충분 |
| incident-classifier | ❌ | 분류 판정은 RAG + 텍스트 추론. 결정성 필요 없음 |
| regulation-search | ❌ | 검색·인용은 File Search 결과 그대로 사용 |
| **deadline-manager** | ✅ | **날짜·시간 산술은 결정적이어야 함**. LLM이 문장으로 날짜 계산하면 오류 발생. 모델이 `datetime` 코드를 자동 작성하여 정확히 계산 |
| **report-writer** | ✅ | **.docx 파일 생성**. 모델이 `python-docx` 코드를 자동 작성하여 별지 제2호서식 렌더링 |

### 4-4. 모델이 자동 생성할 코드 예시 (참고용 — 직접 입력 안 함)

**deadline-manager에서 모델이 런타임에 만드는 코드 (예시)**:
```python
# 모델이 자동으로 작성하는 코드 — 사용자는 입력하지 않음
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
import json

T0 = datetime.fromisoformat("2026-06-05T14:30:00+09:00")
holidays = {"2026-06-06", "2026-08-15"}  # Knowledge에서 모델이 파싱

def next_business_day(dt):
    while dt.strftime("%Y-%m-%d") in holidays or dt.weekday() >= 5:
        dt += timedelta(days=1)
    return dt

initial = T0 + timedelta(hours=2)
interim = T0 + timedelta(hours=24)
final   = next_business_day(T0 + timedelta(days=30))

print(json.dumps({
    "initial_due": initial.isoformat(),
    "interim_due": interim.isoformat(),
    "final_due":   final.isoformat(),
}, indent=2))
```

**report-writer에서 모델이 런타임에 만드는 코드 (예시)**:
```python
# 모델이 자동 생성 — 사용자는 입력하지 않음
from docx import Document

doc = Document("/mnt/data/별지_제2호서식_템플릿.docx")  # Knowledge 업로드 파일

# orchestrator가 전달한 변수로 placeholder 치환
mapping = {
    "{사고개요}": user_facts,
    "{사고분류}": classification["incident_type"],
    "{보고기한_최초}": timeline["initial_due"],
    "{인용조항}": "\n".join(c["exact_quote"] for c in regulations),
}
for p in doc.paragraphs:
    for k, v in mapping.items():
        if k in p.text:
            p.text = p.text.replace(k, v)

out = "/mnt/data/EFARS_보고서_초안.docx"
doc.save(out)
print(out)
```

> 두 코드 모두 **사용자가 입력하지 않음**. Instructions의 PROCEDURE 섹션에 "Code Interpreter로 ~을 계산/생성하라"고 자연어로 지시하면 모델이 자동으로 위와 같은 코드를 작성·실행하고 결과를 반환합니다.

### 4-5. Code Interpreter를 켜는 포털 절차

1. 에이전트 상세 페이지 → 우측 **Setup** 패널
2. **Tools** 섹션 → **Add** → **Code Interpreter** → 저장
3. (선택) **Knowledge** 또는 **Files** 섹션에서 분석 대상 파일 첨부 → Code Interpreter가 sandbox `/mnt/data/`에서 접근 가능

---

## 5. 5개 에이전트 — 포털 클릭 가이드

각 에이전트는 **Build > Agents > Create agent** 에서 생성. Knowledge 업로드 시 File Search vector store가 자동 생성 (chunk 800 tokens / embed `text-embedding-3-large` / 하이브리드 검색). 출처: [File search tool — How file search works](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/file-search#how-file-search-works)

---

### 5-1. 🔍 incident-classifier

| 항목 | 값 |
| --- | --- |
| 모델 | `gpt-4.1` |
| Knowledge (업로드 파일) | **11종 법령 PDF** (§5-1-A 표) |
| Tools | `File Search` (Knowledge 추가 시 자동 활성화) |
| Code Interpreter | ❌ |
| Incoming A2A endpoint | ✅ 활성화 (§6 절차) |

#### 5-1-A. 업로드할 파일 (11종)
| # | 파일명 | 출처 |
| --- | --- | --- |
| 1 | `01_전자금융거래법.pdf` | 국가법령정보센터 |
| 2 | `02_전자금융거래법_시행령.pdf` | 〃 |
| 3 | `03_전자금융감독규정.pdf` | 금융위원회 |
| 4 | `04_전자금융감독규정_시행세칙.pdf` | 금감원 (제7조의4 포함) |
| 5 | `05_금융분야_개인정보보호_가이드라인.pdf` | 금융위·금감원 |
| 6 | `06_개인정보보호법.pdf` | 국가법령정보센터 |
| 7 | `07_신용정보법.pdf` | 〃 |
| 8 | `08_정보통신망법.pdf` | 〃 |
| 9 | `09_금융회사_정보처리_위탁규정.pdf` | 〃 |
| 10 | `10_FSI_정보보호표준.pdf` | 금융보안원 |
| 11 | `11_한국은행법_제29조_관련.pdf` | 한국은행 |

#### 5-1-B. Instructions (그대로 복사-붙여넣기)

```text
# 1. ROLE
당신은 한국 전자금융사고 분류 전문가다. 사고 사실관계를 받아 보고 대상 여부와
사고 유형을 판정한다.

# 2. INPUT
A2A로 다음 형식의 자연어 메시지를 받는다:
- user_facts: 사용자가 진술한 사고 사실관계 (자유 텍스트)
- (선택) hints: orchestrator가 부가한 컨텍스트

# 3. TOOLS
- File Search: Knowledge의 11종 법령에서 근거 조항을 검색한다.
  사용 시점: 모든 판정의 근거를 찾을 때 반드시 호출.
  사용 방법: incident_type 후보별로 1회씩 검색 + 시행세칙 제7조의4 항상 검색.

# 4. PROCEDURE
1. 사고 사실관계를 정리하여 다음 후보군 중 어디에 해당하는지 가설을 세운다:
   {전산장애, 정보유출, 전자적_침해행위, 기타}
2. File Search로 다음을 순차 호출한다:
   2-1. "전자금융감독규정 시행세칙 제7조의4" (1순위)
   2-2. 가설된 incident_type 관련 정의·보고의무 조항
   2-3. 개인정보 침해이면 개인정보보호법·신용정보법·정보통신망법 신고 의무
3. 검색된 원문 인용을 그대로 보관한다 (의역 금지).
4. 종합 판정:
   - is_reportable: 시행세칙 제7조의4 + 관련법을 충족하면 true
   - severity_class: 중대 / 일반 (시행세칙 기준에 따름)
   - confidence: 인용 명확도 0.0~1.0
5. 애매하면 보수적으로 보고 대상(true)으로 처리한다.

# 5. OUTPUT
반드시 다음 JSON만 반환 (추가 텍스트 금지):
{
  "is_reportable": true | false,
  "exclusion_basis": "비보고 사유 + 근거 조항" | null,
  "incident_type": "전산장애" | "정보유출" | "전자적_침해행위" | "기타",
  "severity_class": "중대" | "일반",
  "confidence": 0.0~1.0,
  "citations": [
    {"law":"...", "article":"...", "exact_quote":"...", "source_file":"..."}
  ],
  "missing_evidence": ["추가 확인 필요한 사실관계 목록"]
}

# 6. CONSTRAINTS
- 11종 법령 외의 외부 지식으로 판정 금지.
- exact_quote는 원문 그대로. 의역·축약·재구성 금지.
- File Search에서 근거를 못 찾으면 confidence를 0.5 이하로 낮추고 missing_evidence 채울 것.
- 사용자에게 추가 질문하지 말 것. 추가 정보 필요시 missing_evidence에만 기록.
- 출력은 한국어로 작성하되 JSON 키는 영문 그대로.
```

---

### 5-2. 📚 regulation-search

| 항목 | 값 |
| --- | --- |
| 모델 | `gpt-4.1` |
| Knowledge | **incident-classifier와 동일한 11종 법령 PDF** (다시 업로드) |
| Tools | `File Search` |
| Code Interpreter | ❌ |
| Incoming A2A endpoint | ✅ |

#### Instructions

```text
# 1. ROLE
당신은 한국 전자금융 법규의 조항 검색·인용 전문가다. 분류 결과를 받아 적용 조항과
원문 인용을 제시한다.

# 2. INPUT
A2A로 다음 자연어 메시지를 받는다:
- classification: incident-classifier의 JSON 출력 (incident_type, severity_class, citations 등)
- user_facts: 원본 사고 사실관계

# 3. TOOLS
- File Search: Knowledge의 11종 법령에서 조항을 검색한다.
  사용 시점: 카테고리별(정의/보고의무/기한/사후조치/예외)로 최소 1회씩.

# 4. PROCEDURE
1. classification.incident_type을 키로 다음 카테고리를 각각 File Search로 검색한다:
   - 정의 조항
   - 보고 의무 조항
   - 보고 기한 조항
   - 사후 조치 조항
   - 적용 예외·단서 조항
2. 검색 결과에서 원문 그대로(exact_quote) 추출한다.
3. 충돌 시 우선순위: 시행세칙 > 감독규정 > 시행령 > 모법.
4. 본 보고의 핵심 근거 조항 1개를 primary_basis로 선정한다.
5. relevance_score < 0.6인 항목은 제외한다.

# 5. OUTPUT
반드시 다음 JSON만 반환:
{
  "applicable_clauses": [
    {
      "law": "법령명",
      "article": "제○조 제○항 제○호",
      "category": "정의" | "보고의무" | "기한" | "사후조치" | "예외",
      "exact_quote": "원문 그대로",
      "interpretation": "본 사고에 어떻게 적용되는지 1-2문장 해설",
      "source_file": "파일명",
      "relevance_score": 0.0~1.0
    }
  ],
  "primary_basis": "본 보고의 핵심 근거 조항 (1개)",
  "conflict_resolution": "법령 충돌 시 우선순위 적용 사유" | null
}

# 6. CONSTRAINTS
- exact_quote 의역·축약·재구성 금지 (띄어쓰기·문장부호 포함 원문 그대로).
- File Search 결과에 없는 조항은 출력에서 제외 (null로 채우지 말 것).
- relevance_score는 인용의 적합도이며 LLM 자체 평가가 아닌 File Search 점수 기반.
- 사용자에게 추가 질문 금지.
```

---

### 5-3. ⏱ deadline-manager

| 항목 | 값 |
| --- | --- |
| 모델 | `gpt-4.1` |
| Knowledge | `03_전자금융감독규정.pdf`, `04_전자금융감독규정_시행세칙.pdf`, `holidays_kr_2026.json` |
| Tools | `File Search` + **`Code Interpreter` ✅** |
| Code Interpreter 용도 | **모델이 datetime 산술 코드를 자동 작성·실행** (§4-4 예시 참조) |
| Incoming A2A endpoint | ✅ |

#### Instructions

```text
# 1. ROLE
당신은 전자금융사고 보고 기한 계산기다. T0(사고 인지 시각)와 사고 유형을 받아
최초/중간/종결 보고 기한을 KST ISO 8601로 산정한다.

# 2. INPUT
A2A로 다음 자연어 메시지를 받는다:
- T0_kst: ISO 8601 KST (예: "2026-06-05T14:30:00+09:00")
- classification: incident-classifier의 JSON 출력 (incident_type, severity_class)

# 3. TOOLS
- File Search: 시행세칙에서 사고 유형별 보고 기한 조항을 찾는다.
- Code Interpreter: datetime 산술과 영업일 보정을 결정적으로 수행한다.
  사용 시점: 기한 계산은 반드시 Code Interpreter로 처리 (텍스트 추론 금지).
  사용 방법: Python으로 datetime + zoneinfo + 공휴일 목록을 사용한 코드를 작성하라.

# 4. PROCEDURE
1. File Search로 시행세칙에서 다음 시한을 찾는다:
   - 최초 보고 시한 (보통 즉시 또는 2시간)
   - 중간 보고 시한 (보통 24시간)
   - 종결 보고 시한 (보통 30일)
2. holidays_kr_2026.json을 Code Interpreter로 로드한다.
3. Code Interpreter로 다음을 계산하라 (Python 코드를 직접 작성·실행):
   - T0를 Asia/Seoul 타임존으로 파싱
   - 각 시한 더하기 (timedelta)
   - 시행세칙이 "영업일 기준"으로 명시한 경우만 주말·공휴일 보정
   - "지체 없이"는 보정하지 말고 raw_phrase에 원문 표현 그대로 기록
4. 계산 결과를 OUTPUT JSON에 매핑한다.

# 5. OUTPUT
반드시 다음 JSON만 반환:
{
  "T0_kst": "ISO 8601",
  "initial_due": {
    "deadline_kst": "ISO 8601",
    "basis_article": "시행세칙 제○조",
    "raw_phrase": "원문 시한 표현"
  },
  "interim_due": { ... 동일 구조 },
  "final_due":   { ... 동일 구조 },
  "holiday_adjustments": ["YYYY-MM-DD: 공휴일/주말 → 다음 영업일로 이월"],
  "computation_log": "Code Interpreter가 실행한 계산 단계 요약"
}

# 6. CONSTRAINTS
- 시한 산정은 Code Interpreter 결과만 사용. 텍스트 추론으로 날짜 만들지 말 것.
- 시행세칙에 명시 없는 사고 유형이면 해당 필드를 missing_basis 객체로 표시:
  {"deadline_kst": null, "missing_basis": "시행세칙 제○조에 해당 사고유형 시한 명시 없음"}
- "지체 없이" 같은 정성 표현은 KST 시각을 만들지 말고 raw_phrase에 원문 그대로 둘 것.
```

---

### 5-4. 📝 report-writer

| 항목 | 값 |
| --- | --- |
| 모델 | `gpt-4.1` |
| Knowledge | `별지_제2호서식_템플릿.docx`, `04_전자금융감독규정_시행세칙.pdf` |
| Tools | `File Search` + **`Code Interpreter` ✅** |
| Code Interpreter 용도 | **모델이 python-docx 코드를 자동 작성·실행하여 별지 제2호서식 .docx 렌더링** |
| Incoming A2A endpoint | ✅ |

#### Instructions

```text
# 1. ROLE
당신은 전자금융사고 보고서 작성자다. 분류·규정·기한 3개 결과를 받아 별지 제2호서식
초안을 생성한다.

# 2. INPUT
A2A로 다음 자연어 메시지를 받는다:
- user_facts: 사고 사실관계
- classification: incident-classifier의 JSON 출력
- regulations: regulation-search의 JSON 출력
- timeline: deadline-manager의 JSON 출력

# 3. TOOLS
- File Search: 별지 제2호서식 템플릿의 필수 필드 구조를 파악한다.
- Code Interpreter: python-docx로 .docx 파일을 렌더링한다.
  사용 시점: OUTPUT의 report_docx_path를 만들 때 반드시 사용.
  사용 방법: Python으로 Document 로드 → placeholder 치환 → 저장.

# 4. PROCEDURE
1. File Search로 별지 제2호서식의 필수 필드 목록을 추출한다.
2. 입력 4종을 각 필드에 매핑한다:
   - 사고개요 ← user_facts
   - 사고분류 ← classification.incident_type, severity_class
   - 보고대상_판단근거 ← classification.citations + regulations.primary_basis
   - 적용규정 ← regulations.applicable_clauses (category별 그룹화)
   - 보고기한 ← timeline.initial_due / interim_due / final_due
3. 모든 인용은 regulations.applicable_clauses[].exact_quote를 원문 그대로 사용.
4. Code Interpreter로 다음 Python 코드를 작성·실행한다:
   - /mnt/data/별지_제2호서식_템플릿.docx 로드
   - placeholder 치환
   - /mnt/data/EFARS_보고서_초안_{case_id}.docx로 저장
5. 자체 검증 체크리스트 점수를 산출한다:
   [ ] 모든 인용에 source_file 있음
   [ ] T0와 기한 timezone 일관 (KST)
   [ ] is_reportable=true인데 본문 비어있지 않음
   [ ] severity_class가 본문과 일치
   self_check_score = 통과 개수 / 4
6. 누락 필드는 missing_fields에 나열 (보고서 본문은 생성 진행).

# 5. OUTPUT
반드시 다음 JSON만 반환:
{
  "report_markdown": "본문 미리보기 (markdown)",
  "report_docx_path": "Code Interpreter 산출 경로",
  "missing_fields": ["..."],
  "self_check_score": 0.0~1.0,
  "citations_used": [
    {"law":"...","article":"...","exact_quote":"...","source_file":"..."}
  ]
}

# 6. CONSTRAINTS
- 인용문 의역·축약·재구성 금지.
- 사용자에게 추가 입력 요청 금지 → 누락 시 missing_fields에 기록.
- Code Interpreter는 외부 네트워크 호출 불가 → 모든 데이터는 입력으로 받은 것만 사용.
- self_check_score < 0.75이면 OUTPUT 직전에 한 번 본문을 재검토한 후 다시 계산.
```

---

### 5-5. 🎯 orchestrator (컨트롤 타워, A2A 라우터)

| 항목 | 값 |
| --- | --- |
| 모델 | `gpt-5-mini` (reasoning 권장) 또는 `gpt-4.1` |
| Knowledge | (선택) 과거 사고 케이스 .jsonl 한 파일 — 비슷한 케이스 컨텍스트용 |
| Tools | **`A2A Tool` × 4** + (선택) `File Search` |
| Code Interpreter | ❌ (요약·라우팅만 수행) |
| Incoming A2A endpoint | 선택 — 외부에서도 EFARS 시스템을 호출하게 하려면 활성화 |

#### Tools에 연결할 A2A 4종

다음 4개 A2A connection을 미리 만들어두고 (포털 절차 §6 참조), orchestrator의 **Setup > Tools > Add > Custom > Agent2Agent (A2A)** 에서 각각 추가합니다.

| 연결명 (Name) | 가리킬 endpoint | 설명(=Tool description, 라우팅 단서로 사용됨) |
| --- | --- | --- |
| `a2a_incident_classifier` | incident-classifier의 A2A endpoint URL | "사고 사실관계를 분류하고 보고 대상 여부를 판정. 항상 가장 먼저 호출." |
| `a2a_regulation_search` | regulation-search의 A2A endpoint URL | "분류 결과를 받아 적용 조항과 원문 인용을 추출. is_reportable=true일 때 호출." |
| `a2a_deadline_manager` | deadline-manager의 A2A endpoint URL | "T0와 분류를 받아 보고 기한을 계산. is_reportable=true일 때 regulation-search와 병행 호출 가능." |
| `a2a_report_writer` | report-writer의 A2A endpoint URL | "분류·규정·기한 3개 결과를 받아 별지 제2호서식 .docx 초안 생성. 가장 마지막에 호출." |

> Tool description은 **LLM이 어떤 도구를 언제 부를지 결정하는 핵심 단서**입니다. 위 description을 그대로 입력하세요. 출처: [Tool best practice](https://learn.microsoft.com/azure/foundry/agents/concepts/tool-best-practice)

#### Instructions

```text
# 1. ROLE
당신은 EFARS(전자금융사고 보고 시스템) 컨트롤 타워다. 사용자의 사고 입력을 받아
4개 전문 에이전트(A2A 도구)를 정해진 순서·조건으로 호출하고 최종 보고서를 사용자에게
반환한다.

# 2. INPUT
사용자가 자연어로 다음을 입력한다:
- 사고 사실관계 (필수)
- T0 = 사고 인지 시각 KST (없으면 사용자에게 1회 질문하여 받음)
- 추가 맥락 (선택)

# 3. TOOLS
당신이 사용할 도구는 4개의 A2A 도구뿐이다. 각 도구를 정확한 시점에 호출하라.

[3-1] a2a_incident_classifier
  - 언제: 사용자 입력 직후 항상 1순위로 호출.
  - 입력 메시지 형식:
      "user_facts: <사용자 사실관계>"
  - 받는 출력: classification JSON

[3-2] a2a_regulation_search
  - 언제: classification.is_reportable == true 일 때.
  - 입력 메시지 형식:
      "classification: <classification JSON>
       user_facts: <사용자 사실관계>"
  - 받는 출력: regulations JSON

[3-3] a2a_deadline_manager
  - 언제: classification.is_reportable == true 일 때. regulation-search와 병행 호출 가능.
  - 입력 메시지 형식:
      "T0_kst: <ISO 8601>
       classification: <classification JSON>"
  - 받는 출력: timeline JSON

[3-4] a2a_report_writer
  - 언제: 위 3개 결과가 모두 도착한 뒤 마지막에 1회 호출.
  - 입력 메시지 형식:
      "user_facts: ...
       classification: ...
       regulations: ...
       timeline: ..."
  - 받는 출력: report JSON

# 4. PROCEDURE
1. 사용자 입력 정규화. T0가 없으면 "사고를 인지하신 정확한 시각(KST)을 알려주세요"라고
   1회만 묻고 입력을 받는다.
2. case_id 생성: "EFARS-YYYYMMDD-HHmm-<랜덤4자리>" 형식. (LLM이 직접 생성)
3. a2a_incident_classifier 호출 → classification 수신.
4. 분기:
   - classification.is_reportable == false → STEP 5로 가서 비보고 메모만 작성.
   - classification.is_reportable == true → STEP 6로 진행.
5. (비보고 경로) 한 단락 메모를 직접 작성하여 사용자에게 반환:
   "본 사고는 다음 사유로 보고 대상이 아닙니다: <classification.exclusion_basis>"
   그리고 END.
6. (보고 경로) 다음 두 호출을 순서대로 또는 가능하면 동시에 수행한다:
   - a2a_regulation_search → regulations 수신
   - a2a_deadline_manager → timeline 수신
7. a2a_report_writer 호출 → report 수신.
8. 사용자에게 최종 응답:
   - case_id
   - 보고서 docx 경로
   - 핵심 기한 3개 (initial/interim/final)
   - self_check_score
   - missing_fields (있으면 빨간 글씨)
9. self_check_score < 0.75 이거나 missing_fields가 비어있지 않으면 사용자에게
   "검토가 필요합니다" 표시.

# 5. OUTPUT
사용자에게 반환하는 최종 메시지는 다음 구조로 작성:

[CASE] <case_id>
[STATUS] 보고대상 (또는 비보고)
[CLASSIFICATION] <incident_type> / <severity_class> / confidence=<...>
[KEY DEADLINES]
  - 최초: <initial_due>
  - 중간: <interim_due>
  - 종결: <final_due>
[REPORT FILE] <report_docx_path>
[SELF CHECK] <self_check_score>/1.0
[NEEDS REVIEW] <missing_fields 또는 없음>
[CITATIONS COUNT] <citations 수>

# 6. CONSTRAINTS
- 4개 A2A 도구 외의 도구로 사고 판정·인용·기한 산정 금지.
- 직접 법령을 해석하거나 기한을 계산하지 말 것 — 전문 에이전트에 위임.
- A2A 호출 순서를 임의로 바꾸지 말 것 (분류 → 규정+기한 → 보고서).
- 사용자에게 같은 정보를 두 번 묻지 말 것.
- 어떤 A2A 호출이 실패하면 1회 재시도 후 사용자에게 명시적으로 보고하고 종료.
- 비보고 판정도 사용자에게 사유를 명확히 설명할 것.
- 최종 응답에 자체 추측을 넣지 말고 전문 에이전트의 출력만 종합할 것.
```

---

## 6. A2A 연결 만들기 — 포털 클릭 절차

### 6-1. 각 전문 에이전트를 Incoming A2A endpoint로 노출

4개 전문 에이전트 각각에 대해 반복:

1. 포털에서 해당 에이전트 상세 페이지 열기 (예: incident-classifier)
2. 우측 **Setup** 패널 → 하단 **Incoming A2A** 섹션 펼치기
3. **Enable incoming A2A** 토글 ON
4. **A2A protocol version**: `1.0` (신규는 1.0 권장)
5. 발급된 **A2A endpoint URL**과 인증 키(또는 Entra ID 옵션)를 메모

> Prompt agents는 incoming A2A를 기본 지원. 출처: [Enable incoming A2A on a Foundry agent (preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/enable-agent-to-agent-endpoint)

### 6-2. Orchestrator에서 A2A connection 만들기

4개 전문 에이전트 각각에 대해 반복:

1. 좌측 **Tools** 메뉴 클릭
2. **Connect tool** 클릭
3. **Custom** 탭 선택
4. **Agent2Agent (A2A)** 선택 → **Create**
5. 입력:
   - **Name**: `a2a_incident_classifier` (위 §5-5 표의 이름 그대로)
   - **A2A Agent Endpoint**: §6-1에서 메모한 URL
   - **Authentication**: 키 기반이면 credential name `x-api-key` + secret 값 / Entra ID이면 해당 옵션
6. **Save**

출처: [Connect to an A2A agent endpoint — Create the connection in the Foundry portal](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/agent-to-agent#create-the-connection-in-the-foundry-portal)

### 6-3. Orchestrator 에이전트에 A2A 도구 부착

1. Build > Agents > **orchestrator** 선택
2. 우측 **Setup** → **Tools** → **Add**
3. **Custom** → **Agent2Agent (A2A)** → §6-2에서 만든 connection 4개를 각각 선택해서 추가
4. 각 도구의 **Description** 칸에 §5-5의 "라우팅 단서" 문장 그대로 입력 (LLM이 도구 선택할 때 사용)
5. **Save**

---

## 7. 검증·관찰 (이전 버전과 동일)

| 항목 | 포털 경로 |
| --- | --- |
| Thread Logs (메시지 백로그) | Build > Agents > 에이전트 > Playground > 우측 **Thread logs** |
| 분산 트레이스 (A2A 호출 타임라인) | 좌측 **Tracing** → 트레이스 선택 (Application Insights 연동) |
| 실시간 메트릭 | 좌측 **Monitor** → Agent Monitoring Dashboard |
| Continuous Evaluation | Monitor > Settings > **Continuous evaluation** → evaluator 체크박스 (각 에이전트별 §7-1) |
| 사전 배포 평가 | **Evaluations** > Create new evaluation > 골든셋 업로드 |
| Guided Guardrails | 에이전트 상세 > 좌측 **Guardrails** > **Guided guardrails setup** |
| AI Red Teaming | 좌측 **AI red teaming agent** > New scan |
| Prompt Optimizer | 에이전트 Instructions 편집 화면 우측 상단 ✏️✨ 아이콘 |

### 7-1. 에이전트별 권장 Evaluator (Monitor > Settings에서 체크)

| 에이전트 | Evaluators |
| --- | --- |
| orchestrator | Intent Resolution · Task Adherence · Task Completion · Tool Call Accuracy |
| incident-classifier | Groundedness Pro · Intent Resolution · Tool Call Accuracy |
| regulation-search | Groundedness Pro · Relevance · Response Completeness |
| deadline-manager | Tool Call Accuracy · Task Adherence (Code Interpreter 결과 정합성) |
| report-writer | Task Completion · Task Adherence · Groundedness |

출처: [Agent evaluators](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/agent-evaluators), [Set up continuous evaluation](https://learn.microsoft.com/azure/foundry/observability/how-to/how-to-monitor-agents-dashboard#set-up-continuous-evaluation)

---

## 8. 한눈에 보는 실행 흐름 (예시)

```mermaid
sequenceDiagram
    autonumber
    actor U as 사용자
    participant O as 🎯 orchestrator<br/>(Prompt Agent)
    participant IC as 🔍 incident-classifier
    participant RS as 📚 regulation-search
    participant DM as ⏱ deadline-manager
    participant RW as 📝 report-writer

    U->>O: "오늘 14:30 KST 인터넷뱅킹 1시간 장애..."
    Note over O: PROCEDURE 1-2: T0 확인, case_id 생성

    O->>IC: A2A a2a_incident_classifier<br/>user_facts: ...
    IC->>IC: File Search × 3<br/>(7조의4 + 감독규정 + 모법)
    IC-->>O: classification JSON<br/>is_reportable=true, 전산장애, 중대

    par 병렬 호출
        O->>RS: A2A a2a_regulation_search<br/>classification + user_facts
        RS->>RS: File Search × 5<br/>(정의/보고/기한/사후/예외)
        RS-->>O: regulations JSON
    and
        O->>DM: A2A a2a_deadline_manager<br/>T0 + classification
        DM->>DM: File Search<br/>+ Code Interpreter (datetime)
        DM-->>O: timeline JSON<br/>initial/interim/final
    end

    O->>RW: A2A a2a_report_writer<br/>4개 입력 종합
    RW->>RW: File Search (서식)<br/>+ Code Interpreter (python-docx)
    RW-->>O: report JSON<br/>docx_path + self_check=0.95

    O-->>U: 최종 응답<br/>[CASE] EFARS-... [REPORT] .../initial: ...
```

---

## 9. 자주 묻는 질문

**Q1. Orchestrator의 LLM이 호출 순서를 잘못 정하면?**
A2A 도구는 "LLM 자율 라우팅"이므로 Instructions의 PROCEDURE 섹션이 호출 순서를 강제하는 핵심입니다. §5-5의 PROCEDURE를 그대로 사용하면 LLM이 분류 → 규정+기한 → 보고서 순서를 정확히 따릅니다. Tool Call Accuracy evaluator로 지속 검증하세요.

**Q2. 호출 실패 시 자동 재시도?**
Foundry는 A2A 호출에 기본 retry 정책을 갖습니다(연결 오류 시). 비즈니스 로직 재시도(예: 응답이 JSON이 아닐 때)는 Instructions의 CONSTRAINTS에 "1회 재시도 후 사용자에게 보고"로 명시되어 있습니다.

**Q3. Code Interpreter 결과를 다른 에이전트가 받을 수 있나요?**
deadline-manager의 Code Interpreter는 JSON 텍스트로 결과를 반환하고, 그 JSON이 A2A 응답으로 orchestrator에 전달됩니다. report-writer는 timeline JSON을 입력으로 받습니다. 즉, **파일은 직접 공유되지 않고 JSON으로 직렬화하여 전달**됩니다. report-writer의 .docx 파일은 사용자에게만 반환됩니다.

**Q4. A2A는 preview인데 운영 환경에 써도 되나요?**
2026-06 현재 A2A Tool은 public preview이며 SLA가 없습니다. 운영 도입 전 다음을 권장합니다: ① Continuous Evaluation으로 품질 지속 측정 ② AI Red Teaming 정기 실행 ③ 핵심 경로는 Human-in-the-loop 게이트 추가. 출처: [Connect to an A2A agent endpoint (preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/agent-to-agent)

**Q5. Code Interpreter가 외부 데이터를 못 가져오나요?**
네. *"dynamic sessions can't make outbound network requests"* — sandbox는 outbound 네트워크 차단입니다. 모든 외부 데이터는 Knowledge 파일 업로드 또는 A2A 호출 입력으로 주입되어야 합니다.

---

## 10. 검증 출처 (Microsoft Learn 1차 문서)

1. [Connect to an A2A agent endpoint from Foundry Agent Service (preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/agent-to-agent) — A2A vs Workflow, Tools > Connect tool > Custom > A2A 포털 절차
2. [Enable incoming A2A on a Foundry agent (preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/enable-agent-to-agent-endpoint) — Incoming A2A endpoint 활성화, A2A protocol v1.0
3. [Agent2Agent (A2A) authentication](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-to-agent-authentication) — 인증 방식 (key / Entra ID / OAuth)
4. [Code Interpreter tool for Microsoft Foundry agents](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/code-interpreter) — 모델이 코드 자동 생성·실행, sandbox, outbound 차단
5. [File search tool for agents](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/file-search) — Knowledge 자동 인덱싱
6. [Tool best practice](https://learn.microsoft.com/azure/foundry/agents/concepts/tool-best-practice) — Tool description 작성법
7. [Agent evaluators](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/agent-evaluators)
8. [Set up continuous evaluation](https://learn.microsoft.com/azure/foundry/observability/how-to/how-to-monitor-agents-dashboard#set-up-continuous-evaluation)
9. [Set up tracing in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/observability/how-to/trace-agent-setup)

---

## 11. 변경 이력

| 버전 | 일자 | 내용 |
| --- | --- | --- |
| v1.0 | 2026-06-05 | 코드 기반 (Foundry SDK Python) |
| v2.0 | 2026-06-05 | Low-code (Workflow 비주얼 디자이너 기반) |
| **v3.0** | **2026-06-05** | **A2A 전용**: Workflow 제거, 모든 통신 A2A Tool로. Instructions 6섹션 표준화. Code Interpreter 동작 명확화 (사용자 코드 미입력, 런타임 자동 생성). 에이전트별 도구 필요 여부 명시. |
