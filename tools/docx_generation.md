# 마크다운 → docx 변환 가이드

Documentation 페르소나가 작성한 마크다운 설계서를 최종 `.docx`로 변환하는 방법.

---

## 1. 권장 방식: Pandoc

### 1.1 설치
- Windows: https://pandoc.org/installing.html
- 또는 Chocolatey: `choco install pandoc`

### 1.2 기본 변환
```powershell
pandoc docs\designs\20260523_법령변경알림.md -o docs\designs\20260523_법령변경알림.docx
```

### 1.3 참조 스타일 적용 (선택)
회사 표준 스타일을 따르는 reference.docx가 있다면:
```powershell
pandoc docs\designs\20260523_법령변경알림.md `
  --reference-doc=tools\reference.docx `
  -o docs\designs\20260523_법령변경알림.docx
```

### 1.4 Mermaid 다이어그램
Pandoc은 기본적으로 Mermaid를 렌더링하지 못한다. 다음 중 선택:

#### 옵션 A: 사전에 이미지로 변환
1. https://mermaid.live 에서 다이어그램을 PNG로 export
2. `docs/designs/images/`에 저장
3. 마크다운에서 ` ```mermaid ` 블록 대신 `![Architecture](images/architecture.png)` 사용

#### 옵션 B: mermaid-filter 사용 (Node 필요)
```
npm install -g mermaid-filter
pandoc input.md -F mermaid-filter -o output.docx
```

---

## 2. 대안: Python python-docx

Pandoc 사용이 어려운 환경에서 Python으로 직접 생성 가능.

```python
from docx import Document
from docx.shared import Pt

doc = Document()
doc.add_heading('법령변경알림 설계서', 0)
doc.add_paragraph('작성일: 2026-05-23')
# ...
doc.save('docs/designs/20260523_법령변경알림.docx')
```

표·다이어그램 처리가 복잡해 권장하지 않음. Pandoc 우선.

---

## 3. 대안: Word 직접 변환

1. 마크다운 파일을 텍스트 에디터에서 열기
2. 전체 복사 → Word에 붙여넣기
3. Word의 "Markdown 가져오기" 기능 또는 수동 서식 적용
4. 표·이미지 정리 후 `.docx`로 저장

Claude Code 자동화에는 부적합. 사용자 수동 작업.

---

## 4. 파일명·경로 규칙 (재확인)

- 경로: `docs/designs/`
- 마크다운 초안: `YYYYMMDD_<프로젝트명>.md`
- 최종 docx: `YYYYMMDD_<프로젝트명>.docx`
- 날짜는 한국 시간 기준
- 이미지는 `docs/designs/images/<프로젝트명>/` 하위에 저장

예시:
```
docs/designs/
├── 20260523_법령변경알림.md
├── 20260523_법령변경알림.docx
└── images/
    └── 20260523_법령변경알림/
        ├── architecture.png
        └── flow_overview.png
```

---

## 5. Claude Code 워크플로

1. Documentation 페르소나가 마크다운 작성 → 저장소에 커밋
2. (선택) Mermaid 다이어그램을 mermaid.live 또는 mermaid-filter로 이미지화
3. Pandoc 명령으로 docx 변환
4. 결과 docx 파일도 저장소에 커밋
5. 사용자에게 파일 경로 안내

Claude Code가 Bash/PowerShell에서 Pandoc을 직접 실행해 docx까지 생성하는 것을 권장. 다이어그램이 복잡하지 않은 경우 옵션 A(사전 이미지화)가 가장 안정적.

---

## 6. 트러블슈팅

| 증상 | 해결 |
|------|------|
| Pandoc이 없다고 함 | `pandoc -v` 확인, 설치 |
| 한글이 깨짐 | 마크다운 파일이 UTF-8로 저장되어 있는지 확인 |
| 표가 망가짐 | 마크다운 표 사이 빈 줄 확인, Pandoc 옵션 `--standalone` 시도 |
| Mermaid가 코드로 그대로 출력 | 옵션 A 또는 B 적용 |
