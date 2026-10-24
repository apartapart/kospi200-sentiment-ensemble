# 파이프라인 상세

전처리부터 최종 평가까지 실제로 코드가 실행된 순서. 각 단계마다 **입력 → 노트북/작업 → 출력** 을 기록함.
모든 노트북은 Google Colab + Google Drive 기준으로 작성됨. 경로 변수(`/content/drive/MyDrive/...`)를 로컬 경로로 바꾸면 그대로 실행 가능.

```
[A] 정형 데이터          [B] 비정형 데이터 (뉴스)
 KRX KOSPI200 csv         빅카인즈 csv (1군/2군, 기업 10개)
      │                         │
      ▼ 01                      ▼ 05
 kospi200_cleaned_data.csv   df_first_* / df_second_* (제목+키워드+본문 concat)
      │                         │
      │                         ▼ 03 → 04 (klue/bert-base 이진 분류 fine-tune, test acc 0.93)
      │                         ▼ 06  기사별 pos/neg 확률
      │                         ▼ 07  일자별 평균 (1군, 2군 각각)
      │                         ▼ 07b 휴장일 병합(평균) → 1군×3 + 2군×1 → ÷4
      │                         │   = sentiment/06_weighted_avg_div4_FINAL_INPUT/*_AVG.csv
      ▼ 08                      │
 기업별 거래량·변동률 ──────────┤
                                ▼ 09
              기업 10개 × 피처 6개 = 60차원, 21일 윈도우, T+1 종가 예측
              LSTM / GRU / CNN-LSTM seed ensemble → hybrid ensemble (soft voting)
                                │
                                ▼ [G] grid search (3584조건, VS Code에서 실행)
              individual_model_results.json / ensemble_model_results.json (상위 60)
                                │
                                ▼ 10, 11  결과 시각화·혼동행렬
                                ▼ 12, 13  백테스팅 (2025-01~05, 2018~2019)
```

---

## 0. 대상·기간

- 기간: **2022-01-27 ~ 2024-12-30** (LG에너지솔루션 상장일 기준)
- 기업 10개: Hyundai, Kbfg, Kia, Lgchem, Lgensol, Naver, PoscoHoldings, SamsungBiologics, Samsung, Skhynix
- 타깃: KOSPI200 **다음 거래일 종가** (`target = closing price.shift(-1)`)

---

## 1. `01_kospi200_preprocessing.ipynb`

| | |
|---|---|
| 입력 | `kospi 200 2022 2023 2024.csv` (KRX, 컬럼 `date, closing price, market price, high price, low price, volume_M, change`) |
| 처리 | `date` → `y,m,d` 분리. `volume_M` 의 `M`/`K` 문자 제거 후 float. `change` 의 `%` 제거 후 float |
| 출력 | `dataset/kospi200_cleaned_data.csv` → 리포 `data/processed/kospi200/` (원본 `kospi200_raw_2022_2024.csv` 와 동일 로직으로 재생성한 cleaned csv 포함, 718행) |

## 2. `02_kospi200_eda.ipynb`

종가·시가·고가·저가·거래량 시각화. 산출물 없음. 그림: `results/figures/02_kospi200_eda_*.png`

## 3. `03_sentiment_finetune_klue_bert.ipynb` — 감성 분류기 학습

| | |
|---|---|
| 입력 | `NewsResult_20220127-20241231_pos.csv`, `NewsResult_20220127-20241231_neg.csv` (빅카인즈에서 긍정 기사·부정 기사를 각각 수집한 파일. 파일 자체가 라벨. 빅카인즈 1회 export 상한 20,000건 → 각 20,000행, 전처리 후 19,673 / 19,783) |
| 전처리 | 제목 기준 중복 제거 → `<오늘의 인사>`, `기업공시` 제목 제외 → `content = 제목 + ' ' + 키워드 + ' ' + 본문` |
| 라벨 | pos = 1, neg = 0 |
| 분할 | train 80 / val 10 / test 10, stratified, `random_state=42` |
| 모델 | `klue/bert-base` + `AutoModelForSequenceClassification(num_labels=2)` |
| 하이퍼파라미터 | epochs 5, batch 16, lr 5e-5, weight_decay 0.01, max_length 512, HF `Trainer` |
| 출력 | `kobert_model/model_output/final_model_kobert_sentiment/` (config, tokenizer. 가중치 `model.safetensors` 는 리포 미포함) |
| 성능 | validation `eval_accuracy` 0.99 (노트북 출력). 논문·발표의 **0.93** 은 별도 샘플 200건 수동 검증 |
| 역할 | 이 라벨셋으로 분류기를 학습한 뒤, **라벨 없는 나머지 기사 전체**(기업별 1군/2군)에 6번에서 pos/neg 확률을 부여 |

