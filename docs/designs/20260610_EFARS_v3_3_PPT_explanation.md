# EFARS 멀티에이전트 시스템 v3.3 — PPT 설명자료

> **출처 설계서**: `20260608_EFARS_multi_agent_Azure_AI_Foundry_v3_3_internal_regulation_alignment.md`
> **목적**: 전자금융사고 보고서 자동 작성 멀티에이전트 시스템 소개 PPT용 설명 자료
> **작성일**: 2026-06-10

---

## 📌 슬라이드 1. 한 줄 소개

> **EFARS는 전자금융 사고가 발생했을 때, 사고 사실관계만 자연어로 입력하면 사내 세칙·외부 법령에 정렬된 금감원 EFARS 제출용 보고서 초안을 자동으로 작성해주는 5개 전문 AI 에이전트 협업 시스템입니다.**

- **EFARS** = 전자금융사고 보고 시스템 (Electronic Financial Accident Reporting System)
- **구축 기반**: Azure AI Foundry **A2A (Agent-to-Agent) 전용 Low-Code**
- **버전**: v3.3 (Internal Regulation Alignment Edition)

---

## 📌 슬라이드 2. 왜 이 에이전트가 필요한가? (Why)

### 🎯 현업의 4가지 페인 포인트

| # | 문제 | 현장 실태 |
| --- | --- | --- |
| 1 | **보고 시한 압박** | 사내 세칙 제11조 — 사고 인지 후 **12시간 이내** 1차 보고 필수 (외부 법령 24시간보다 강화) |
| 2 | **법령·세칙 이중 적용** | 사내 세칙(1차 근거) + 외부 법령 11종(보강 근거)을 동시에 인용해야 함 |
| 3 | **수작업 보고서의 누락·환각 위험** | 사고 유형 3분류·등급 1~4등급·책임자 4역할·긴급복구 5개 반 등 매트릭스가 복잡 |
| 4 | **반복적이고 고숙련도 요구 업무** | IT사고대응반장이 야간·휴일에도 12h 내 작성해야 함 → 휴먼 에러 발생 |

### ✅ EFARS 멀티에이전트가 주는 가치

- **자동화**: 사실관계 입력 → 분류·법령 검색·시한 산정·보고서 작성까지 **분 단위로 완료**
- **정렬성**: 사내 세칙을 **1차 근거**로 격상, 외부 법령을 보강 근거로 자동 정렬 (v3.3 신규)
- **환각 방지**: 모든 인용은 `exact_quote` 원문 그대로, 추정 시 `confidence` 자동 하향
- **임원 보고 가능**: 별지 제2호서식 `.docx` + 10섹션 임원 본문 + 220자 단문 요약 동시 생성
- **감사 추적성**: Application Insights 자동 트레이싱 + 5종 Custom Evaluator로 지속 검증

---

## 📌 슬라이드 3. 전체 토폴로지 (System Overview)

```
                       👤 사용자
                          │ (자연어 입력)
                          ▼
        ┌──────────────────────────────────┐
        │     🎯 orchestrator              │
        │     (컨트롤 타워 + A2A x 4)      │
        └──────────────────────────────────┘
              │       │       │       │
   A2A 호출   ▼       ▼       ▼       ▼
        ┌────────┐┌────────┐┌────────┐┌────────┐
        │🔍 분류 ││📚 규정 ││⏱  시한 ││📝 보고서│
        │incident││regul-  ││deadline││report-  │
        │classi- ││ation-  ││manager ││writer   │
        │fier    ││search  ││        ││         │
        └────────┘└────────┘└────────┘└────────┘
              │       │       │       │
              └───────┴───────┴───────┘
                          ▼
              📄 .docx + .txt + brief.txt
```

- **구조 원칙**: orchestrator는 라우팅만, 판정·해석·산정·작성은 4개 전문 에이전트에 위임
- **호출 방식**: A2A (Agent-to-Agent) Tool로 직접 호출, Workflow 사용 안 함
- **공통 Knowledge**: 사내 「전자금융 IT사고·장애 대응 규정」 4개 에이전트 모두 업로드

---

## 📌 슬라이드 4. 5개 에이전트 한눈에 보기 (Agent Roster)

