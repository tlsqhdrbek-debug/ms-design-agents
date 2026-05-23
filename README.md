# MS Design Agents

Microsoft Power Automate + Copilot Studio 기반 업무 자동화 및 AI Agent **설계서를 자동 산출**하는 멀티 페르소나 에이전트 시스템.

사용자가 "~한 자동화/에이전트를 만들고 싶어"라고 요청하면 Claude Code가 5개 페르소나를 차례로 또는 병렬로 연기하면서 협업해 **금융권 망분리 규정을 준수하는 최적의 설계서**를 산출한다.

---

## 1. 시스템 개요

```
사용자 요청
   ↓
[Orchestrator] ── 요구사항 분해, 페르소나 호출 순서 결정
   ↓
[Architect] ── 외부망/내부망/연계 배치 결정
   ↓
[Developer] ── Power Automate 플로우, Copilot Studio 토픽 설계
   ↓
[Security] ── 망분리·개인정보보호법 준수 검토 (Veto 권한)
   ↓
[Orchestrator] ── 충돌 시 재설계 루프 (최대 3회)
   ↓
[Documentation] ── 최종 설계서 (.docx) 작성
   ↓
/docs/designs/YYYYMMDD_<프로젝트명>.docx
```

---

## 2. 페르소나 구성

| # | 페르소나 | 역할 | 정의 파일 |
|---|---------|------|-----------|
| 1 | **Architect** | 전체 아키텍처, 외부망/내부망 배치 결정 | [personas/architect.md](personas/architect.md) |
| 2 | **Developer** | Power Automate 액션·커넥터, Copilot Studio 토픽 설계 | [personas/developer.md](personas/developer.md) |
| 3 | **Security** | 금융권 망분리, 개인정보보호법 검토 | [personas/security.md](personas/security.md) |
| 4 | **Documentation** | 설계서·다이어그램·구현 가이드 작성 | [personas/documentation.md](personas/documentation.md) |
| 5 | **Orchestrator** | 의견 조율, 합의 도출, 재설계 루프 관리 | [personas/orchestrator.md](personas/orchestrator.md) |

### 왜 Orchestrator를 추가했는가

원 요구사항(4개 페르소나)에서 한 명을 추가한 이유:

1. **충돌 중재 필요** — Architect와 Security가 충돌할 때(예: "외부 API가 필요" vs "외부망 사용 불가") 누군가가 최종 판단을 내려야 함
2. **무한 루프 방지** — 합의가 안 되는 케이스에서 "3회 재설계 후 사용자 에스컬레이션" 같은 정책 집행자가 필요
3. **사용자 인터페이스** — 사용자에게 모호한 요구사항을 되묻거나 트레이드오프를 제시하는 단일 창구 역할

4명만 두면 Security의 거부권이 토론을 멈춰버리거나, 결론 없이 끝없이 의견을 주고받는 문제가 생긴다. Orchestrator는 어떤 페르소나의 의견을 채택할지 결정하는 메타 역할이다.

---

## 3. 회사 환경 제약 (필수)

본 시스템은 **금융권 망분리 환경**에서 작동한다. 모든 설계는 다음 제약을 반드시 준수해야 한다.

| 테넌트 | 인터넷 통신 | 사용 데이터 | 주요 용도 |
|--------|------------|------------|-----------|
| **외부망 M365** | 가능 | 외부 API, 웹, AI Builder 외부 호출 | 법제처 API, 외부 데이터 수집 |
| **내부망 M365** | **불가** | M365 내부 데이터만 (Teams, SharePoint, Planner, Outlook, OneDrive) | 개인정보 처리, 내부 업무 자동화 |

**핵심 규칙:**
- 개인정보·민감정보는 반드시 내부망에서만 처리
- SaaS 클라우드에 개인정보 저장 금지
- 외부망 → 내부망 연계는 승인된 게이트웨이/일방향 전송 등 망분리 규정 준수 방식만 허용

자세한 내용은 [constraints/](constraints/)를 참조.

---

## 4. 설계 분기 로직

| 조건 | 설계 위치 |
|------|-----------|
| 외부 인터넷·API 불필요, M365 내부 데이터만 사용 | 내부망 M365 단독 |
| 외부 API/데이터가 필수 | 외부망 M365 단독 |
| 외부 데이터를 수집해 내부 직원에게 전달 필요 | 외부망 → 내부망 연계 (보안 방안 명시) |

