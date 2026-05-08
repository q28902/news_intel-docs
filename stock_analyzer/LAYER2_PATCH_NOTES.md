# L2 패치 노트 — 2026-04-18

테스트 브리핑 중간 우선순위 3건 반영.

## 변경 파일

| 파일 | 변경 |
|------|------|
| `layer2_engine.py` | `classify_variable()` 재작성 — 3개 플래그 실구현 / `VariableState`에 신규 필드 추가 / 포맷터에 시각화 아이콘 추가 |
| `data_sources.py` | `MarketReading`에 `bp_change_5d`, `prev_direction` 필드 추가 / mock 데이터에 금리 관련 시나리오 보강 |
| `layer2_output.json` | v2 출력 샘플 (신규 플래그 포함) |

## 이슈별 상세

### 1. `inverted` 플래그 실구현

**이전**: thresholds.json에 `"inverted": true`가 있었지만 엔진이 무시.
**이후**: 
- `VariableState.inverted: bool` 필드 추가
- `classify_variable()`이 inverted 지표에 대해 `notes`에 해석 힌트 생성 ("interpret: value up = improving" 등)
- 포맷터에 `↕` 시각 표시

적용 변수: `EURUSD`, `CN_PMI`, `EU_PMI`, `CN_PROPERTY`

### 2. `momentum_alert_bps` 실구현

**이전**: US_10Y 등 금리 변수의 bp 기준 모멘텀이 무시됨.
**이후**:
- `MarketReading.bp_change_5d: float | None` 필드 추가
- `classify_variable()`에서 `momentum_alert_bps`가 있으면 **pct보다 우선 적용**
- bp 임계값 초과 시 level 한 단계 승격 + `momentum_flag` 부착

### 3. `alert_on_direction_flip` 실구현

**이전**: 금리 방향 전환 감지 stub 상태.
**이후**:
- `MarketReading.prev_direction: str | None` 필드 추가
- `VariableState.direction_flip: bool` 필드 추가
- 강한 반전(`up ↔ down`): level 한 단계 승격 + flag
- 약한 반전(`flat ↔ up/down`): flag만 부착
- 포맷터에 `🔄` 시각 표시

## 검증

극한 시나리오(US_10Y 30bp 급등 + FED `down→up` 반전)로 테스트:
- US_10Y: `caution → alert` 승격, `⚡` flag
- FED_FUNDS: `normal → caution` 승격, `🔄` flag
- 모든 플래그 JSON 출력에 정상 직렬화

## Breaking Changes

없음. `MarketReading`과 `VariableState`에 추가된 필드는 모두 `Optional` 또는 기본값 `False`.

## Next

다음 단계는 실시간 데이터 소스 연결 (`data_sources.py`의 mock 함수들을 yfinance/FRED/네이버금융 실 데이터로 교체).

---

# L2 패치 노트 — 2026-05-01 (실데이터 어댑터 도입)

PATCH_NOTES "Next" 처리 — mock_realtime_data() 실데이터 버전 신설.

## 변경 파일

- `data_sources.py`: 끝부분에 어댑터 영역 신설 (~250줄). `mock_realtime_data()` 그대로 보존 (breaking change 0).

## 신규 함수

- `fetch_yfinance(name, ticker, ttl=900)` — Yahoo Finance, 15분 캐시
- `fetch_fred(name, series_id, ttl=86400)` — FRED API, 1일 캐시. CPI는 YoY 계산
- `fetch_komis(name, ttl=86400)` — KOMIS 비철·희소금속, 1일 캐시
- `fetch_realtime_data(verbose=False)` — 통합 (yfinance + FRED + KOMIS + mock fallback)

## 데이터 소스 매핑 (29개 변수)

### yfinance 13개 (무료, key 불필요)
- FX(5): USDKRW, USDCNY, USDJPY, EURUSD, DXY
- Energy(4): WTI(CL=F), Brent(BZ=F), HenryHub(NG=F), TTF(TTF=F)
- Materials(3): Copper(HG=F), Gold(GC=F), Wheat(ZW=F)
- Sentiment(1): VIX(^VIX)

### FRED 5개 (key는 .env `FRED_API_KEY`)
- Rates(3): FED_FUNDS(DFF), US_10Y(DGS10), ECB_RATE(ECBDFR)
- Economic(1): US_CPI_YoY(CPIAUCSL, YoY 계산)
- Rates(1): BOJ_RATE(IR3TIB01JPM156N, monthly. 5d 비교는 5개월 변동으로 해석되는 점 주의)