> 폴더명·노트북명은 `kobert` 지만 실제 베이스 모델은 **`klue/bert-base`** 임 (`config.json`: BertTokenizer, vocab 32000). 논문 3.1.1의 "SKTBrain KoBERT" 서술은 코드와 불일치.
> `03a_*_abandoned.ipynb` 는 `monologg/kobert` 3-class 로 시도했다가 폐기한 버전.

## 4. `04_sentiment_inference_test.ipynb`

저장된 모델 로드 후 샘플 뉴스 제목 100건으로 추론 확인. 출력 `prediction_results.csv` (컬럼: `Original News, Predict label, Neg Probability (0), Pos Probability (1), Predictive sensitivity`). 뉴스 제목 포함이라 리포 미수록. 본 파이프라인에는 쓰이지 않는 검증용.

## 5. `05_news_group_split_and_topicality_score.ipynb` — 뉴스 전처리

| | |
|---|---|
| 입력 | `뉴스기사/1군/{기업}1군.csv`, `뉴스기사/2군/{기업}2군.csv` (빅카인즈 원본), `Score_sum/{company}_score_sum_holiday.csv` (화제성 점수) |
| 처리 | `columns_to_keep` 로 필요한 컬럼만 유지 → `content = 제목 + 키워드 + 본문` |
| 출력 | `dataset/df_first/df_first_{company}.csv` (1군), `dataset/df_second/df_second_{company}.csv` (2군), `dataset/df_score/df_score_{company}.csv` |

## 6. `06_sentiment_inference_first_second.ipynb` — 기사별 감성 추론

| | |
|---|---|
| 입력 | `df_first_{company}.csv`, `df_second_{company}.csv` |
| 처리 | `predict_sentiment()`: batch 16, max_length 512, softmax → `부정 확률 (0)`, `긍정 확률 (1)`, `예측 라벨`, `감성` |
| 출력 | `result/first_file/{company}_sentiment_result.csv`, `result/second_file/{company}_sentiment_result.csv` (→ `emotion_analysis/` 로 복사) |

## 7. `07_sentiment_daily_aggregate.ipynb` — 일자별 집계

| | |
|---|---|
| 입력 | 6번 출력 (`content` 컬럼 drop) |
| 처리 | `groupby('일자')` 후 `부정 확률 (0)`, `긍정 확률 (1)` **평균**. `감성 = 긍정 if 긍정확률 >= 부정확률` |
| 출력 | `emotion/first_team/{company}_aggregated_sentiment.csv`, `emotion/second_team/{company}_aggregated_sentiment.csv` → 리포 `data/processed/sentiment/01_daily_mean_group{1,2}/` |

> 1군·2군 각각 일자별 평균을 낸 뒤 7b 에서 가중 결합.

## 7b. `07b_holiday_merge_and_group_weighting.ipynb` — 휴장일 처리 + 1군/2군 가중 집계

Colab `files.upload()` 방식의 단일 셀 스크립트 7개를 순서대로 실행. 각 셀은 csv 업로드 → 처리 → 다운로드. 중간 파일은 `data/processed/sentiment/02_*` ~ `06_*` 에 보존.

| 셀 | 작업 | 로직 | 출력 접미사 | 리포 폴더 |
|---|---|---|---|---|
| [1] | 화제성 csv 휴장일 플래그 | `holiday_set`(2022~2024 KRX 휴장일 하드코딩) ∪ 주말 → D열 `1` | `_holiday` | `topicality_score/02_holiday/` |
| [3] | 화제성 휴장일 병합 + MinMax | 영업일(D=0) 행이 그룹 시작, 이어지는 휴장일(D=1) 행을 흡수해 **평균**. `E = (B-min)/(max-min)` | `_sum` | `topicality_score/03_minmax/` |
| [5] | 감성 csv 휴장일 플래그 | `Weekday`, `Market_Closed` 컬럼 추가 | `_holiday` | `sentiment/02_group{1,2}_holiday/` |
| [7] | 감성 휴장일 병합 | 영업일 행 + 이어지는 휴장일 행의 `부정`, `긍정` 확률 **평균**. 요일 문자열 concat | `_화제성_sum` | `sentiment/03_group{1,2}_holiday_merged/` |
| [9] | 1군 × 3 | `B열 *= 3`, `C열 *= 3` | `_triple` | `sentiment/04_group1_weighted_x3/` |
| [11] | 1군 + 2군 | `merge(on='A', how='outer')`, `B = B_1 + B_2`, `C = C_1 + C_2`, 결측 0 | `_merged_result` | `sentiment/05_weighted_sum_group1x3_plus_group2/` |
| [13] | ÷ 4 | `B열 /= 4`, `C열 /= 4` → 0~1 확률 복원 | `_AVG` | `sentiment/06_weighted_avg_div4_FINAL_INPUT/` |