| 에이전트 | 한 줄 정체성 | 모델 | Knowledge | 핵심 도구 |
| --- | --- | --- | --- | --- |
| 🎯 **orchestrator** | EFARS 컨트롤 타워 — 라우팅·종합·알람 | `gpt-5-mini` 또는 `gpt-4.1` | 과거 사고 케이스(선택) | A2A Tool × 4 |
| 🔍 **incident-classifier** | 한국 전자금융 사고 분류 전문가 | `gpt-4.1` | 사내 세칙 + 외부 법령 11종 | File Search |
| 📚 **regulation-search** | 사내 세칙·외부 법령 조항 검색·인용 전문가 | `gpt-4.1` | 사내 세칙 + 외부 법령 11종 | File Search |
| ⏱ **deadline-manager** | 보고 기한 산정 전문가 | `gpt-4.1` | 사내 세칙 + 시행세칙 | File Search + **Code Interpreter** |
| 📝 **report-writer** | 보고서 작성자 (별지 제2호서식 + 임원본문) | `gpt-4.1` | 사내 세칙 + 별지서식 + 시행세칙 | File Search + **Code Interpreter** |

---

## 📌 슬라이드 5. 🎯 Orchestrator — 컨트롤 타워

### 역할 (Role)
> "EFARS 컨트롤 타워. 사고 판정·법령 해석·기한 산정·보고서 작성은 일절 하지 않는다. **라우팅 + 위험 등급 알람**만 수행한다."

### 어떻게 동작하는가
1. 사용자 입력 받음 → **T0(사고 인지 시각) 누락 시 1회만 질문**
2. 결정적 `case_id` 생성 — `EFARS-{YYYYMMDD}-{HHmm}-{sha1(T0+facts)[:4]}` (재실행 시 동일 ID)
3. **🔍 incident-classifier 즉시 호출** (1순위, 안내 멘트 금지)
4. 분기 판정:
   - `is_reportable=false` → 비보고 사유 안내 후 종료
   - `is_reportable=true` → 📚 regulation-search + ⏱ deadline-manager **병렬 호출**
5. **⚠️ 1·2등급 알람** (v3.3 신규) — "비상대응위원장 최종 확정 필요" 텍스트 자동 추가
6. **⚠️ 30만원 누적 분기 안내** (v3.3 신규) — 누적 이력 확인 권고
7. 📝 report-writer 호출 → 최종 응답 포맷팅

### 지침(Instructions) 구성 — 6섹션 표준
- **# 1. ROLE** — 한 줄 정체성
- **# 2. INPUT** — 사용자 자연어 입력 구조
- **# 3. TOOLS** — A2A Tool 4개 + 호출 시점
- **# 4. PROCEDURE** — 10단계 절차 (case_id 생성 → 분류 → 분기 → 병렬 호출 → 알람 → 보고서 → 응답)
- **# 5. OUTPUT** — `[CASE]·[SEVERITY]·[ALARM]·[BRIEF]·[REPORT BODY]` 포맷
- **# 6. CONSTRAINTS** — 직접 해석 금지, 즉시 호출 의무, 금지어 목록(`~하겠습니다`, `잠시만요`), 2회 재시도 백오프

---

## 📌 슬라이드 6. 🔍 Incident-Classifier — 사고 분류 전문가

### 역할 (Role)
> "사고 사실관계를 받아 **사내 세칙을 1차 근거로**, 외부 법령을 보강 근거로 적용하여 사고 유형·등급·보고 대상 여부를 판정한다."

### 어떻게 동작하는가 (PROCEDURE 6단계)
1. **사고 유형 3분류 강제** (사내 세칙 제6조) — 시스템 장애사고 / 보안 침해사고 / 부정거래 사고
2. **보고 대상 여부 판정** (제8조) — 시스템장애 20분 이상 OR 5분+5천명 등 임계 분기
3. **제외 사유 정밀 분기** (제9조) — 1영업점 한정, 단순 공개 웹서버, 30만원 미만(누적 단서 확인)
4. **사고 등급 산정 1~4등급 강제** (제7조) — 영향범위·지속시간·피해금액 매트릭스
5. **등급 확정 권한 표시** (제7조②) — 1차 IT사고대응반장 / 2등급↑ 비상대응위원장
6. **사내 세칙 vs 외부 법령 차이 비교** — `internal_vs_external_diff` 배열 출력