### KOMIS 2개 (한국광해광업공단, 무료, key 불필요)
- Lithium: HP002/MNRL0001/773 (Carbonate 99.5%, USD/kg)
- Nickel:  HP001/MNRL0002/502 (LME CASH, USD/ton)

### Mock 유지 9개 (TE/ECOS 키 도착 후 추가 예정)
- CN/EU economic(4): CN_PMI, CN_PROPERTY, CN_STIMULUS, EU_PMI → Trading Economics
- 한국 변수(1): KOR_US_SPREAD → ECOS API (L4 KR 모듈 분리 결정)
- 별도 source 부재(2): SCFI(Shanghai 공식 주1회), GLOBAL_FLOW(파생 지표)
- Policy 텍스트(2): US_TARIFF, OPEC_POLICY → news_intel 도킹 + LLM (P3+)

## 환경

- venv: `~/Projects/claude/LAYER1/venv` (LAYER1과 공유)
- 패키지 추가: `yfinance==1.3.0`, `fredapi==0.5.2`, `python-dotenv==1.2.2`
- `.env` 로드: `/Volumes/AIDRIVE/.env` (chmod 600), `FRED_API_KEY` 표준 변수명
- 캐시: `~/.cache/layer2_data/{yf,fred,komis}_<name>.pkl`

## 사용법

```python
from data_sources import fetch_realtime_data
from layer2_engine import run_layer2

realtime = fetch_realtime_data()  # mock_realtime_data() 자리에 주입
out = run_layer2(asof="2026-05-01", realtime=realtime)
```

mock vs 실데이터 둘 다 `dict[str, MarketReading]` 반환 — `run_layer2()` 시그니처 변경 없음.

## 검증 결과 (2026-05-01)

실데이터 20/29:
- USDKRW 1473 (alert), WTI 105 (alert+momentum), VIX 17.05 (normal)
- US_10Y 4.42% (+12bp 5d), CPI YoY 3.32%, BOJ_RATE 1.27%
- Lithium 20.92 USD/kg (+3.05%), Nickel 19,270 USD/ton (+6.00%)
- TTF 46.46 (+3.56%)

Layer2Output: 28 variables classified, 1 anomaly (oil_tariff_stagflation), 1 LLM correction queue entry.

## Next (5/1 시점)

- Trading Economics 무료 tier 어댑터 → CN_PMI / CN_PROPERTY / EU_PMI / CN_STIMULUS (key 도착 후)
- ECOS API 어댑터 → KOR_US_SPREAD 등 한국 거시 (L4 KR 모듈 신설 시, key 도착 후)
- SCFI Shanghai Shipping Exchange 공식 페이지 scraping (주1회)
- BOJ_RATE monthly 처리 별도화 (5d 표기 의미 보정)
- layer2_engine.py classify 로직 EURUSD inverted alert 분류 버그 점검

---

# L2 패치 노트 — 2026-05-01 (2회차: ECOS + TE 폐기 결정)

PATCH_NOTES Next 처리 + TE 무료 tier 실측 후 폐기 결정.

## TE (Trading Economics) 무료 tier 폐기 결정

정찰 결과:
- guest:guest credential → HTTP 410 "guest account has been discontinued"
- 가입 후에도 free plan 없음, paid trial만 (Personal ~$75/월)
- ROI 낮음 → **TE 가입 진행 중단**

대체 시도 (FRED OECD-mirror):
- `CSCICP02CNM460S` (CN Consumer Confidence) — 91.6, 26 rows ✅
- `BSCICP02DEM460S` (Germany BCI) — -14.5, 27 rows ✅
- `QCNN628BIS` (BIS China property) — 115.6, quarterly 8 rows ✅
- `CP0000EZ19M086NEST` (EA HICP) — 101.98, 27 rows ✅

**그러나 thresholds.json 임계값과 단위 호환 X**:
- CN_PMI는 PMI 척도 (40~52 임계). CN_CCI(91.6) / DE_BCI(-14.5)와 단위 다름
- 강제 매핑 시 분류 로직 오작동 위험 더 큼 → **mock 유지가 정답**
- 진짜 PMI source는 NBS(중국통계국) / S&P Global Caixin뿐 — 한국 IP WAF 차단

