# 해외금융계좌신고 자동화 (OFA Report)

Claude Code + Playwright + Google Drive MCP를 이용해 매년 해외금융계좌신고(OFA) 제출용 XLSX 파일을 자동 생성합니다.

investing.com에서 월말 종가·환율 데이터를 수집하고, Google Drive 템플릿에 채워 넣은 뒤 Drive에 업로드합니다.

---

## 사전 요구사항

| 항목 | 내용 |
|------|------|
| [Claude Code](https://claude.ai/code) | CLI 설치 필요 |
| Playwright MCP | `@playwright/mcp` 서버 연결 |
| Google Drive MCP | `@google/drive-mcp` 서버 연결 |
| Python 3 + openpyxl | `pip install openpyxl` |

### MCP 서버 설정 (`~/.claude/settings.json`)

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp"]
    },
    "google-drive": {
      "command": "npx",
      "args": ["@google/drive-mcp"]
    }
  }
}
```

---

## 스킬 설치

이 저장소의 스킬을 Claude Code에서 사용하려면 `.claude/skills` 디렉토리를 복사하거나 심볼릭 링크를 걸면 됩니다.

```bash
# 저장소 클론
git clone https://github.com/lon-lee/ofa_report.git
cd ofa_report

# 스킬 설치 (전역)
cp -r .claude/skills/ofa-report ~/.claude/skills/
```

---

## 사용법

Claude Code에서 다음과 같이 입력합니다.

```
2025년 쿠팡 해외금융계좌신고 만들어줘. 보유주수는 14500주야.
```

또는 직접 스킬을 호출합니다.

```
/ofa-report
```

### 필수 입력값

| 항목 | 예시 | 설명 |
|------|------|------|
| year | 2025 | 신고 대상 연도 |
| ticker_korean | 쿠팡 | 파일명에 사용할 한글 종목명 |
| url_slug | coupang-llc | investing.com URL 슬러그 |
| holdings | 14500 | 보유 주수 |

`url_slug`는 investing.com 종목 페이지 URL에서 확인합니다.  
예) `https://kr.investing.com/equities/coupang-llc-historical-data` → `coupang-llc`

---

## 처리 흐름

```
1. investing.com에서 해당 연도 월말 종가·환율 수집 (Playwright)
2. Google Drive 템플릿 다운로드
3. openpyxl로 B/C/G/I 열 데이터 입력 (D/E/F 열은 수식 자동계산)
4. 완성 파일을 Google Drive에 업로드
```

---

## 템플릿 구조

`template/ofa_template.xlsx` 기준:

| 열 | 내용 | 비고 |
|----|------|------|
| B (15~26행) | 기준일 (YYYY-MM-DD) | 각 월의 마지막 거래일 |
| C (15~26행) | 보유주식 수 | 입력값 |
| D (15~26행) | 신고 대상 여부 | 수식 자동계산 |
| E (15~26행) | USD 평가액 | 수식 자동계산 |
| F (15~26행) | KRW 평가액 | 수식 자동계산 |
| G (15~26행) | 종가 (USD) | investing.com 기준 |
| I (15~26행) | 환율 (USD/KRW) | investing.com 기준 |

---

## 출력 파일명

```
[{ticker_korean}]+해외금융계좌신고+요청자료({year}년)_최선영.xlsx
```

예) `[쿠팡]+해외금융계좌신고+요청자료(2025년)_최선영.xlsx`

최종 파일은 Google Drive의 지정 폴더에 자동 업로드됩니다.
