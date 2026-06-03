# 해외금융계좌신고 자동화 (OFA Report)

Claude Code + Playwright MCP를 이용해 매년 해외금융계좌신고(OFA) 제출용 XLSX 파일을 자동 생성합니다.

investing.com에서 월말 종가·환율 데이터를 수집하고, 로컬 템플릿(`template/ofa_template.xlsx`)에 채워 넣어 신고용 파일을 저장합니다. **여러 종목을 한 번에 처리할 수 있습니다.**

---

## 사전 요구사항

| 항목 | 내용 |
|------|------|
| [Claude Code](https://claude.ai/code) | CLI 설치 필요 |
| Playwright MCP | `@playwright/mcp` 서버 연결 |
| Python 3 + openpyxl | `pip install openpyxl` |

### MCP 서버 설정 (`~/.claude/settings.json`)

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp"]
    }
  }
}
```

---

## 스킬 설치

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
# 단일 종목
2025년 애플 1000주 해외금융계좌신고 만들어줘.

# 복수 종목 (환율은 한 번만 수집)
2025년 애플 1000주, 쿠팡 14500주 해외금융계좌신고 만들어줘.
```

또는 직접 스킬을 호출합니다.

```
/ofa-report
```

### 필수 입력값

| 항목 | 예시 | 설명 |
|------|------|------|
| year | 2025 | 신고 대상 연도 |
| ticker_korean | 애플 | 파일명에 사용할 한글 종목명 |
| url_slug | apple-computer-inc | investing.com URL 슬러그 |
| holdings | 1000 | 보유 주수 |

복수 종목 시 ticker_korean, url_slug, holdings를 종목 수만큼 제공합니다.

`url_slug`는 investing.com 종목 페이지 URL에서 확인합니다.  
예) `https://kr.investing.com/equities/apple-computer-inc-historical-data` → `apple-computer-inc`

---

## 처리 흐름

```
1. investing.com에서 환율 데이터 수집 (1회, 종목 수와 무관)
2. 종목별로 종가 데이터 수집 (Playwright)
3. 종목별로 템플릿 로드 후 B/C/G/I 열 입력 (D/E/F 열은 수식 자동계산)
4. 종목별 파일을 현재 디렉토리에 저장
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
[{ticker_korean}]+해외금융계좌신고+요청자료({year}년).xlsx
```

예) `[애플]+해외금융계좌신고+요청자료(2025년).xlsx`
