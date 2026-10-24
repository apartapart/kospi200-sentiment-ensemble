# data/

원본 데이터(`data/raw/`)는 저작권 문제로 리포에 포함하지 않음. `data/processed/` 는 원문이 제거된 집계 결과만 포함.

## 원본 출처

| 데이터 | 출처 | 기간 | 비고 |
|---|---|---|---|
| 뉴스 기사 | [빅카인즈](https://www.bigkinds.or.kr) | 2022-01-27 ~ 2024-12-30 | KOSPI200 상위 10개 기업 검색. 1군(방송사 5개: KBS, MBC, SBS, JTBC, tvN) / 2군(경제지 12개) 분리 수집 |
| KOSPI200 지수 | KRX / Investing.com | 2022-01-27 ~ 2024-12-30 | `kospi 200 2022 2023 2024.csv` → `kospi200_cleaned_data.csv` |
| 기업별 주가·거래량 | Investing.com 형식 (`날짜,종가,시가,고가,저가,거래량,변동 %`) | 동일 | `trading_volume_volatility/df_{company}.csv` |
| 시가총액 | KRX | 동일 | 3개월 평균 시총, 코스피 비중 |

## 대상 기업 (10개)

| 코드명 | 기업 |
|---|---|
| Hyundai | 현대자동차 |
| Kbfg | KB금융 |
| Kia | 기아 |
| Lgchem | LG화학 |
| Lgensol | LG에너지솔루션 |
| Naver | 네이버 |
| PoscoHoldings | POSCO홀딩스 |
| SamsungBiologics | 삼성바이오로직스 |
| Samsung | 삼성전자 |
| Skhynix | SK하이닉스 |

## 언론사 그룹

- **1군 (가중치 3)**: KBS, MBC, SBS, JTBC, tvN
- **2군 (가중치 1)**: 대한경제, 매일경제, 머니투데이, 메트로경제, 브릿지경제, 서울경제, 아시아경제, 아주경제, 이데일리, 이투데이, 파이낸셜뉴스, 한국경제, 헤럴드경제

## processed/ 폴더 설명

| 폴더 | 내용 | 생성 단계 |
|---|---|---|
| `kospi200/` | `kospi200_raw_2022_2024.csv` (KRX 원본, 719행) + `kospi200_cleaned_data.csv` (노트북 01 로직 재생성). 컬럼: `y,m,d,closing price,market price,high price,low price,volume_M,change` | 노트북 01 |
| `sentiment/01_daily_mean_group{1,2}/` | 기사별 감성 확률을 일자별 평균 | 노트북 07 |
| `sentiment/02_*_holiday/` | 휴장일 컬럼 추가 (`Weekday`, `Market_Closed`) | 노트북 07b 셀[5] |
| `sentiment/03_*_holiday_merged/` | 영업일 + 이어지는 휴장일 확률 평균 | 노트북 07b 셀[7] |
| `sentiment/04_group1_weighted_x3/` | 1군 확률 × 3 | 노트북 07b 셀[9] |
| `sentiment/05_weighted_sum_group1x3_plus_group2/` | 1군×3 + 2군×1 | 노트북 07b 셀[11] |
| `sentiment/06_weighted_avg_div4_FINAL_INPUT/` | 위 합계 ÷ 4. **노트북 09의 최종 입력** | 노트북 07b 셀[13] |
| `calculate_index/` | `계산식` 피처(가중 불일치 지수) 실험 버전 first/second | 실험용 |
| `trading_volume_volatility/` | 기업별 `날짜,거래량,변동 %` | 노트북 08 |
| `market_cap/` | 기업별 3개월 평균 시총, 코스피 비중 (`cap_on` 실험용) | KRX 다운로드 |
| `topicality_score/` | 일자별 기사 수 기반 화제성 점수 → 휴장일 처리 → MinMax | 노트북 05 + 07b 셀[1],[3] |
| `backtesting_2025/` | 2025-01 ~ 2025-05 백테스팅용 동일 파이프라인 산출물 | 노트북 12a, 12, 13 |
| `backtesting_2018_2019/` | 2018 ~ 2019 백테스팅용 동일 파이프라인 산출물 | 노트북 12a, 12, 13 |

노트북 07b 산출 파일의 컬럼명 `A,B,C,E,F,(D)` 는 각각 `일자, 부정확률, 긍정확률, 요일, 휴장일여부(, 기타)` 임. 노트북 09에서 `new_col_names` 로 다시 매핑함.