### 핵심 출력 (OUTPUT)
```json
{
  "incident_type": "시스템 장애사고|보안 침해사고|부정거래 사고",
  "severity_class": "1등급|2등급|3등급|4등급",
  "severity_decision_authority": "IT사고대응반장 1차 평가 | 비상대응위원장 최종 확정 필요",
  "is_reportable": true,
  "exclusion_basis": null,
  "internal_vs_external_diff": [...],
  "citations": [{"source":"internal_regulation","article":"...","exact_quote":"..."}],
  "confidence": 0.92
}
```

### 핵심 제약 (CONSTRAINTS)
- `incident_type`은 **3개 enum 강제** / `severity_class`는 **4개 enum 강제**
- `exact_quote` 의역·축약 금지 — 원문 그대로
- 사내 세칙과 외부 법령이 다를 때 **사내 세칙 적용**, 두 기준 모두 인용
- 30만원 미만 부정거래는 **누적 판정 필수** — 누락 시 `missing_fields`에 기록

---

## 📌 슬라이드 7. 📚 Regulation-Search — 법령 인용 전문가

### 역할 (Role)
> "분류 결과를 받아 **사내 세칙을 1차 근거로**, 외부 법령을 보강 근거로 적용 조항·원문 인용·위반 시 제재·소관 기관을 함께 제시한다."

### 어떻게 동작하는가
1. **사내 세칙 11종 카테고리 우선 검색** (v3.3 신규)
   - 정의(제2조) / 유형(제6조) / 등급(제7조) / 보고의무(제8조) / 제외(제9조)
   - 채널(제10조) / 시한(제11조) / 책임체계(제12조) / 대응절차(제13·15조)
   - 복구조직(제16조) / 사후관리(제18조) / 보존(제20조)
2. **외부 법령 보강 검색** — 키워드 OR 검색
   - 사후 조치: "통보·신고·보고·점검·재발방지·원인분석·공시·개선계획"
   - 제재: "과태료·과징금·징역·벌금·행정처분·시정명령·공시명령·영업정지"
3. **소관 기관(agency) 추출** — 금감원/개인정보위/KISA/한국신용정보원 등
4. **자체 강화 기준 표시** — `is_internal_strengthened=true` + `external_baseline` 인용
5. **법령 우선순위 적용** — 사내 세칙 > 시행세칙 > 감독규정 > 시행령 > 모법

### 핵심 출력
```json
{
  "applicable_clauses": [
    {
      "source": "internal_regulation",
      "law": "사내 세칙",
      "article": "제11조",
      "category": "기한",
      "exact_quote": "...",
      "interpretation": "본 사고에 어떻게 적용되는지",
      "actionable": true,
      "penalty": {"description":"...","exact_quote":"..."},
      "agency": "금융감독원",
      "is_internal_strengthened": true,
      "external_baseline": "관행적 24시간"
    }
  ],
  "primary_basis": "사내 세칙 제11조"
}
```

### 핵심 제약
- `exact_quote` 원문 그대로 (띄어쓰기·문장부호 포함)
- `penalty`·`agency`는 검색에서 찾은 것만 — **자체 추정 금지**
- 사내 세칙이 최상위 — `primary_basis`는 사내 세칙 우선

---

## 📌 슬라이드 8. ⏱ Deadline-Manager — 시한 산정 전문가

### 역할 (Role)
> "T0(사고 인지 시각)와 분류 결과를 받아 **사내 세칙 제11·18조의 보고 단계별 기한**을 산정한다."

### 어떻게 동작하는가 — 4가지 시한 산정
| 보고 단계 | 시한 | 근거 |
| --- | --- | --- |
| **1차 보고** | T0 + **12시간** (외부 법령 24h보다 강화) | 사내 세칙 제11조 |
| **경과 보고** | (A) 중대 변동 시 즉시 / (B) 인지일 + 45일 + 매 90일 | 사내 세칙 제11조 |
| **종료 보고** | 조치 완료 시점 (datetime 산출 불가, 조건부 트리거) | 사내 세칙 제11조 |
| **원인분석 보고서** | 종료일 + **21일** 이내 IT운영부장 제출 | 사내 세칙 제18조 |

