---
name: ofa-report
description: Use when asked to generate 해외금융계좌신고 (overseas financial account report) XLSX file. Triggers on phrases like "해외금융계좌신고 만들어줘", "OFA 신고 파일", ticker + year + 신고, "해외계좌 보고서". Supports multiple tickers in one request. Requires year, and per-ticker: Korean ticker name, investing.com URL slug, and holdings.
---

# 해외금융계좌 신고 보고서 생성

## Overview
연도와 1개 이상의 종목·보유주수를 입력받아 investing.com에서 데이터를 수집하고, 종목별로 신고용 XLSX 파일을 생성한다.  
환율 데이터는 종목과 무관하므로 **한 번만 수집**하고, 종목별 종가 수집 후 각각 파일을 저장한다.

## Required Inputs
| 항목 | 예시 | 설명 |
|------|------|------|
| year | 2025 | 신고 대상 연도 |
| tickers | 아래 참고 | 종목 목록 (1개 이상) |

각 종목별 필요 정보:
| 항목 | 예시 | 설명 |
|------|------|------|
| ticker_korean | 애플 | 파일명에 사용할 한글 종목명 |
| url_slug | apple-computer-inc | investing.com URL의 종목 슬러그 |
| holdings | 1000 | 보유 주수 |

**입력 예시**:
- 단일: `2025년 애플 1000주`
- 복수: `2025년 애플 1000주, 쿠팡 14500주`

**파일명 형식**: `[{ticker_korean}]+해외금융계좌신고+요청자료({year}년).xlsx`

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

## Step 1: 환율 데이터 수집 (1회)

**환율 URL**: `https://kr.investing.com/currencies/usd-krw-historical-data`

Playwright로 날짜 범위를 `{year}-01-01` ~ `{year}-12-31`로 설정 후 데이터 추출.

### Playwright 날짜 범위 설정 절차
1. `browser_navigate`로 URL 접속
2. `browser_snapshot`으로 날짜 필터 UI 요소 확인
3. 날짜 범위 클릭 → 입력창 열림
4. 시작일 `{year}-01-01`, 종료일 `{year}-12-31` 입력
5. 적용 버튼 클릭 후 `browser_wait_for`로 해당 연도 데이터 로딩 대기
6. `browser_evaluate`로 테이블 데이터 추출

### 테이블 데이터 추출 (JavaScript)
```javascript
const rows = document.querySelectorAll('table tbody tr');
const data = [];
rows.forEach(row => {
  const cells = row.querySelectorAll('td');
  if (cells.length >= 2) {
    data.push({ date: cells[0].textContent.trim(), close: cells[1].textContent.trim() });
  }
});
return JSON.stringify(data);
```

## Step 2: 종목별 종가 수집 (종목 수만큼 반복)

**종가 URL**: `https://kr.investing.com/equities/{url_slug}-historical-data`

각 종목마다 동일한 날짜 범위 설정 절차를 반복한다.

## Step 3: 데이터 파싱 유틸

```python
import calendar
from datetime import datetime

def parse_investing_date(date_str: str) -> str:
    try:
        return datetime.strptime(date_str.strip(), '%Y년 %m월 %d일').strftime('%Y-%m-%d')
    except ValueError:
        return datetime.strptime(date_str.strip(), '%b %d, %Y').strftime('%Y-%m-%d')

def parse_investing_value(val: str) -> float:
    return float(val.replace(',', '').strip())

def parse_raw_json(raw_json_str: str) -> dict:
    """browser_evaluate 결과(이중 인코딩 문자열)를 날짜→값 dict로 변환"""
    import json
    rows = json.loads(json.loads(raw_json_str))
    result = {}
    for d in rows:
        date = parse_investing_date(d.get('date', '')) if '년' in d.get('date', '') else None
        if not date:
            continue
        try:
            result[date] = parse_investing_value(d['close'])
        except:
            pass
    return result

def get_monthly_last(data: dict, year: int) -> dict:
    """각 월의 마지막 거래일 추출. returns {1: ('2025-01-31', 236.0), ...}"""
    result = {}
    for month in range(1, 13):
        last_day = calendar.monthrange(year, month)[1]
        for day in range(last_day, 0, -1):
            key = f"{year}-{month:02d}-{day:02d}"
            if key in data:
                result[month] = (key, data[key])
                break
    return result
```

## Step 4: 종목별 XLSX 생성

환율은 공통이므로 `exch_monthly`는 Step 1에서 한 번만 계산. 종목마다 템플릿을 새로 로드해서 저장.

```python
import openpyxl

def generate_xlsx(year, ticker_korean, holdings, stock_monthly, exch_monthly):
    wb = openpyxl.load_workbook('template/ofa_template.xlsx')
    ws = wb.active
    for i, month in enumerate(range(1, 13)):
        row = 15 + i
        date_str, close_price = stock_monthly[month]
        _, exch_rate = exch_monthly[month]
        ws[f'B{row}'] = date_str
        ws[f'C{row}'] = holdings
        ws[f'G{row}'] = close_price
        ws[f'I{row}'] = exch_rate
    filename = f'[{ticker_korean}]+해외금융계좌신고+요청자료({year}년).xlsx'
    wb.save(filename)
    print(f"저장 완료: {filename}")

# 복수 종목 처리 예시
for ticker in tickers:
    generate_xlsx(year, ticker['korean'], ticker['holdings'], ticker['stock_monthly'], exch_monthly)
```

**주의**: openpyxl은 기존 수식을 보존한다. D/E/F/H열 수식은 Excel에서 파일을 열면 자동 재계산되므로 수정 불필요.

## Common Issues

| 문제 | 해결 방법 |
|------|----------|
| 테이블 로딩 안됨 | `browser_wait_for`로 해당 연도 첫 달 텍스트 대기 |
| 월말이 주말/공휴일 | `get_monthly_last()`이 역순 탐색으로 자동 처리 |
| 환율에 쉼표 포함 | `parse_investing_value()`로 정규화 |
| 날짜 형식 불일치 | `parse_investing_date()`의 두 가지 형식 처리 |
| url_slug 모를 때 | investing.com에서 종목 검색 후 URL에서 확인 |
| openpyxl 수식 덮어쓰기 | D/E/F/H열은 절대 직접 쓰지 말 것 |