컬럼명은 `A,B,C,E,F` = `일자, 부정, 긍정, 요일, 휴장일여부`. 노트북 09의 `new_col_names` 에서 매핑.
셀 [1]~[3]의 화제성 점수는 최종 모델 입력에는 쓰이지 않음(실험용).

## 7c. `07c_normalization_methods_experiment.ipynb`

첫 번째 숫자 컬럼에 MinMax / Max / Z-score / Log / Rank / Sigmoid / Softmax / Robust 8종 정규화를 한 번에 적용해 비교. 논문 Table 6 의 `MinmaxScaler / SoftmaxScaler` 선택 근거 실험. `_normalized.csv` 출력.

## 8. `08_trading_volume_volatility.ipynb`

| | |
|---|---|
| 입력 | `new/{company}.csv` (Investing.com 형식 기업별 주가) |
| 처리 | `종가,시가,고가,저가` drop → `날짜, 거래량, 변동 %` 만 유지 |
| 출력 | `기업별_거래량_변동성/df_{company}.csv` → 리포 `data/processed/trading_volume_volatility/` |

## 9. `09_lstm_gru_cnn_hybrid_ensemble.ipynb` — 시계열 학습 (핵심)

### 9-1. 피처 병합

| 소스 | 컬럼 | 변환 |
|---|---|---|
| `kospi200_cleaned_data.csv` | `date, closing price, change` | `target = closing price.shift(-1)` |
| `df_{company}.csv` | `trading`, `change` | `1.41M → 1410000`, `-0.34% → -0.34` |
| `*_AVG.csv` | `neg`, `pos` | `emotion = 1 if pos > neg else -1` |
| 파생 | `calculate` | `pos - neg`. 발표자료 명칭 **순감성민감도(NNC)**, 논문은 Net Sentiment Score [9] |

기업별 `[neg, pos, emotion, trading, change, calculate]` 6개 → 10개 기업 inner join on `date` → **60차원** + `target`.

### 9-2. 전처리

- `MinMaxScaler` (X, y 별도). 논문은 train 기준 fit 명시. (노트북 셀에서는 전체 fit 후 분할 — 재현 시 주의)
- `Sequences(X, y, time_steps=21)` → 21일 윈도우
- 시계열 순서 보존 분할: train 0.71 / val 0.151 / test 0.139

### 9-3. 모델

```
LSTM(100, return_sequences) → Dropout(0.25)
LSTM(50,  return_sequences) → Dropout(0.19)
LSTM(25)                    → Dropout(0.10)
Dense(1)
optimizer=adam, loss=mse, epochs=50, batch=16, EarlyStopping(val_loss, patience=7)
```

GRU: 동일 구조에서 `LSTM → GRU`. CNN-LSTM: `Conv1D → MaxPooling1D → LSTM`.
논문 Table 6 기준 탐색 범위: units `[100,50,25]` / `[72,36,18]`, dropout `[0.3,0.15,0.1]` / `[0.25,0.19,0.1]`, time_step 20/21.

### 9-4. 앙상블

1. **Seed ensemble**: 같은 구조를 `seed = 0..num_models-1` 로 `num_models` 회 학습, 예측 평균. (노트북 `num_models=9`, 마크다운 메모는 10, 논문은 5 — 최종 결과 json 기준은 grid search 설정 따름)
2. **Hybrid ensemble**: `(avg_lstm + avg_gru + avg_cnn_lstm) / 3` soft voting. 가중 평균은 성능 저하로 폐기.
3. 역정규화 `scaler_y.inverse_transform` 후 RMSE / MAE / R² / MAPE 계산. Adam 기본값(lr 0.001, β1 0.9, β2 0.999, ε 1e-8). SGD/RMSProp/Adagrad 는 수렴 불안정으로 탈락.
4. 방향성: `y_true > 전일 종가` vs `y_pred > 전일 종가` → F1, confusion matrix.

### 9-5. 단계별 결과 (논문 Table 1~5)