### Code Interpreter 활용
- Python `datetime` + `zoneinfo("Asia/Seoul")`로 타임존 일관 보장
- T0의 시각 형식 정상성 검증
- baseline_comparison 자동 생성 — 사내 vs 외부 차이를 표로 출력

### 핵심 출력
```json
{
  "T0_kst": "2026-06-08T09:22:00+09:00",
  "initial_due": "2026-06-08T21:22:00+09:00",
  "initial_basis": "사내 세칙 제11조 — T0 + 12시간",
  "initial_channel_options": ["EFARS (원칙)", "서면·팩스·유선 (부득이 시)"],
  "interim_first_due": null,
  "interim_basis": "사고 변동 시 즉시",
  "final_due": null,
  "final_trigger_condition": "모든 조치 완료 + 정상 업무 수행 가능",
  "rca_report_due_offset_days": 21,
  "quarterly_followup": true
}
```

### 핵심 제약
- 모든 datetime은 **Asia/Seoul 타임존 + ISO 8601** 형식
- 1차보고 시한 반드시 **T0 + 12시간** (사내 세칙 우선)
- 경과보고 (B) 트리거 단서가 없으면 `false` + `confidence` 하향
- `final_due`는 datetime이 아닌 조건부 트리거 → `null` 출력

---

## 📌 슬라이드 9. 📝 Report-Writer — 보고서 작성자

### 역할 (Role)
> "분류·규정·기한 3개 결과를 받아 별지 제2호서식 초안과 **사내 세칙 정렬 10섹션 임원 보고용 본문**을 생성한다. IT운영부장·비상대응위원장·CISO·경영진에게 직접 보고된다."

### 어떻게 동작하는가 — 10섹션 임원 본문 자동 생성

```
§0. 임원 요약 (Executive Summary) — 6~8줄
§1. 사고 개요 (타임라인·영향·유형·등급·등급확정권한)
§2. 사고 영향 분석 (4축: 업무·고객·재무·규제·평판) — 정량 금액 금지, 정성 3등급만
§3. 사고 원인 분석 (초기 가설 라벨 의무) — 직접/Kill Chain/근본 원인 후보
§4. 적용 법령 및 의무사항 (적용 조항 표 / 보고 의무 매트릭스 / 자체 강화 비교)
§5. 사고 선포 및 대응 절차
   §5-0. 사고 선포 6단계 체크리스트 (세칙 제15조)
   §5-1. 완료된 조치 (대응 우선순위 4단계 — 세칙 제13조)
   §5-A. 긴급복구 5개 반 (총괄지휘·복구실행·현업지원·대외협력·법무준법 — 세칙 제16조)
§6. 후속 보고 일정 및 책임 체계
   §6-1. 보고 일정 (1차 12h / 경과 45+90일 / 종료 / 21일 RCA)
   §6-2. 책임 체계 (IT운영부장·IT사고대응반장·준법감시부서·비상대응위원장 — 세칙 제12조)
§7. 결정 요청 사항 (To 경영진·비상대응위원장)
부록 A. 인용된 법령·세칙 원문 (사내 세칙 우선 정렬)
부록 B. 자체 점검 (self_check_score 12항목)
부록 C. 사후관리 (원인분석 21일 / 분기별 이행 점검 / 5년 보존)
```

### Code Interpreter 활용
- `python-docx`로 별지 제2호서식 `.docx` 자동 생성
- `.txt` 백업 본문 + 220자 단문 요약 `brief.txt` 동시 생성
- 모든 판정 항목에 인라인 `[근거: 사내 세칙 제○조 — "exact_quote 앞 30자..."]` 자동 삽입

### 핵심 제약
- **정량 금액 생성 금지** — §2 영향 분석은 정성 3등급(저/중/고)만 허용
- **§3 원인 분석 모든 항목 첫 줄에 "초기 가설 — 포렌식 결과로 확정 필요" 라벨 의무**
- `responsible_role`은 **사내 세칙 4역할 enum**만 사용
- `self_check_score` 12항목 자체 점검 후 < 0.75면 본문 재검토
- 본 보고서는 "초안" — 외부 제출 전 IT운영부장·준법감시부서 검토 필요 1줄 명시

---

## 📌 슬라이드 10. 공통 지침(Instructions) 표준 — 6섹션

> **모든 에이전트가 동일한 6섹션 구조**로 지침을 작성. 일관성·유지보수성·평가 용이성 확보.