→ CN/EU economic 4개(CN_PMI / CN_PROPERTY / CN_STIMULUS / EU_PMI) **mock 유지 확정 박제**

## 신규 어댑터: fetch_ecos (한국은행 Open API)

- `.env`의 `ECOS_API_KEY` 자동 로드
- 무료, key 1초당 N회 호출 한도 충분
- JSON GET, `StatisticSearch.row[]` 응답
- ttl=1일 캐시 (`~/.cache/layer2_data/ecos_*.pkl`)

ECOS_VARIABLES 매핑 (KOR_US_SPREAD 산출용 raw):
- KOR_10Y_RAW: 817Y002 / 010210000 / D / %

## 신규 derived 변수: KOR_US_SPREAD

`fetch_realtime_data()` 내부에서 산출:
```python
spread = ECOS KOR_10Y - FRED US_10Y
```
- 한미 금리차 (carry trade·자본 흐름 신호)
- L2 rates 그룹의 mock(-1.75) → 실데이터 -0.497% (5/1 기준 한미 역전)

## L4 KR 모듈 후보 시리즈 (이번 회차 미적용, ECOS 매핑만 박제)

향후 L4 KR 신설 시 동일 어댑터로 즉시 사용 가능:
- 한국 기준금리: 722Y001 / 0101000 / D
- KOSPI: 802Y001 / 0001000 / D
- USD/KRW: 731Y001 / 0000001 / D (yfinance USDKRW=X와 비교 검증 가치)
- KOR_CPI: 901Y009 / 0 / M (YoY 후처리)
- KOR_BSI 제조업: 512Y014 / C0000 / M

## 검증 결과 (2026-05-01 2회차)

실데이터 21/29 (이전 20 → +1):
- KOR_US_SPREAD -0.497% (4/30 KR 3.86 - 4/29 US 4.42, 한미 역전 정상 반영)

mock 8개 잔존:
- CN/EU economic 4 (단위 호환 X 박제)
- SCFI / GLOBAL_FLOW / US_TARIFF / OPEC_POLICY

## Next (5/1 2회차 시점)

- SCFI Shanghai 공식 scraping (주1회)
- L4 KR 모듈 신설 시 ECOS 매핑 활용 (KOSPI / 한국 기준금리 / KOR_CPI / KOR_BSI)
- BOJ_RATE monthly 처리 별도화
- layer2_engine.py EURUSD inverted alert 분류 버그 점검
- Policy 텍스트(US_TARIFF / OPEC_POLICY) — news_intel 도킹 + LLM (P3+)

---

# L2 패치 노트 — 2026-05-01 (3회차: Apify Investing.com 캘린더 채택)

DBnomics NBS stale 확인 후 Apify로 우회 보강. **실데이터 21/29 → 25/29 (86% cover)** 달성.

## DBnomics 검증 후 폐기

정찰 결과:
- NBS provider 14개 카테고리 모두 last=2026-02 동결 (5주+ stale)
- DBnomics provider별 cron 독립 — provider-level `indexed_at`은 카탈로그 갱신일 뿐 (함정)
- 안전 provider: OECD DSD_STES / FED H10/H15 / INSEE / BOC. 위험: NBS / BIS / BOJ / WB / ECB EXR

→ NBS / Caixin DBnomics 채택 비추 박제

## Apify 채택 (FREE plan, $5/월 크레딧)

계정: eligible_poster, residential proxy 20GB/월 포함
- Actor: `pintostudio/economic-calendar-data-investing-com` ($0.01/result, 5.0★)
- 한 actor로 China + Germany 2회 호출 → 4개 변수 동시 추출

신규 어댑터 (data_sources.py)
- `_fetch_apify_country(country, ttl=86400)` — 1개월 range 호출, 1일 캐시
- `fetch_apify_variable(name, country, matcher)` — event 매칭 + 가장 최근 actual 추출
- `_parse_apify_date()` / `_parse_apify_value()` — DD/MM/YYYY + B/M/T/K suffix 처리

APIFY_RULES 매핑
- CN_PMI: china + "manufacturing pmi" (NBS Mfg PMI Apr **50.3**)
- CN_PROPERTY: china + "house prices yoy" (NBS 70-city YoY Mar **-3.4%**)
- CN_STIMULUS: china + "loan prime rate" (LPR 5Y Apr **3.5%**)
- EU_PMI: germany + "manufacturing pmi" (HCOB Germany Mfg PMI Apr **51.2**) — Germany proxy 결정 박제

