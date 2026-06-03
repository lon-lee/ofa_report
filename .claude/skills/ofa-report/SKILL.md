---
name: ofa-report
description: Use when asked to generate 해외금융계좌신고 (overseas financial account report) XLSX file. Triggers on phrases like "해외금융계좌신고 만들어줘", "OFA 신고 파일", ticker + year + 신고, "해외계좌 보고서". Requires year, Korean ticker name, investing.com URL slug, and holdings as inputs.
---

# 해외금융계좌 신고 보고서 생성

## Overview
연도·종목·보유주수를 입력받아 investing.com에서 월말 종가·환율 데이터를 수집하고, 로컬 템플릿 XLSX를 업데이트하여 신고용 파일을 생성한다.

## Required Inputs
| 항목 | 예시 | 설명 |
|------|------|------|
| year | 2025 | 신고 대상 연도 |
| ticker_korean | 애플 | 파일명에 사용할 한글 종목명 |
| url_slug | apple-computer-inc | investing.com URL의 종목 슬러그 |
| holdings | 1000 | 보유 주수 |

**파일명 형식**: `[{ticker_korean}]+해외금융계좌신고+요청자료({year}년)_최선영.xlsx`

## Template Cell Mapping
```
B15:B26  기준일 (YYYY-MM-DD, 각 월의 마지막 거래일)
C15:C26  보유주식 수 (입력받은 holdings 값)
D15:D26  신고 대상 여부 → 수식 자동계산 (수정 불필요)
E15:E26  USD 평가액    → 수식 자동계산 (수정 불필요)
F15:F26  KRW 평가액    → 수식 자동계산 (수정 불필요)
G15:G26  종가 (USD, investing.com 기준 월말 종가)
H15:H26  공란
I15:I26  환율 (USD/KRW, investing.com 기준 월말 환율)
```

## Step 1: investing.com 데이터 수집

Playwright로 JavaScript 렌더링 페이지에서 연간 일별 데이터를 수집한다.

**환율 URL**: `https://kr.investing.com/currencies/usd-krw-historical-data`  
**종가 URL**: `https://kr.investing.com/equities/{url_slug}-historical-data`

### Playwright 날짜 범위 설정 절차
1. `browser_navigate`로 URL 접속
2. `browser_snapshot`으로 날짜 필터 UI 요소 확인 (보통 우측 상단에 날짜 범위 표시)
3. 날짜 범위 클릭 → 달력 UI 열림
4. 시작일 `{year}-01-01`, 종료일 `{year}-12-31` 입력
5. 적용 버튼 클릭 후 테이블 로딩 대기 (`browser_wait_for`)
6. `browser_snapshot`으로 데이터 테이블 확인 후 `browser_evaluate`로 추출

### 테이블 데이터 추출 (JavaScript)
```javascript
// browser_evaluate로 실행
const rows = document.querySelectorAll('table tbody tr');
const data = [];
rows.forEach(row => {
  const cells = row.querySelectorAll('td');
  if (cells.length >= 2) {
    data.push({
      date: cells[0].textContent.trim(),
      close: cells[1].textContent.trim()
    });
  }
});
return JSON.stringify(data);
```

### 월말 마지막 거래일 추출 (Python)
```python
import calendar

def get_monthly_last_trading(data: dict, year: int) -> dict:
    """
    data = {'2025-01-31': 1450.5, ...}  날짜→종가/환율 dict
    returns {1: ('2025-01-31', 1450.5), 2: ('2025-02-28', 1430.0), ...}
    """
    result = {}
    for month in range(1, 13):
        last_day = calendar.monthrange(year, month)[1]
        for day in range(last_day, 0, -1):
            key = f"{year}-{month:02d}-{day:02d}"
            if key in data:
                result[month] = (key, data[key])
                break
    return result

def parse_investing_value(val: str) -> float:
    """'1,470.00' → 1470.0, '21.98' → 21.98"""
    return float(val.replace(',', '').strip())
```

### 날짜 파싱 (investing.com 한국어 형식)
```python
from datetime import datetime

def parse_investing_date(date_str: str) -> str:
    """
    '2025년 01월 31일' → '2025-01-31'
    또는 영문 'Jan 31, 2025' → '2025-01-31'
    """
    try:
        dt = datetime.strptime(date_str.strip(), '%Y년 %m월 %d일')
    except ValueError:
        dt = datetime.strptime(date_str.strip(), '%b %d, %Y')
    return dt.strftime('%Y-%m-%d')
```

## Step 2: 로컬 템플릿 로드

저장소 내 `template/ofa_template.xlsx`를 openpyxl로 직접 로드한다:
```python
import openpyxl

template_path = 'template/ofa_template.xlsx'
wb = openpyxl.load_workbook(template_path)
```

## Step 3: XLSX 셀 업데이트 후 저장

```python
ws = wb.active  # 첫 번째 시트 사용

for i, month in enumerate(range(1, 13)):
    row = 15 + i  # 15~26행
    date_str, close_price = stock_monthly[month]
    _, exch_rate = exch_monthly[month]

    ws[f'B{row}'] = date_str        # 기준일
    ws[f'C{row}'] = holdings        # 보유 주수
    ws[f'G{row}'] = close_price     # 종가
    ws[f'I{row}'] = exch_rate       # 환율

output_filename = f'[{ticker_korean}]+해외금융계좌신고+요청자료({year}년)_최선영.xlsx'
wb.save(output_filename)
print(f"저장 완료: {output_filename}")
```

**주의**: openpyxl은 기존 수식을 보존한다. D/E/F/H열 수식은 Excel에서 파일을 열면 자동 재계산되므로 수정 불필요.

## Common Issues

| 문제 | 해결 방법 |
|------|----------|
| 테이블 로딩 안됨 | `browser_wait_for`로 테이블 `tbody tr` 요소 대기 |
| 월말이 주말/공휴일 | `get_monthly_last_trading()`이 역순 탐색으로 자동 처리 |
| 환율에 쉼표 포함 | `parse_investing_value()`로 정규화 |
| 날짜 형식 불일치 | `parse_investing_date()`의 두 가지 형식 처리 |
| 데이터 로딩 페이지 수 부족 | 스크롤 또는 페이지네이션 확인 필요 |
| openpyxl 수식 덮어쓰기 | D/E/F/H열은 절대 직접 쓰지 말 것 |