| 섹션 | 의미 | 작성 원칙 |
| --- | --- | --- |
| `# 1. ROLE` | 한 줄로 정체성 정의 | "당신은 ~ 전문가다" 한 문장 |
| `# 2. INPUT` | 받게 되는 데이터의 형식·필드 | A2A 호출 시 orchestrator가 전달하는 메시지 구조 명시 |
| `# 3. TOOLS` | 사용 가능 도구와 각 도구를 언제·어떻게 쓸지 | 도구별 발화 트리거 명시 |
| `# 4. PROCEDURE` | 단계별 절차 | 번호 목록 1→2→3, 분기 조건 명시 |
| `# 5. OUTPUT` | 출력 형식 (반드시 JSON 스키마) | 추가 텍스트 금지 명시 |
| `# 6. CONSTRAINTS` | 금지·예외 처리·신뢰도 하한 | "X 외 외부 지식 금지", "근거 못 찾으면 confidence 낮추기" |

---

## 📌 슬라이드 11. 간략한 실행 흐름 (End-to-End Flow)

### 시나리오 예시
> *"2026-06-08 09:22 KST에 여신심사 PC가 랜섬웨어에 감염되어 업무 중단되었습니다."*

```
1) [사용자] → orchestrator
   ├ "사고 사실관계 + T0 = 2026-06-08 09:22 KST"

2) orchestrator → case_id 생성 → 🔍 incident-classifier (즉시 호출)
   ├ File Search: 사내 세칙 제6·7·8·9조 → 외부 법령 보강
   └ 출력: incident_type="보안 침해사고", severity_class="2등급",
            severity_decision_authority="비상대응위원장 최종 확정 필요",
            is_reportable=true

3) orchestrator → ⚠️ 2등급 알람 텍스트 준비
                  → 📚 regulation-search ‖ ⏱ deadline-manager (병렬 호출)

   ├─ regulation-search:
   │    사내 세칙 11종 카테고리 + 외부 법령(보안 침해 시 개인정보보호법 제34조 등)
   │    → applicable_clauses (사내 1차 / 외부 보강), penalty·agency 포함
   │
   └─ deadline-manager:
        1차=T0+12h=2026-06-08 21:22 KST / 경과=45+90일 / 종료=조건부 / RCA=21일
        → timeline

4) orchestrator → 📝 report-writer (case_id + 4개 입력 전달)
   ├ 10섹션 본문 생성 + 별지 제2호서식 .docx + .txt + brief.txt
   └ 모든 판정 항목에 인라인 [근거: 사내 세칙 제○조] 자동 삽입

5) orchestrator → 사용자 최종 응답
   [CASE] EFARS-20260608-0922-a1b2
   [SEVERITY] 2등급 · IT사고대응반장 책임 운영
   [ALARM] ⚠️ 비상대응위원장 최종 확정 + 사고 선포 결정 필요
   [KEY DEADLINES] 1차보고 2026-06-08 21:22 KST 등
   [REPORT FILES] .docx / .txt / .brief
   [BRIEF] 220자 단문 요약
   [REPORT BODY] 10섹션 전체 본문
   [FOLLOW UP ACTIONS] · [DECISIONS REQUESTED] · [ASSUMPTIONS NEEDED]
```

---

## 📌 슬라이드 12. v3.3의 핵심 차별점 — 사내 세칙 정렬

### 외부 법령 ↔ 사내 세칙, 기준이 다를 때

| 항목 | 외부 법령(관행) | 사내 세칙 (강화) | 적용 |
| --- | --- | --- | --- |
| 1차보고 시한 | 24시간 | **12시간** | 사내 세칙 적용 + 둘 다 인용 |
| 사고 등급 | 정의 없음 | **1~4등급 명시** | 사내 세칙 적용 |
| 등급 확정 권한 | 정의 없음 | **2등급↑ 비상대응위원장** | 사내 세칙 적용 |
| 책임자 | CISO 등 | **IT운영부장·IT사고대응반장·준법감시부서** | 사내 세칙 적용 |
| 사후관리 | 정의 미세 | **종료+21일 원인분석 / 5년 보존** | 사내 세칙 적용 |

