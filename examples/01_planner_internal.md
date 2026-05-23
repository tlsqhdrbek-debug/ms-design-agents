# 예시 1: 내부망 단독 — Planner 업무 관리 에이전트

## 사용자 요청 (가상)
> 우리 팀 Planner의 미완료 업무를 매일 아침에 정리해서 Teams로 알려주는 에이전트를 만들고 싶어요.

## 페르소나 협업 흐름

### [Orchestrator] 요구사항 분해
- 데이터 소스: Planner (내부 SaaS, M365 내부)
- 알림 대상: 팀원 (내부 직원)
- 외부 API/데이터: 불필요
- 개인정보 처리: 직원 이름·사번 등이 알림에 포함될 가능성 있음
- 추가 확인: 알림 형식은 개인별 또는 팀 전체?

### [Architect] 결정 트리 적용
- Q1 (개인정보): 예 — 직원별 미완료 작업 포함
- Q2 (외부 의존): 아니오
- **결정: 패턴 A — 내부망 단독**

컴포넌트:
| 컴포넌트 | 위치 | 기술 |
|---------|------|------|
| 일정 트리거 | 내부망 | Power Automate (Schedule) |
| Planner 조회 | 내부망 | Power Automate (Planner 커넥터) |
| 메시지 구성 | 내부망 | Power Automate (Compose) |
| Teams 알림 | 내부망 | Power Automate (Teams 커넥터) |

### [Developer] 플로우 명세

| 순번 | 단계 | 액션 | 커넥터 | 입력 | 출력 |
|-----|------|------|--------|------|------|
| 1 | 트리거 | Recurrence | Schedule | Frequency=Day, At=08:30, TZ=KST | - |
| 2 | Planner 조회 | List tasks | Planner | Group Id, Plan Id | tasks |
| 3 | 필터 | Filter array | Built-in | tasks의 percentComplete < 100 | openTasks |
| 4 | 메시지 조립 | Compose | Built-in | HTML 표 형식 | message |
| 5 | Teams 전송 | Post message in chat or channel | Teams | Team, Channel, message | - |

사용 커넥터: 모두 Standard, 내부망 사용 가능.

### [Security] 체크리스트
- A1 외부 호출 없음: ✅
- A2 외부망 접근 없음: ✅
- B1 개인정보 외부망 처리: ✅ 내부망에서만 처리
- B4 최소 수집: ⚠️ Planner 작업의 담당자 이름만 가져옴 (사번 미포함)
- E2 감사 로그: ⚠️ Power Automate 기본 실행 이력만 사용 (별도 감사 불필요)

**판정: 조건부 통과** — 작업 담당자 이름이 Teams 채널에 노출됨. 채널 멤버 범위 확인 필요.

### [Orchestrator] 합의
- 패턴 A 내부망 단독 확정
- 조건: Teams 채널 멤버를 팀원으로 한정 (외부 게스트 미포함 확인)
- Documentation으로 진행

### [Documentation]
→ `/docs/designs/YYYYMMDD_Planner_업무관리.docx`로 산출 (구조는 templates/design_document.md)

---

## 학습 포인트

- **외부 의존 없음 + 데이터·수신자 모두 내부** → 패턴 A의 전형
- 개인정보가 있어도 내부망 단독이면 별도 마스킹 불필요
- 조건부 통과는 흔하다. "운영 시 채널 멤버 검증" 같은 항목으로 통과 가능
