# Power Automate 플로우 명세 템플릿

Developer 페르소나가 플로우를 설계할 때 채우는 표준 양식. 최종 설계서의 "4. Power Automate 플로우 명세" 절에 그대로 들어간다.

---

## 1. 플로우 개요

| 항목 | 값 | 비고 |
|------|----|------|
| 플로우명 | {{ 플로우명 }} | 한글 가능 |
| 위치 | 외부망 / 내부망 | 망분리 결정에 따름 |
| 환경(Environment) | {{ 환경명 }} | 회사 환경명 표기 |
| 트리거 종류 | 자동 / 인스턴트 / 일정 | |
| 실행 빈도 | {{ 예: 매일 09:00 KST }} | 일정 기반인 경우 |
| 예상 일 호출 횟수 | {{ 예: 1회 }} | DLP/처리량 영향 |
| 라이선스 등급 | Standard / Premium | 사용 커넥터 중 최고 등급 |

---

## 2. 사용 커넥터 목록

| 커넥터 | 등급 | 테넌트 | 용도 |
|--------|------|--------|------|
| {{ 예: HTTP }} | Premium | 외부망 | 외부 API 호출 |
| {{ 예: SharePoint }} | Standard | 내부망 | 데이터 적재 |

---

## 3. 단계 명세

| 순번 | 단계명 | 액션 종류 | 커넥터 | 입력 매개변수 | 출력 변수 | 비고 |
|-----|--------|----------|--------|--------------|----------|------|
| 1 | 트리거 | Recurrence | Schedule | Frequency=Day, At=09:00, TimeZone=Korea Standard Time | - | - |
| 2 | 외부 API 호출 | HTTP | HTTP | Method=GET, URI=https://..., Headers=... | response | 인증 토큰 필요 |
| 3 | JSON 파싱 | Parse JSON | Data Operations | Content=@body('HTTP'), Schema=... | parsed | - |
| 4 | 조건 분기 | Condition | Built-in | length(parsed.items) > 0 | - | true 분기만 사용 |
| 5 | 반복 | Apply to each | Built-in | @parsed.items | currentItem | 병렬 제한 50 |
| 6 | 데이터 적재 | Create item | SharePoint | List URL, Title=@currentItem.title | - | - |

---

## 4. 변수 목록

| 변수명 | 타입 | 초기값 | 용도 |
|-------|------|--------|------|
| {{ varName }} | String / Integer / Array / Object / Boolean | {{ initial }} | {{ 용도 }} |

---

## 5. 조건 분기 상세

분기가 단순하면 생략 가능. 복잡하면 다음 형식:

```
If length(items) > 0:
  → For each item:
      → If item.status == "Active":
          → Create SharePoint item
      → Else:
          → Skip
Else:
  → Send no-data notification
```

---

## 6. 에러 핸들링

### 6.1 실패 시 동작
| 단계 | 실패 시 동작 |
|------|------------|
| HTTP 호출 실패 | 3회 재시도 (지수 백오프), 그래도 실패 시 관리자에게 메일 |
| SharePoint 적재 실패 | 1회 재시도, 실패 시 로그 |

### 6.2 재시도 정책
- Power Automate 기본 재시도 설정: ...
- Run After 구성: ...

---

## 7. 처리량·한도 검토

- 외부 API 호출 한도: 사용 API의 일별 한도
- Power Automate 처리량 한도: 라이선스별 한도
- 예상 호출 패턴이 한도 내인지 확인

---

## 8. DLP 정책 영향

- 사용 커넥터가 모두 같은 DLP 그룹에 속하는가? (Business + Non-Business 혼합 금지)
- 회사 DLP 정책의 Custom Connector·HTTP·Premium 정책 검토 결과

---

## 9. 알려진 제약

사용 커넥터·액션의 특이사항. 예:
- SharePoint List `Get items`는 5,000건 한도. Filter Query 활용 또는 페이지네이션 필요
- HTTP 액션은 120초 타임아웃