| 실험 | 입력 | RMSE | MAE | R² |
|---|---|---|---|---|
| 결과 1 | 종가만 | 15.4074 | 11.3164 | -0.0125 |
| 결과 2 | + pos, neg | 12.1246 | 10.0023 | 0.3730 |
| 결과 3 | + calculate (pos−neg) | 11.0593 | 9.1936 | 0.4783 |
| 결과 4 | + 하이퍼파라미터, 시총 비중 | 10.0694 | 8.0649 | 0.4999 |
| Ensemble 최종 | + 거래량, 변동성, hybrid ensemble | **7.4357** | **6.1045** | **0.7286** |

## [G] Grid search — `results/*.json` 생성

노트북 09의 로직을 조건 루프로 돌린 스크립트. VS Code 로컬(TF 2.10.0, CUDA 11.2)에서 실행. **스크립트 원본은 리포 미포함.**

탐색 조건 (논문 Table 6):

| 축 | 값 |
|---|---|
| model | LSTM, CNN, GRU (+ hybrid 조합 7종) |
| units / dropout | `[100,50,25]/[0.3,0.15,0.1]`, `[72,36,18]/[0.25,0.19,0.1]` |
| time_step | 20, 21 |
| train/val ratio | 0.7/0.15, 0.71/0.151 |
| data_case | A `neg,pos,calculate` / B `+trading` / C `+change` / D `+trading,change` |
| cap_method | cap_on (시총 비중 포함) / cap_off |
| scaler | minmax / standard |

총 3584 조건 중 상위 60개씩 저장:

- `results/individual_model_results.json` — 단일 모델. 필드: `model, model_params, time_step, train_ratio, val_ratio, data_case, cap_method, scaler, RMSE, MAE, R2, MAPE, F1_Score, y_true, y_pred, dates`
- `results/ensemble_model_results.json` — 앙상블. 필드: `ensemble_name, models_in_ensemble, ...` 동일
- `results/all_model_results.csv`, `results/all_ensemble_data.csv` — 지표만 요약

## 10. `10_results_top_models_visualization.ipynb`

json 2개 로드 → 모델별 성능순 정렬 → 상위 모델 예측 vs KOSPI200 실제 그래프, 혼동행렬. 그림: `results/figures/10_*.png`

## 11. `11_results_ensemble_visualization.ipynb`

`ensemble_model_results.json` → csv 변환 → 상위 10개 앙상블 예측 vs 실제 (`results/figures/11_results_ensemble_visualization_02.png`)

## 12, 13. 백테스팅 (논문 외 추가 실험)

- `12a_backtesting_topicality_sentiment_preprocessing.ipynb`: 백테스팅 구간(2018-01~2019-02, 2025-01~05)용 뉴스 전처리. 빅카인즈 xlsx → csv 변환, `exchange_calendars`(XKRX) 로 실제 거래일 필터, 일자별 기사 수 → 1군×3 → 1군+2군 SUM → 휴장일 → 요일 병합·MinMax (`backtesting_*/article_count/`). 감성 쪽은 기사별 확률에 1군×3 / 2군×1 → 일자별 평균 → 휴장일 처리 (`backtesting_2018_2019/sentiment/`). 07b 와 로직 동일하나 `files.upload()` 대신 Drive 폴더 단위 일괄 처리.
- `12_backtesting_preprocessing.ipynb`: 2025-01~05 데이터로 1~9 과정 반복 → `backtesting_2025/merged_features/df_{company}.csv` → 기업별 LSTM 학습 → `backtesting_2025/predictions/result_{company}.csv`
- `13_backtesting.ipynb`: 예측 결과 시각화. 전략 규칙(발표자료): 감성 1 매수 / 0 매도, 초기 자본 1천만 원, `y_pred` 기준 자본 추적, Buy&Hold 비교. 결과: 삼성전자 +4.31%, 삼성바이오로직스 +2.60%, 네이버 +1.83%, KB금융 -13.56%. 2025 구간 KOSPI200 원본은 `backtesting_2025/kospi200/kospi200_raw_2021_2025-07.csv` (Investing.com 한글 형식)
- `backtesting_2018_2019/`: 2018~2019 구간 동일 파이프라인 산출물

---

## 최종 결과 (논문 Table 17)

| Ensemble | data_case | cap | scaler | ts | RMSE | MAE | R² | MAPE | F1 |
|---|---|---|---|---|---|---|---|---|---|
| **LSTM+GRU** | C | cap_on | minmax | 20 | **7.564** | **6.065** | **0.712** | 0.0178 | **0.608** |
| LSTM+GRU+CNN | C | cap_on | minmax | 20 | 7.573 | 6.262 | 0.711 | 0.0183 | 0.541 |