## 정책 박제 (인세열님 결정)

EU_PMI는 Germany proxy 채택:
- pintostudio actor의 country 필터에 "euro zone" / "eurozone" / "EU" 모두 매칭 안 됨
- country 비우고 사후 zone 필터링은 $2.63/월 (무료 한도 50%+)
- Germany는 EU 경제 30%+ + PMI 강한 상관 + 단순
- 가성비 우선

## 비용 시뮬레이션

- China 1회: ~40 events × $0.01 = $0.40
- Germany 1회: ~80 events × $0.01 = $0.82
- 월 1회씩 (1일 캐시) = **$1.22/월**, 무료 $5 한도 25% 사용
- Apify residential proxy 미사용 (현 actor가 자체 proxy 처리)

## 검증 결과 (2026-05-01 3회차)

실데이터 25/29 (이전 21 → +4):
- CN_PMI 50.3 (NBS Manufacturing PMI Apr 30 발표) ← 이전 mock 48.9 대신 진짜 값
- CN_PROPERTY -3.4% (NBS House Prices YoY Mar)
- CN_STIMULUS 3.5% (LPR 5Y Apr 20)
- EU_PMI 51.2 (HCOB Germany Mfg PMI Apr 23, EU proxy)

mock 4개 잔존:
- SCFI (Shanghai 공식 scraping 별도 회차)
- GLOBAL_FLOW (파생 지표, 정량 source 없음)
- US_TARIFF / OPEC_POLICY (정책 텍스트, news_intel 도킹 + LLM P3+)

## Next (5/1 3회차 시점)

- SCFI Shanghai Shipping Exchange 공식 페이지 scraping (주1회 무료)
- US_TARIFF / OPEC_POLICY → news_intel cluster themes 기반 LLM 분류 (P3+ 진입 시)
- L4 KR 모듈 신설 시 ECOS 매핑 활용
- BOJ_RATE monthly 처리 별도화
- layer2_engine.py EURUSD inverted alert 분류 버그 점검
- Apify residential proxy 활용 — NBS/PBoC 직접 scrape 가능 (Investing.com 누락 시 백업)

---

# L2 패치 노트 — 2026-05-01 (7회차 main: Stage 3 — v2 보조 7개)

3단계 작업의 3단계. v2 명세 보조 7개 추가 시도, **5/7 채택**.

## 변경 요약

YFINANCE_TICKERS +2:
- URANIUM: URA (Global X Uranium ETF) — UX=F 미상장으로 ETF 채택. 56.42
- RARE_EARTH: REMX (VanEck Rare Earth ETF) — 미중 기술 갈등 신호. 105.43 (+7.32%)

FRED_SERIES +4:
- NFP_PAYEMS: PAYEMS (Total Nonfarm employment, monthly) — 158,637천명 (Mar)
- UNRATE: UNRATE — 4.3% (Mar)
- FED_BALANCE: WALCL (Fed Total Assets, weekly) — $6.70T
- ECB_BALANCE: ECBASSETSW (ECB Weekly Assets) — €6.22T

FRED_MONTHLY_SERIES +2: PAYEMS, UNRATE 추가 (5d 비교 회피)

신규 어댑터:
- fetch_fao_food(ttl=604800) — FAO HTML scraping (regex), 1주일 캐시
  - "FFPI averaged 128.5" 패턴 추출

thresholds.json +7 변수 정의

## 2개 보류 결정

### 풋콜 비율 (^CPC)
- yfinance ^CPC 미지원 (404)
- CBOE 직접 또는 별도 source 필요
- ROI 작음 → 보류

### BOJ_BALANCE
- FRED JPNASSETS 시리즈 Internal Server Error
- 다른 BOJ balance sheet 시리즈 필요 (보류)

## 검증 (2026-05-01)

Stage 3 7개 모두 정상:
- URANIUM 56.42 / RARE_EARTH 105.43 (+7.32%) / NFP 158,637 / UNRATE 4.3% / FED $6.70T / ECB €6.22T / FAO 128.5

전체:
- **실데이터 40/44 (90.9% cover)** — 직전 33/37 → +7
- mock 4 잔존: SCFI / GLOBAL_FLOW / US_TARIFF / OPEC_POLICY (도킹 branch)
- run_layer2 OK: 44 variables classified, anomalies=1