자세한 결정 트리는 [workflow/decision_tree.md](workflow/decision_tree.md) 참조.

---

## 5. 사용법 (Claude Code)

### 5.1 새 설계 요청 처리 절차

1. 사용자가 자동화/에이전트 요구사항을 자연어로 제시
2. Claude Code는 다음을 순서대로 수행:
   1. [workflow/collaboration_protocol.md](workflow/collaboration_protocol.md)에 정의된 협업 프로토콜을 로드
   2. Orchestrator 페르소나로 요구사항 분해 및 명확화
   3. Architect → Developer → Security 순으로 페르소나 전환하며 설계 진행
   4. 충돌 발생 시 [workflow/consensus_process.md](workflow/consensus_process.md)에 따라 재설계
   5. Documentation 페르소나로 최종 설계서 작성
3. 산출물을 `/docs/designs/YYYYMMDD_<프로젝트명>.docx`로 저장

### 5.2 페르소나 전환 명시

Claude Code가 작업할 때 채팅 내에서 현재 어떤 페르소나로 발언하고 있는지를 명시한다:

```
[Architect] 이 요구사항은 외부 법제처 API 호출이 필요하므로 외부망 배치가 필요합니다...
[Security] 그러나 결과를 내부 직원에게 전달하려면 망분리 규정상...
[Orchestrator] Architect 의견을 우선하되 Security의 망연계 조건을 반영합시다...
```

### 5.3 산출물 형식

- **1차 산출**: 마크다운으로 설계서 초안 작성 (`/docs/designs/YYYYMMDD_<프로젝트명>.md`)
- **최종 산출**: docx 변환 (`/docs/designs/YYYYMMDD_<프로젝트명>.docx`)
- 변환 방법은 [tools/docx_generation.md](tools/docx_generation.md) 참조

---

## 6. 디렉토리 구조

```
ms-design-agents/
├── README.md                          # 본 문서
├── CLAUDE.md                          # Claude Code 세션 진입 가이드
├── personas/                          # 5개 페르소나 정의
│   ├── architect.md
│   ├── developer.md
│   ├── security.md
│   ├── documentation.md
│   └── orchestrator.md
├── constraints/                       # 환경 제약사항
│   ├── network_separation.md          # 금융권 망분리 규정
│   ├── compliance.md                  # 개인정보보호법 준수 사항
│   └── tenant_capabilities.md         # 외부망/내부망 가능·불가 액션
├── workflow/                          # 에이전트 협업 정의
│   ├── collaboration_protocol.md      # 발언 형식, 호출 순서
│   ├── decision_tree.md               # 망 배치 결정 로직
│   └── consensus_process.md           # 합의 도출, 재설계 루프
├── templates/                         # 산출물 템플릿
│   ├── design_document.md             # 설계서 전체 템플릿
│   ├── flow_specification.md          # Power Automate 플로우 명세
│   ├── copilot_topic.md               # Copilot Studio 토픽 명세
│   └── security_checklist.md          # 보안 검토 체크리스트
├── examples/                          # 시나리오 예시
│   ├── 01_planner_internal.md         # 내부망 단독 케이스
│   ├── 02_law_external.md             # 외부망 단독 케이스
│   └── 03_law_bridge.md               # 외부망→내부망 연계 케이스
├── tools/
│   └── docx_generation.md             # 마크다운 → docx 변환 가이드
└── docs/
    └── designs/                       # 산출된 설계서 저장 위치
```

---

## 7. 향후 확장

- **검토자 페르소나**: 사용자가 명시적으로 "검토 모드"를 요청하면 이미 작성된 설계서를 비판적으로 재검토하는 별도 페르소나 추가 가능
- **운영자 페르소나**: 배포 후 모니터링·운영 가이드 작성 담당 추가 가능

페르소나 추가 시 `personas/`에 정의 파일을 만들고 본 README의 페르소나 표와 [workflow/collaboration_protocol.md](workflow/collaboration_protocol.md)를 업데이트할 것.

---

## 8. 참고 리소스

- **Microsoft Learn MCP** — 최신 공식 문서 (Claude Code에서 자동 연결)
- **Power Platform 커뮤니티** — https://powerusers.microsoft.com/
- **Power Automate 커넥터 카탈로그** — https://learn.microsoft.com/connectors/connector-reference/
