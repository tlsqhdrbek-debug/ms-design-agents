# Copilot Studio 토픽 명세 템플릿

Developer 페르소나가 Copilot Studio 토픽을 설계할 때 사용. Copilot Studio를 사용하지 않는 설계에서는 생략.

---

## 1. 에이전트 개요

| 항목 | 값 | 비고 |
|------|----|------|
| 에이전트명 | {{ 에이전트명 }} | |
| 위치 | 외부망 / 내부망 | |
| 채널 | Teams / Web Chat / 기타 | |
| 오케스트레이션 | 생성형 (Generative) / 클래식 | 권장: 생성형 |
| 사용 LLM | 회사 승인 모델명 | 외부 모델 사용 시 사전 승인 필수 |

---

## 2. 토픽 목록

### 2.1 시스템 토픽 (수정 사항)
| 시스템 토픽 | 수정 내용 |
|-----------|----------|
| {{ 예: Greeting }} | 환영 메시지를 회사명 포함으로 변경 |
| {{ 예: Fallback }} | 답변 불가 시 관리자 안내 추가 |

### 2.2 커스텀 토픽

| 토픽명 | 트리거 구문 또는 설명 | 엔티티 사용 | 호출 액션 (플로우/커넥터) | 변수 |
|-------|---------------------|------------|--------------------------|------|
| {{ 법령조회 }} | "법령을 알려줘", "~ 법령 알려줘" / 법령 정보를 조회 | 법령명 (사용자 정의) | LawSearchFlow | lawName, lawText |
| {{ 신청결과확인 }} | "내 신청 상태" / 사용자의 신청 진행 상태 확인 | 신청번호 | GetApprovalStatus | requestId, status |

---

## 3. 토픽별 상세 (필요 시)

### 3.1 {{ 토픽명 }}

**용도**: ...

**트리거**:
- 생성형 모드: 토픽 설명 — "이 토픽은 ~할 때 사용"
- 클래식 모드: 트리거 구문 목록

**대화 흐름**:
```
1. 사용자 발화 인식
2. Question 노드: "어떤 법령을 조회할까요?"
   → 엔티티: 법령명 (사용자 정의)
   → 변수 저장: Topic.lawName
3. Action 호출: LawSearchFlow
   → 입력: lawName
   → 출력: lawText, lastUpdated
4. Send a message: "{{lawName}} 법령 내용입니다. (마지막 갱신: {{lastUpdated}})"
5. Send adaptive card: 법령 본문
6. 종료
```

**오류 처리**:
- LawSearchFlow 실패 시: "법령 정보를 가져오지 못했습니다. 잠시 후 다시 시도해주세요."

---

## 4. 엔티티 정의

### 4.1 시스템 엔티티 사용
- {{ 예: BooleanPrebuiltEntity }}
- {{ 예: DateTimePrebuiltEntity }}

### 4.2 사용자 정의 엔티티
| 엔티티명 | 종류 | 값 목록 또는 정규식 |
|---------|------|---------------------|
| {{ 법령명 }} | List | 개인정보보호법, 전자금융거래법, ... |
| {{ 부서코드 }} | RegEx | `^[A-Z]{2}\d{3}$` |

---

## 5. 호출하는 외부 액션

| 액션 종류 | 이름 | 매개변수 | 반환값 |
|----------|------|---------|--------|
| Power Automate Flow | LawSearchFlow | lawName (string) | lawText (string), lastUpdated (datetime) |
| Connector Action | SharePoint - Get item | siteUrl, itemId | item (object) |

---

## 6. 인증·인가

- 채널 인증: Teams SSO / Web Chat 익명 / 인증 토큰 등
- 사용자 식별: User.Id, User.Email 등 사용 여부
- 권한 분기: 역할별 토픽 노출 여부

---

## 7. 생성형 AI 사용 시

- 사용 모델: ...
- RAG 데이터 소스: SharePoint, Dataverse 등 (URL/위치)
- 답변 가드레일: 컨텐츠 필터, 출처 표시 여부
- 입력·출력 데이터 분류: 개인정보 포함 여부

---

## 8. 채널 게시

- 게시 채널: Teams / Web / 기타
- 게시 환경: 개발 / 스테이징 / 운영
- 게시 절차: 검토 → 승인 → 게시