## v2 명세 cover 평가

핵심 29개:
- 환율 5/6 (USDKRW/USDCNY/USDJPY/EURUSD/DXY) + EM_FX_STRESS (Stage 1)
- 에너지 2/3 (WTI/Brent + HenryHub/TTF) + BDI(BDRY) (Stage 2) — SCFI mock
- 금리·통화정책 5/5 (Fed, US 10Y, 한미 spread, BOJ, ECB)
- 원자재 5/5 (Lithium, Nickel, Copper, Gold, Wheat) + Corn/Soybean/Cobalt (Stage 1)
- 경기 4/4 (US CPI, CN PMI, CN PROPERTY, EU PMI) + US PCE (Stage 1)
- 자본흐름 2/3 (VIX + BTC) — GLOBAL_FLOW mock
- 정책 2/3 (US TARIFF, OPEC POLICY — branch에서) — CN STIMULUS는 LPR 5Y로

보조 7개:
- ✅ NFP / 우라늄 / FAO 식량 / 글로벌 반도체 매출(SOXX) / Fed+ECB 대차대조표 / 희토류
- ❌ 풋콜 / BOJ 대차대조표

→ **v2 명세 본질 충족**. 무료 stack 한계 도달.

## Next

- L3 시나리오 합성 진입 (LLM 첫 도입)
- L4 KR 모듈 (ECOS 매핑 활용)
- branch (feature/news-intel-docking) 1주일 평가 후 main 머지 판단
- thresholds.json 임계값 분기 검토 (4월 기준 → 시점 보정)

---

# L2 패치 노트 — 2026-05-01 (6회차 main: Stage 2 — BDI + 반도체 매출 proxy)

3단계 작업의 2단계. v2 명세 BDI/SIA 처리.

## 변경 요약

YFINANCE_TICKERS +2:
- BDI: BDRY (Breakwave Dry Bulk ETF) — ^BDIY 미상장으로 ETF로 대체. 11.72 (+3.35% 5d)
- SEMI_INDEX: SOXX (iShares Semiconductor ETF) — SIA 매출 직접 scraping 대체. 461.44 (+4.63%)

thresholds.json +2:
- BDI: normal [8, 18] / caution [18, 25] / alert [25, ∞] (BDRY ETF 가격대 기준)
- SEMI_INDEX: normal [350, 600] / caution [600, 750] / alert [750, ∞] (SOXX ETF)

## 결정 박제

### BDI: ^BDIY 미상장 → BDRY ETF 채택
- yfinance ^BDIY → 404 "delisted"
- BDRY (Breakwave Dry Bulk Shipping ETF) — Baltic Dry Index futures roll 상품. 시장 측면 동일 신호
- ETF 가격 자체는 BDI 인덱스와 단위 다르지만 변동률·방향 동일

### SIA Global Semi Sales: scraping 대신 SOXX ETF 채택
- semiconductors.org URL 변경 (404), 정형 데이터 부재
- SIA RSS는 200 OK이지만 매출 숫자 정형 추출 작업량 大
- **SOXX ETF가 가성비**: yfinance 즉시, 일별 갱신, SIA 매출 발표에 시장이 가격으로 반영
- 보조지표 본질(반도체 사이클 판단)에는 시장 반영이 더 빠르고 정확

## 검증 (2026-05-01)

- BDI 11.72 (+3.35% 5d) → normal
- SEMI_INDEX 461.44 (+4.63%) → normal (반도체 사이클 정상)
- **실데이터 31/35 → 33/37 (89% cover)**

mock 잔존 4: SCFI / GLOBAL_FLOW / US_TARIFF / OPEC_POLICY (도킹 branch)

## Next (Stage 3 예정)

v2 보조 7개 일부 즉시 추가 가능:
- NFP (FRED PAYEMS)
- 우라늄 (yfinance UX=F 또는 URA ETF)
- 풋콜 비율 (yfinance ^CPC 검증)
- FAO 식량 (FAO API 무료)
- 희토류 (REMX ETF)
- 중앙은행 대차대조표 (FRED Fed + ECB + BOJ 합산)

---

# L2 패치 노트 — 2026-05-01 (5회차 main: Stage 1 — v2 명세 6개 변수 추가)