### 우선순위 원칙 (세칙 제5조 — 자체 강화 기준 운영)
> **사내 세칙 > 시행세칙 > 감독규정 > 시행령 > 모법**
>
> 단, 외부 법령이 더 짧은 시한을 요구하면(예: 개인정보보호위원회 72시간) → 짧은 쪽 적용

---

## 📌 슬라이드 13. 품질 보증 — 검증·관찰

### Evaluator 매트릭스
- **공통**: Groundedness Pro · Task Adherence · Tool Call Accuracy
- **v3.3 신규 Custom Evaluator 5종**
  | 이름 | 합격 기준 |
  | --- | --- |
  | `internal_regulation_priority` | 사내 세칙 인용 ≥ 1건 AND primary_basis가 사내 세칙 → 90%↑ |
  | `severity_enum_strict` | 3유형·4등급 enum 강제 → 100% |
  | `initial_due_12h` | initial_due == T0+12h (±5분) → 100% |
  | `responsibility_enum` | follow_up_actions의 4역할 enum → 95%↑ |
  | `severity_alarm_coverage` | 1·2등급 시 알람 텍스트 출력 → 100% |

### 관찰
- Azure **Application Insights** 자동 트레이싱 — 모든 A2A 호출·도구 호출·지연 시간 캡처
- 지속적 평가(Continuous Evaluation) 설정 가능

---

## 📌 슬라이드 14. 요약 — 한 페이지 정리

### 🎯 무엇을 하는가
> 전자금융 사고 사실관계 → **12시간 내 EFARS 제출용 보고서 초안**(별지서식 + 임원본문) 자동 작성

### 🏗 어떻게 구성되어 있는가
- **5개 에이전트** (orchestrator 1 + 전문 4)
- **Azure AI Foundry A2A Low-Code** — Workflow 없이 포털 UI + A2A Tool만으로 구축
- **6섹션 표준 Instructions** (ROLE·INPUT·TOOLS·PROCEDURE·OUTPUT·CONSTRAINTS)

### 📋 지침의 핵심 원칙
- **사내 세칙 1차 근거** + 외부 법령 보강
- **enum 강제** (사고 유형 3·등급 4·책임자 4)
- **exact_quote 원문 인용** + 인라인 [근거: ...]
- **정량 금액 금지** + "초기 가설" 라벨 의무
- **self_check_score 12항목 자체 점검**

### 🔁 동작 흐름
1. 분류 (3유형·1~4등급·보고대상 판정)
2. 규정 검색 (사내 11종 카테고리 + 외부 보강)
3. 시한 산정 (12h / 45+90일 / 21일)
4. 보고서 작성 (10섹션 + .docx + .txt + brief)
5. 최종 응답 (1·2등급 알람 + 30만원 누적 안내)

### ✨ v3.3의 가치
- 사내 세칙 13개 조항을 **자동으로 1차 적용**
- 외부 법령과의 차이를 자동 비교·노출
- 비상대응위원장·IT운영부장 책임 체계를 보고서에 자동 반영

---

## 부록. PPT 슬라이드 구성 권장 순서

| # | 슬라이드 제목 | 본 문서 섹션 |
| --- | --- | --- |
| 1 | 표지 — "EFARS 멀티에이전트 v3.3" | 슬라이드 1 |
| 2 | 왜 필요한가 — 4가지 페인 포인트 | 슬라이드 2 |
| 3 | 시스템 토폴로지 | 슬라이드 3 |
| 4 | 5개 에이전트 한눈에 보기 | 슬라이드 4 |
| 5 | 🎯 Orchestrator 상세 | 슬라이드 5 |
| 6 | 🔍 Incident-Classifier 상세 | 슬라이드 6 |
| 7 | 📚 Regulation-Search 상세 | 슬라이드 7 |
| 8 | ⏱ Deadline-Manager 상세 | 슬라이드 8 |
| 9 | 📝 Report-Writer 상세 (10섹션) | 슬라이드 9 |
| 10 | 공통 지침 6섹션 표준 | 슬라이드 10 |
| 11 | 실행 흐름 시나리오 (랜섬웨어 예시) | 슬라이드 11 |
| 12 | v3.3 핵심 차별점 — 사내 세칙 정렬 | 슬라이드 12 |
| 13 | 품질 보증 — Evaluator | 슬라이드 13 |
| 14 | 요약 한 페이지 | 슬라이드 14 |
