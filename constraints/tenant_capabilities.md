# 테넌트별 가능·불가 액션 (참조 매트릭스)

외부망/내부망 각 테넌트에서 어떤 Power Automate 액션 및 Copilot Studio 기능이 사용 가능한지 정리.

> 본 매트릭스는 일반적 기준이며, 실제 회사 DLP 정책 및 라이선스 상태에 따라 변동될 수 있다. 의심스러우면 회사 Power Platform 관리자에게 확인할 것.

---

## 1. Power Automate 커넥터

### 외부망 M365에서 사용 가능
| 커넥터 | 등급 | 비고 |
|--------|------|------|
| HTTP | Premium | 외부 REST API 호출 (가장 빈번) |
| HTTP with Microsoft Entra ID | Premium | 인증 필요한 외부 API |
| RSS | Standard | 외부 RSS 피드 |
| SharePoint | Standard | 외부망 사이트만 |
| Outlook (Office 365) | Standard | 외부망 메일박스 |
| Teams | Standard | 외부망 Teams (내부 직원과 다름) |
| AI Builder | Premium | 외부 모델 호출 (사전 승인 필요) |
| OpenAI / Azure OpenAI | Premium | 외부 LLM (사전 승인 필요) |
| Translator | Standard | 외부 번역 API |
| Excel Online (Business) | Standard | 외부망 OneDrive |

### 내부망 M365에서 사용 가능
| 커넥터 | 등급 | 비고 |
|--------|------|------|
| SharePoint | Standard | 내부망 사이트 |
| Outlook (Office 365) | Standard | 내부망 메일박스 |
| Teams | Standard | 내부 직원 Teams |
| Planner | Standard | 내부 업무 관리 |
| OneDrive for Business | Standard | 내부망 |
| Forms | Standard | 내부 설문 |
| Excel Online (Business) | Standard | 내부망 |
| Approvals | Standard | 내부 결재 |
| Schedule | Standard | 일정 기반 트리거 |
| Data Operations (Compose 등) | Standard | 내장 |
| Variable | Standard | 내장 |

### 내부망에서 사용 불가 (인터넷 차단)
- ❌ HTTP, HTTP Webhook
- ❌ RSS
- ❌ 외부 SaaS 커넥터 (Twitter, Salesforce 등)
- ❌ 외부 AI Builder 모델 호출
- ❌ OpenAI / 외부 LLM 직접 호출
- ❌ 외부 Translator
- ❌ Custom Connector (외부 호스트 대상)

---

## 2. Copilot Studio 기능

### 외부망 Copilot Studio
- 생성 AI 답변 (외부 LLM 기반): 가능 (사전 승인된 모델)
- 외부 데이터 소스 연동: 가능
- 외부 커넥터/HTTP 노드 호출: 가능

### 내부망 Copilot Studio
- 생성 AI 답변: **회사 승인 내부 모델만**
- 데이터 소스: SharePoint, Dataverse 등 내부 SaaS만
- HTTP 노드, 외부 커넥터: ❌ 불가

---

## 3. 연계 채널

### 외부망 → 내부망 단방향 전송 (승인된 방식)
- 회사 게이트웨이를 통한 데이터 전달 (회사별 명칭 상이, 관리자 확인 필수)
- 메일을 통한 텍스트 전송 (제목·본문에 개인정보 없을 것)
- 일부 회사는 별도 데이터 이송 시스템(예: 망연계 솔루션) 운영

### 내부망 → 외부망 전송
- **원칙적으로 금지**
- 회사 정책상 특정 비민감 정보(공지사항 등)에 한해 가능할 수 있으나 본 설계 시스템에서는 다루지 않음 (Architect는 "불가"로 간주)

---

## 4. 페르소나별 사용 지침

- **Architect**: 본 매트릭스 보고 컴포넌트 배치 가능 여부 1차 판단
- **Developer**: 사용 커넥터가 해당 테넌트에서 가능한지 확인
- **Security**: 사용 불가 커넥터가 설계에 포함되어 있으면 Veto
- **Documentation**: 설계서 "사용 커넥터" 절에 등급·테넌트 명시

---

## 5. 매트릭스 갱신

새 커넥터 출시, DLP 정책 변경, 회사 승인 모델 추가 시 본 문서를 갱신할 것. Microsoft Learn MCP의 `microsoft_docs_search`로 최신 정보 확인 가능.