> 인세열님 결정: v2 아키텍처 문서(geopolitics_stock_analyzer_architecture_v2.md) 핵심 29 명세 점검 후 누락 변수 보강. 3단계로 나눠 진행. **Stage 1 = 무료 즉시 추가 6개**.

## v2 명세 누락 점검 결과

29개 핵심 중 21~22개만 cover (직전 4회차). 누락 8개:
1. 옥수수 / 대두 (곡물, Wheat만 있었음)
2. 코발트 (배터리 원자재 보강)
3. 비트코인 (자본흐름·심리 보조)
4. 신흥국 통화 스트레스
5. PCE (Fed 선호 인플레, CPI 외 추가)
6. BDI (해운, Stage 2)
7. 반도체 소재 (DRAM 현물가, paywall)
8. 글로벌 자금 흐름 (mock 유지, EPFR 유료)

## Stage 1 신규 어댑터 매핑

YFINANCE_TICKERS +4:
- Corn: ZC=F (시카고 옥수수 선물, USc/bu)
- Soybean: ZS=F (시카고 대두, USc/bu)
- BTC: BTC-USD (USD)
- EM_FX_STRESS: CEW (WisdomTree EM Currency ETF, inverted)

FRED_SERIES +1:
- US_PCE_YoY: PCETRIM12M159SFRBDAL (Trimmed Mean PCE 12-month %, 직접 사용)
  - special-case 처리: 시리즈 자체가 12m % 변동률이라 YoY 계산 불필요

KOMIS_VARIABLES +1:
- Cobalt: HP002 / MNRL0003 / 709 (LME CASH 99.8%, USD/ton, Nickel과 동일 단위)

thresholds.json 신규 정의 6건:
- Corn: normal [350, 600] / caution [600, 800] / alert [800, ∞] (USc/bu)
- Soybean: normal [900, 1400] / caution [1400, 1800] / alert [1800, ∞]
- Cobalt: normal [25K, 45K] / caution [45K, 65K] / alert [65K, ∞] (USD/ton)
- BTC: normal [40K, 100K] / caution [100K, 130K] / alert [130K, ∞], momentum 15%
- EM_FX_STRESS: normal [18.5, 22] / caution [17, 18.5] / alert [0, 17], **inverted** (낮을수록 EM 약세)
- US_PCE_YoY: normal [1.5, 2.5] / caution [2.5, 3.5] / alert [3.5, ∞] (Fed 목표 2%)

## 검증 (2026-05-01)

- Corn 477.25 (+4.89% 5d) — normal
- Soybean 1202.5 (+3.33%) — normal
- BTC 77,244 (-1.80%) — normal
- EM_FX_STRESS 19.38 (-0.10%) — normal
- US_PCE_YoY 2.36% (Mar) — normal (Fed 목표 2% 근접, 합리적)
- Cobalt 55,850 USD/ton — caution (배터리 원자재 회복 국면)

run_layer2 OK: slow=13, fast=22, anomalies=1
**실데이터 31/35 (88.6% cover)** — mock 4 잔존: SCFI / GLOBAL_FLOW / US_TARIFF / OPEC_POLICY (도킹 branch에서 처리 중)

## Cobalt 단위 fix (작업 중 발견)

KOMIS Cobalt LME CASH는 USD/kg 아닌 USD/ton (Nickel과 동일). 첫 시도 thresholds [25, 50] (USD/kg 가정) → 실값 55,850으로 alert 오분류. USD/ton 임계 [25K, 45K]로 재정의 → caution 정상.

## Next (Stage 2 예정)

- BDI (Baltic Dry Index) — yfinance ^BDIY 미상장, BDRY ETF 가능 (검증 완료)
- SIA Global Semiconductor Sales (월간) — semiconductors.org scraping

## Next (Stage 3 예정)

- v2 보조 7개: NFP / 우라늄 / 풋콜 / FAO 식량 / 희토류 / 중앙은행 대차대조표
- 대부분 yfinance/FRED로 가능

---

# L2 패치 노트 — 2026-05-01 (4회차: D 잔류 fix + SCFI 보류)

D 잔류 항목 처리 + SCFI 정찰 후 보류 결정.

## D-1: EURUSD inverted alert 분류 버그 fix (layer2_engine.py)

증상: EURUSD 1.17이 inverted=true임에도 alert로 분류 (실제는 normal 상한 초과 = 매우 좋은 EUR 상태).

원인: `_classify_numeric()` fallback이 모든 범위 밖 값을 alert로 처리. inverted 변수의 normal 상한 초과 = 더 좋은 상태인데 같은 fallback에 떨어짐.

처방:
```python
n_lo, n_hi = normal
if inverted and value >= n_hi:
    return "normal"  # inverted: 더 좋은 쪽
return "alert"
```

검증: EURUSD 1.1745 → level=normal ✅. CN_PMI 50.3, EU_PMI 51.2, CN_PROPERTY -3.4% 등 다른 inverted 변수들도 영향 받지 않고 정상 분류.

## D-1b: categorical spec에 numeric 들어왔을 때 defensive fallback

증상: CN_STIMULUS thresholds가 categorical(`values: [...]`)인데 우리가 Apify로 LPR 3.5% (numeric) 박음 → `_classify_numeric` 호출 → KeyError("normal").

처방: `_classify_numeric` 진입 시 `if "normal" not in spec: return _classify_categorical(str(value), spec)` defensive fallback.

근본 처방: thresholds.json `CN_STIMULUS`를 numeric으로 재정의:
```json
"CN_STIMULUS": {
    "category": "slow",
    "unit": "% (PBoC LPR 5Y, lower = more stimulating)",
    "normal": [3.0, 3.8],
    "caution": [3.8, 4.5],
    "alert": [4.5, 99.0],
    "inverted": false
}
```

## D-2: BOJ_RATE monthly 처리 별도화 (data_sources.py)

증상: BOJ_RATE(IR3TIB01JPM156N)가 monthly 시리즈인데 fetch_fred가 daily처럼 처리 → "5d 변동 +46bp" 표기 (실제로는 5개월 변동).

처방:
```python
FRED_MONTHLY_SERIES = {"IR3TIB01JPM156N", "CPIAUCSL"}

if series_id in FRED_MONTHLY_SERIES:
    prev_m = float(s.iloc[-2])
    reading = MarketReading(
        value=..., prev_value=prev_m,
        pct_change_5d=None, bp_change_5d=None, direction=None,
        asof=...
    )
```

검증: BOJ_RATE 1.27% / pct_5d=None / bp_5d=None ✅. US_CPI_YoY 3.32% / pct_5d=None ✅.

## C: SCFI 보류 결정 (정찰 후)

정찰 결과:
- container-news.com/scfi/ — table 0개 (글 형식, scraping 불가)
- SSE 공식 (en.sse.net.cn): `querySCFI2()` AJAX 동적 로드, 단순 requests 불가
- SSE 차트 이미지 + Tesseract OCR — 복잡, 정확도 위험
- Playwright + AJAX — 안정성 높지만 venv 추가 + Playwright 설치 부담
- 무료 + 안정 자동 source 사실상 부재

결정: **mock 유지 + 별도 회차로 미룸**. SCFI 1개 변수 위해 Playwright 설치하는 ROI 낮음. P3+ 진입 시 news_intel 도킹과 함께 처리하거나 SSE 공식 querySCFI2() 역엔지니어링으로 단순화 후 추가.

## 검증 결과 (2026-05-01 4회차)

실데이터 25/29 유지, mock 4 유지 (D 잔류 fix는 분류 정확도 개선)

분류 정확도 개선:
- EURUSD: alert (잘못) → normal (정상)
- BOJ_RATE: "5d +46bp" (잘못) → 5d 표기 None (정확)
- US_CPI_YoY: 같은 monthly fix 적용
- CN_STIMULUS: KeyError → 정상 분류 (LPR 3.5% = normal)

defensive: categorical/numeric 혼동 시 자동 fallback (향후 source 변경 시 안전)

## Next (5/1 4회차 시점)

- US_TARIFF / OPEC_POLICY → news_intel 도킹 + LLM 분류 (P3+)
- SCFI → SSE querySCFI2() 역엔지니어링 또는 P3+ news_intel 도킹 시 함께
- L4 KR 모듈 신설 (KOSPI/KR 기준금리/KR_CPI/KR_BSI)
- L3 시나리오 합성 진입 (LLM 첫 도입)
- thresholds.json 임계값 분기 검토 (4월 시점 기준 → 5월 시장 반영, 다수 alert 변수가 임계값 시점 차이로 잘못 분류 가능성)
