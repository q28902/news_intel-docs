# 지정학 → 주가 분석기 (Stock Analyzer)

`~/Projects/claude/{LAYER1..LAYER5,layer-app}` 프로젝트의 통합 문서.

> ⚠️ news_intel-docs 본래 용도는 news_intel 프로젝트 미러. 본 파일은 별개
> 프로젝트(지정학→주가) 문서를 단일 파일로 통합 보관용으로 두 번째 트랙으로 추가.

## 목차

- [L1 — 지정학적 친화도 (README)](#l1--지정학적-친화도-readme)
- [L1 — PATCH NOTES](#l1--patch-notes)
- [L2 — PATCH NOTES](#l2--patch-notes)
- [L3 — RAG / 이벤트 카드 (README)](#l3--rag--이벤트-카드-readme)
- [L3 — RAG DESIGN](#l3--rag-design)
- [L4 — 섹터별 dynamic weighting (README)](#l4--섹터별-dynamic-weighting-readme)
- [L5 — 종목 단위 (README)](#l5--종목-단위-readme)
- [layer-app — Android 클라이언트 (README)](#layer-app--android-클라이언트-readme)

## 동기화 정책

원본은 `~/Projects/claude/{LAYER*,layer-app}/`. 본 통합 문서는 수기 머지로 갱신.
운영 스냅샷(`LAYER3/snapshots/*.md`)·코드·캐시는 미포함 — 설계·운영 가이드만.

---

# L1 — 지정학적 친화도 (README)

# 🌏 Geopolitical Affinity Score Engine — 1층 (Layer 1)

> 국제정세 → 주가 영향 분석기의 **1층 모듈**.
> 국가간 관계의 구조적 배경을 측정하는 엔진.

## 역할 (v2 최종안)

- **1층의 책임**: 국가간 관계의 구조적 측정 (배경)
- **이벤트 반응**: 2층 시장 데이터에서 처리 (1층은 안정성 유지)
- **출력 해상도**: 연간 + 월간 (2개)

## 설치

```bash
unzip geopolitical_affinity_package.zip
pip install -e ".[all]"
```

## 최초 셋업

```bash
geoaffinity setup fbic ~/Downloads/Diplometrics_FBIC_Index_....csv
geoaffinity setup gdelt
```

## 2층 인터페이스 — 핵심 3개 출력

### 1. 연간 배경 — `get_snapshot(year)`

```bash
geoaffinity snapshot 2024
geoaffinity snapshot --json
```

```python
from geopolitical_affinity import engine
engine.init()
snap = engine.get_snapshot(2024)
# → key_indices: us_china, us_russia, china_russia, us_allies_avg,
#                 israel_iran, india_china, russia_ukraine, korea_japan,
#                 korea_china, global_tension
```

### 2. 연간 변동 — `get_deltas(year)`

```bash
geoaffinity deltas 2024
```

```python
deltas = engine.get_deltas(2024)
# → significant_changes, grade_transitions, summary
```

### 3. 월간 맥박 — `get_monthly(months=6)`

```bash
geoaffinity monthly --months 6
```

```python
pulse = engine.get_monthly(6)
# → trends (3m vs 6m 모멘텀), key_indices_monthly
```

## 데이터 갱신

```bash
# 원스톱 갱신 (GDELT + 사건 자동감지 + 재초기화)
geoaffinity update              # 승인 모드
geoaffinity update --auto       # 자동 모드

# 사건만 갱신
geoaffinity update --events-only

# 사건 분석
geoaffinity scan --days 30
geoaffinity analyze "북한이 ICBM을 발사했다"
```

## 검증 결과

| 검증 | 적중률 |
|------|--------|
| 등급 적중 | 73% |
| 방향 적중 | 87% |
| 순위 적중 | 100% |
| **종합** | **87%** |

---

# L1 — PATCH NOTES

# L1 패치 노트 — 2026-04-17

테스트 브리핑에서 지적된 3건 이슈 수정.

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| `geopolitical_affinity/gdelt_query.py` | **신규 생성**. GDELT BigQuery 단일 진실 소스 |
| `geopolitical_affinity/bootstrap.py` | `_fetch_gdelt()` / `_try_fetch_gdelt()` → gdelt_query 위임 |
| `geopolitical_affinity/updater.py` | `update_gdelt()` → gdelt_query 위임 (SQL 중복 제거) |
| `geopolitical_affinity/layers/L1_structure.py` | dead code `load_and_compute()` 삭제 |
| `geopolitical_affinity/layers/L4_events.py` | `compute()` year_range 디폴트 None으로 변경 + config 참조 |
| `geopolitical_affinity/config.py` | `L4_YEAR_RANGE_START`, `L4_YEAR_RANGE_BUFFER`, `default_l4_year_range()` 추가 |
| `geopolitical_affinity/engine.py` | FBIC 실제 연도 범위를 L4에 주입 |

## 이슈별 상세

### 이슈 #1 — L4 year_range 하드코딩 (2000, 2027)

**문제**: `L4_events.compute(year_range=(2000, 2027))` 이 하드코딩되어 있어 2027년 이후 데이터 생성 시 잘림.

**해결**:
- `config.py`에 `L4_YEAR_RANGE_START=2000`, `L4_YEAR_RANGE_BUFFER=2` 추가
- `config.default_l4_year_range()` 함수로 현재 연도 + buffer 자동 계산
- `L4_events.compute(year_range=None)` 디폴트를 None으로 하고 실제 사용 시 config에서 가져옴
- `engine.init()`이 FBIC 실제 `year.min()` ~ `year.max() + 1 + buffer`를 계산해서 주입

**검증**:
```
config.default_l4_year_range() = (2000, 2028)  # 2026년 시점
```
내년에 실행하면 자동으로 2029로 확장.

### 이슈 #2 — L1.load_and_compute 반환값 불일치

**문제**: 
- 시그니처는 `-> pd.DataFrame`
- 실제로는 `return compute(fbic), fbic` — 튜플 반환
- engine.py는 이 함수를 **호출하지 않음** (직접 `pd.read_csv + L1_structure.compute` 사용)

**해결**: dead code이므로 삭제. L2/L4의 `load_and_compute`는 실제 사용되므로 유지.

### 이슈 #3 — GDELT SQL 중복

**문제**: 40줄짜리 BigQuery SQL이 `bootstrap.py`와 `updater.py`에 문자 단위로 복붙.

**해결**: `gdelt_query.py` 신설, 세 계층으로 분리:

| 함수 | 역할 |
|------|------|
| `build_query(countries)` | SQL 문자열 생성 (순수 함수, 테스트 가능) |
| `fetch_gdelt_dataframe(...)` | BigQuery 실행 → DataFrame 반환 |
| `fetch_and_save(path, ...)` | 위 둘 + CSV 저장 |

bootstrap/updater 모두 `fetch_and_save()` 한 줄 호출로 통일. 국가 코드 변경, WHERE 절 조건 수정, 기간 파라미터 추가 등 향후 진화가 한 곳에서만 처리됨.

## 하위 호환성

- `bootstrap._fetch_gdelt(project_id, output_path)` 시그니처 유지 (내부적으로 gdelt_query로 위임)
- `updater.update_gdelt(project_id)` 시그니처 유지
- `L4_events.compute(events, year_range)` 시그니처는 동일하나 year_range 디폴트만 변경 (기존 하드코딩 대체)
- **Breaking**: `L1_structure.load_and_compute` 삭제 — 하지만 프로젝트 내 호출처 없으므로 영향 없음

## 검증 결과

```
✅ 모든 모듈 import 성공
✅ CLI 모듈 import 성공
✅ py_compile 전체 통과
✅ gdelt_query.build_query 국가 코드 주입 동작
✅ L4.compute year_range 동적 디폴트 동작
✅ L1.load_and_compute 삭제 확인
```

실데이터(FBIC CSV, GDELT BigQuery)가 있는 환경에서 `engine.init()` 호출 테스트는
Mac mini 로컬 환경에서 세열님이 직접 확인 필요.

---

# L2 — PATCH NOTES

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

---

# L3 — RAG / 이벤트 카드 (README)

# LAYER3 — Global Macro → US Equity Sector Analysis Engine

> **상태**: zero-shot PoC (branch: `feature/layer3-zero-shot`, main 머지 보류)
> **버전**: 0.1.0 (2026-05-01)

v2 아키텍처 문서(`geopolitics_stock_analyzer_architecture_v2.md` §3층) 구현.

## 역할

- **입력**: LAYER2 `Layer2Output` (44개 변수 + anomaly + LLM correction queue + news_events)
- **출력**: 미국 증시 GICS 11섹터 영향 분석
  - `market_phase`: 5단계 (탐욕/강세/보합/약세/공포)
  - `sectors[].impact`: 5단계 (강한 호재 ~ 강한 악재)
  - `sectors[].confidence`: 3단계 (높음/중간/낮음)
  - `sectors[].short_term`, `mid_term`: 시간 지평별 영향 (1~4주 / 1~3개월)
  - `sectors[].rationale`: 한국어 1~2줄 근거

## 방식

LLM 중심 (현재 zero-shot, 향후 이벤트 카드 RAG 추가).

- **모델**: Claude Opus 4.5 via Claude Code CLI (`claude -p --max-turns 1`)
- **호출 비용**: subscription 한도 내 (API 직접 비용 0)
- **호출 시간**: 30~60초/회

## 사용

```python
from data_sources import fetch_realtime_data  # LAYER2
from layer2_engine import run_layer2          # LAYER2
from layer3_engine import run_layer3, format_output

realtime = fetch_realtime_data()
l2 = run_layer2(asof="2026-05-01", realtime=realtime)
l3 = run_layer3(l2)

print(format_output(l3))
```

또는 CLI 시연:
```bash
cd ~/Projects/claude/LAYER3
python3 layer3_engine.py
```

## 출력 예시 (2026-05-01 실데이터 기반)

```
asof: 2026-05-01  |  market_phase: 약세  |  model: claude-opus-4.5-cli
layer2 입력: slow=20, fast=24, anomalies=1, events=2

섹터                        영향         확신     단기         중기
──────────────────────────────────────────────────────────────────────
IT                        악재         중간     악재         중립
Health Care               중립         중간     호재         중립
Financials                악재         중간     악재         중립
Consumer Discretionary    강한 악재    높음     강한 악재    악재
Consumer Staples          호재         중간     호재         중립
Industrials               악재         중간     악재         중립
Energy                    강한 호재    높음     강한 호재    호재
Materials                 호재         중간     호재         중립
Utilities                 호재         중간     호재         중립
Real Estate               악재         높음     악재         악재
Communication Services    중립         낮음     악재         중립
```

자세한 sample: `layer3_output_sample.json`

## 폴더 구조

```
LAYER3/
├── README.md                    # 이 파일
├── layer3_engine.py             # 메인 엔진 (~250줄)
└── layer3_output_sample.json    # PoC 시연 결과 sample
```

## 의존성

- LAYER2 (sibling 폴더, `from data_sources import`, `from layer2_engine import`)
- venv: LAYER1과 공유 (`~/Projects/claude/LAYER1/venv`)
- Claude Code CLI (`/opt/homebrew/bin/claude`)

## 검증된 분석 품질 (2026-05-01)

LLM이 LAYER2 변수를 직접 인용해 근거 생성:
- IT: SEMI_INDEX +4.6% 활용
- Energy: WTI/Brent alert + OPEC 감산 종합
- Consumer Disc.: oil_tariff_stagflation anomaly 직접 인용
- Real Estate: US 10Y +12bp + CN_PROPERTY -3.4% 조합

변수 충돌 자동 인식 → confidence "낮음" 표기 (Communication Services 등)

## branch 전용 사유

- 1주일 평가 후 main 머지 결정
- 평가 항목:
  - 매일 1~2회 호출 → 시장 결과와 대조
  - rationale 품질 (LAYER2 변수 활용도)
  - 모순 분석 일관성
  - 비용 (subscription 한도 영향)

## Next (zero-shot 안정 후)

- **이벤트 카드 RAG**: v2 부록 C "다음 단계 1" — 이벤트 카드 210건 재정리 → 과거 패턴 레퍼런스
- **LAYER4 한국 증시 번역기**: L3 출력 + 한국 고유 변수 → 19섹터 영향
- **피드백 루프**: 예측 vs 실제 대조 (월간 회고)

## 변경 이력

- **2026-05-01 v0.1.0**: zero-shot PoC, branch `feature/layer3-zero-shot`

---

# L3 — RAG DESIGN

# 이벤트 카드 RAG 설계

> v2 부록 C "다음 단계 1: 이벤트 카드 재정리 210건" 처리.
> 목표: zero-shot LAYER3/LAYER4 → few-shot RAG로 정확도 향상.

---

## 1. 카드 schema (확정안)

```json
{
  "id": "evt_2024-04-13_iran_strike_israel",
  "date": "2024-04-13",
  "title": "이란, 이스라엘 본토 직접 미사일·드론 공격",
  "event_type": "geopolitics_oil",
  "tags": ["middle_east", "iran_israel", "oil_spike", "energy", "defense", "risk_off"],

  "context": {
    "trigger": "이란 IRGC가 시리아 다마스쿠스 영사관 폭격(4/1)에 대한 보복으로 이스라엘 영토에 직접 공격",
    "preconditions": {
      "WTI": "85.5 (caution 진입 직전)",
      "VIX": "16.2 (normal)",
      "DXY": "105.9",
      "Gold": "2380"
    },
    "background_l1": {"israel_iran": 1.2, "us_china": 2.0, "global_tension": 4.1},
    "regime": "trump_admin_2024 / fed_pause / china_property_crisis"
  },

  "market_response": {
    "horizon_1d": {
      "WTI": "+0.8%", "Brent": "+0.7%", "Gold": "+0.5%", "VIX": "+12%",
      "SP500": "-1.2%",
      "us_sectors": {"Energy": "+1.8%", "Industrials": "-0.9%", "Consumer Discretionary": "-1.5%"},
      "kr_sectors": {"정유": "+2.3%", "방산": "+3.1%", "자동차": "-1.8%"}
    },
    "horizon_5d": {...},
    "horizon_1m": {...}
  },

  "lessons": "이란→이스라엘 직접 충돌 시 (1) 유가는 기대만큼 안 오름 (이미 선반영) (2) 방산 강세 1주 지속 (3) 글로벌 risk_off → KOSPI 동조 약세, 단 정유·방산만 디커플링",

  "confidence": "high",
  "source": "manual_curation",
  "card_version": 1,
  "created_at": "2026-05-02T10:00:00Z",
  "linked_cards": ["evt_2024-04-19_israel_counter_strike", "evt_2025-06-13_iran_israel_war"]
}
```

### 핵심 필드 의도
- **tags**: 1차 필터 (검색 boost)
- **preconditions**: 현재 시점 LAYER2 변수와 유사도 매칭에 활용
- **background_l1**: LAYER1 key_indices 비교 (구조적 유사성)
- **regime**: 거시 국면 (trump 1기/2기, Fed cycle 등) — 시간 거리 보정용
- **market_response**: 시간 지평 3종 (1d / 5d / 1m), L2 변수 + L3 섹터 + L4 섹터 모두
- **lessons**: LLM이 직접 인용할 1~2 문장 결론
- **linked_cards**: 동일 사건의 후속 또는 유사 사건 chain

---

## 2. 저장소·인프라

### Qdrant 채택 (이미 가동 중)
- collection: `event_cards`
- vector dim: **1024** (bge-m3 임베딩, news_intel·law-bot과 동일)
- distance: cosine
- hybrid: dense (semantic) + sparse (BM25 tags) — 추후 추가
- payload: 위 schema 전체

### 이유
- bge-m3 embed server (localhost:5051) 이미 안정 가동
- law_cases_hybrid (71K) 운영 경험 → 코드 패턴 재사용
- Python qdrant-client 안정

### 대안 (포기)
- SQLite + sentence-transformers: 단순하지만 hybrid X, scalability X
- pgvector: news_intel pg에 별도 extension. 복잡도 ↑

---

## 3. 임베딩 텍스트 구성

bge-m3에 입력할 단일 텍스트:

```
[제목] {title}
[날짜] {date}
[유형] {event_type}
[태그] {", ".join(tags)}
[발생 맥락] {context.trigger}
[당시 환경] WTI {preconditions.WTI}, VIX {preconditions.VIX}, israel_iran {background_l1.israel_iran}
[교훈] {lessons}
```

→ 약 200~400자. bge-m3 1024 dim으로 임베딩.

---

## 4. 검색 알고리즘

### 입력
현재 시점 LAYER2 + LAYER1 + news_events:
```python
query_text = f"""
[현재 변수] WTI {l2.WTI.value} ({l2.WTI.level}), VIX {l2.VIX.value}, USDKRW {l2.USDKRW.value}, ...
[anomaly] {l2.anomalies[0].rule_id if l2.anomalies else "none"}
[L1 긴장] israel_iran {l1.israel_iran}, us_china {l1.us_china}
[news] {news_events[0]['headline'][:200]}
"""
```

### 검색 단계
1. query_text → bge-m3 embed (1024 dim)
2. Qdrant search top-K=10
3. **boost**: 현재 변수의 anomaly tag와 카드 tags 일치 시 score +0.1
4. **time decay** (optional): 1년 이상 카드는 score ×0.95, 3년 이상 ×0.9
5. final top-N=5 카드 반환

### Hybrid 보강 (Phase 2)
- BM25 sparse vector (tags + lessons 키워드)
- dense×0.7 + sparse×0.3 (law-bot 70/30 패턴 차용)

---

## 5. RAG 통합 (LAYER3/LAYER4 프롬프트)

### LAYER3 prompt 변경

```python
SYSTEM_PROMPT = """... (기존) ...

[과거 유사 사례 N건] (RAG retrieval, time-sorted)
{card_1.title} ({card_1.date}, {card_1.tags})
  당시 환경: {card_1.preconditions}
  결과 (1d/5d/1m): {card_1.market_response}
  교훈: {card_1.lessons}

{card_2 ...}

분석 시 위 사례를 reference로 활용. 단 현재 변수와 다른 부분은 명시적으로 차이 인지."""
```

LLM이 cards 보고 패턴 인용:
- "2024-04-13 이란 공격 시 유가가 기대만큼 안 올랐던 것처럼, 현재도 WTI는 이미 선반영…"

---

## 6. 카드 누적 흐름

### Phase 0: 수동 seed (사용자 보유 210건)
- 사용자 카드 보유 형태?
  - 별도 파일 (Notion / Markdown / Excel)?
  - 이전 ChatGPT 세션?
  - 또는 우리가 news_intel archive에서 추출?

### Phase 1: 자동 추가 (forecast 평가 후)
```
화요일 09:00 평가 → 적중률 70%+ 사례를 자동 카드 후보 큐
→ LLM이 카드 schema로 변환 (event/preconditions/response/lessons)
→ Qdrant ingest
```

### Phase 2: news_intel 도킹
```
news_intel cluster importance > 5000 + multilang count > 30
→ "주요 글로벌 의제" 후보
→ 7일 후 market_response 측정 + 카드화 (자동)
```

---

## 7. 작업 단계

### Step 1: PoC 인프라 (반나절)
- [x] schema 확정 (이 문서)
- [ ] `event_cards/cards/` 디렉토리 + 샘플 5건 수동 작성
- [ ] Qdrant collection 생성 스크립트
- [ ] ingest 스크립트 (JSON → embed → Qdrant upsert)
- [ ] search 함수 (`search_similar_cards(query_text, top_k=5)`)

### Step 2: LAYER3/LAYER4 통합 (반나절)
- [ ] LLM 프롬프트에 RAG 섹션 추가 (`build_prompt`)
- [ ] zero-shot vs few-shot 비교 평가 (5/4 forecast 결과로)
- [ ] branch: `feature/event-cards-rag`

### Step 3: 자동화 (1일)
- [ ] forecast 평가 적중 사례 자동 카드 추출
- [ ] news_intel cluster importance threshold 자동 큐잉
- [ ] 주간 카드 갱신 launchd

### Step 4: seed 210건 (사용자 협업)
- [ ] 카드 source 결정 (수동 vs 자동 추출)
- [ ] LLM 보조 카드화 (현재 markdown/notion → schema 변환)
- [ ] Qdrant 일괄 ingest

---

## 8. 결정 필요 (사용자 확인)

### Q1. 카드 210건 source
- A. 사용자 보유 별도 파일 (Notion/MD/Excel) — 제공 시 우리가 자동 변환
- B. 새로 작성 (LLM 보조로 historical 사건 schema화) — 시간 더 걸림
- C. news_intel archive에서 추출 (importance 상위) — 자동, 단 raw 카드 (lessons 부재)

### Q2. seed 시점
- A. PoC 인프라 먼저 완성 (5~10건 수동 seed) → 가동 → seed 채우기
- B. seed 210건 먼저 모두 채운 후 가동

→ **A 추천** (점진 누적)

### Q3. branch 또는 main
- A. branch `feature/event-cards-rag` (zero-shot vs few-shot 비교 후 머지)
- B. main 직접 (RAG는 부가 기능)

→ **A 추천** (1주일 평가 후 main 머지)

---

# L4 — 섹터별 dynamic weighting (README)

# LAYER4 — Korean Equity Sector Analysis Engine

> **상태**: zero-shot PoC (branch: `feature/layer4-kr`, main 머지 보류)
> **버전**: 0.1.0 (2026-05-01)

v2 아키텍처 §4층 한국 증시 번역기.

## 역할

- **입력**:
  - LAYER3: 미국 GICS 11섹터 영향 (직렬 70~80%)
  - LAYER2: 한국 분석에 직접 영향 큰 글로벌 변수 (병렬 20~30%, USDKRW/CN_PMI/SCFI/BDI/SEMI_INDEX 등)
  - LAYER1: 한중·한일·미중·글로벌 긴장 점수 (구조적 배경)
  - 한국 고유: KOSPI / 기준금리 / CPI / BSI / 외인매매(mock) / n8n news_kr_daily / news_intel 한국 cluster
- **출력**:
  - 한국 19섹터 영향 5단계 + 확신도 3단계 + 시간 지평 2단계 + 한국어 rationale

## 한국 19섹터

반도체 / IT-소프트웨어 / 조선 / 방산 / 건설 / 일반산업재 / 정유 / 원자력 / 2차전지-소재 / 철강-화학 / 자동차 / 소비재(임의) / 소비재(필수) / 바이오-헬스케어 / 금융 / 통신 / 엔터-콘텐츠 / 유틸리티 / 부동산

## 방식

LLM 중심 (zero-shot). LLM이 직렬·병렬 비중 동적 조정.
- **모델**: Claude Opus 4.5 via Code CLI (`claude -p --max-turns 1`)
- **비용**: subscription 한도 내
- **소요**: 30~60초/회

## 사용

```python
from data_sources import fetch_realtime_data
from layer2_engine import run_layer2
from layer3_engine import run_layer3
from data_sources_kr import fetch_all_kr
from layer4_engine import run_layer4, format_output

l2 = run_layer2(asof="2026-05-01", realtime=fetch_realtime_data())
l3 = run_layer3(l2)
kr_data = fetch_all_kr()
l4 = run_layer4(l3, l2, kr_data)

print(format_output(l4))
```

CLI 시연:
```bash
cd ~/Projects/claude/LAYER4
python3 layer4_engine.py
```

## 한국 데이터 source

| 변수 | source | 상태 |
|---|---|---|
| KOSPI | yfinance ^KS11 | ✅ |
| 기준금리 | ECOS 722Y001 | ✅ |
| CPI YoY | ECOS 901Y009 (후처리) | ✅ |
| BSI 제조업 | ECOS 512Y014 | ✅ |
| 외인매매 | mock | ❌ pykrx 막힘, ECOS 시리즈 미식별 (별도 회차) |
| 한국 정치/사경 뉴스 | n8n news_kr_daily 테이블 | ⚠️ n8n 워크플로우 수정 후 활성 |
| 한국 cluster | news_intel (KOR locations + title 매칭) | ✅ 10건 |

## 폴더 구조

```
LAYER4/
├── README.md                # 이 파일
├── data_sources_kr.py       # 한국 변수 + n8n + news_intel fetcher (~280줄)
├── layer4_engine.py         # 메인 엔진 (~280줄)
└── snapshots/               # daily_snapshot이 LAYER3과 공유 (LAYER3/snapshots에 통합 저장)
```

## 의존성

- LAYER2, LAYER3 (sibling 폴더)
- venv: LAYER1과 공유 (`~/Projects/claude/LAYER1/venv`)
- Claude Code CLI
- 추가 패키지: pykrx (현재 KRX 정책으로 작동 제한, 외인 매매 보류)

## daily 자동화

LAYER3의 `scripts/daily_snapshot.py`가 L4까지 한 번에 처리 (Q3 결정).
- launchd `com.layer3.daily` (매일 18:00 KST)
- 결과: `LAYER3/snapshots/YYYY-MM-DD.json` (L3 + L4 통합) + `.md`

## 검증된 분석 품질 (2026-05-01)

LLM이 입력 layer를 직접 인용해 rationale 생성:
- **조선**: L2 SCFI alert + BDI + Brent → 한국 직접 수혜 (LNG선)
- **자동차**: L3 Consumer Disc + L2 US_TARIFF escalating → 미 관세 직격
- **정유**: L3 Energy + L2 Brent alert → 정제마진 확대
- **방산**: L1 global_tension + L2 Brent + 트럼프 유럽철군 → 자주국방
- **유틸리티**: L3 Utilities 호재 vs L2 유가 alert → **미국과 반대 결론** (한국 한전 연료비 부담, LLM이 차이 인지)

변수 충돌 자동 인식 → confidence "낮음" (2차전지/일반산업재/통신/엔터)

## branch 전용 사유

- 1주일 평가 후 main 머지 결정
- 평가 항목:
  - L4 시장 국면 vs KOSPI 실제 등락 대조
  - 19섹터 vs 실제 KRX 섹터 등락
  - rationale 품질 (입력 layer 활용도)
  - n8n news_kr_daily 활성화 후 정확도 변화
- 1주일 후 회고: 일치율 / 모순 패턴 / LLM 비용 (subscription 영향)

## Next (zero-shot 안정 후)

- **외인 매매 source 확보**: ECOS 외인 시리즈 식별 또는 KRX 인증 우회
- **n8n news_kr_daily 가동** (인세열님 직접 워크플로우 수정 후)
- **이벤트 카드 RAG** (v2 부록 C 다음 단계)
- **5층 종목 필터링** (DART + 증권사 리포트, 19섹터 × 대장주 3 = 57종목)
- **피드백 루프**: 예측 vs 실제 KRX 등락 대조

## 변경 이력

- **2026-05-01 v0.1.0**: zero-shot PoC, branch `feature/layer4-kr`
  - data_sources_kr.py + layer4_engine.py
  - daily_snapshot에 L4 통합 (Q3 결정)

---

# L5 — 종목 단위 (README)

# LAYER5 — 한국 종목 특화 분석 엔진

> ## ⚠️ Disclaimer (개인 연구용)
> **This system generates research signals for personal study only. NOT investment advice.**
> 본 시스템은 개인 연구·학습 목적의 신호 생성기입니다. 자본시장법상 투자권유·자문에 해당하지 않으며, 운용·손익에 대한 책임은 사용자에게 있습니다.
> - 자기 사용 한정. 외부 배포·상업화 금지
> - LLM 비결정성·데이터 lag·mock 잔존으로 모든 출력은 reference only
> - watchlist.json은 `.gitignore` 강제 (개인 보유 정보 절대 미공개)

> **상태**: 골격 단계 (PoC 코드 미작성)
> **버전**: 0.0.1 (2026-05-03 설계 초안)

LAYER4 19섹터 영향 분석 → **개별 종목 단위 신호 생성**.

## 역할

- **입력**:
  1. LAYER4 `Layer4Output` (19섹터 phase + impact + confidence + rationale)
  2. 한국 종목 OHLCV (FinanceDataReader)
  3. 외국인·기관 매매 (pykrx)
  4. 분기 재무제표 + 공시 (DART)
- **출력**: 종목별 trade signal
  - `signal`: 강한 매수 / 매수 / 중립 / 매도 / 강한 매도
  - `confidence`: 높음 / 중간 / 낮음
  - `horizon`: 단기 (1~4주) / 중기 (1~3개월)
  - `target_price`, `stop_loss`: 구체적 trade plan
  - `portfolio_weight`: 추천 비중 (0~1)
  - `rationale`: L4 섹터 영향 + 종목 정량·펀더멘털 + 외인 수급 종합 한국어 근거
  - `risk_factors`: 종목별 내생·외생 리스크

## 종목 universe (사용자 결정)

**섹터별 시총 상위 5 + 사용자 watchlist**:
- LAYER4 19섹터 × 5종목 = **95종목** baseline
- + `watchlist.json` 보유·관심 종목 추가
- 중복 제거 후 최종 universe (예상 100~120종목)

## 호출 빈도 (사용자 결정)

**D = 매일 + 이벤트 트리거**:
- 정기: 매일 18:00 KST launchd (LAYER3·4와 동시)
- 이벤트: LAYER1 `crisis_mode=true` 활성 또는 LAYER4 `market_phase` 변동 시 즉시 트리거

## 분석 방식

**zero-shot LLM** (Claude Opus 4.5, LAYER3·4 통일) — 1차 PoC

종목별 prompt에 박는 데이터:
- L4 해당 섹터의 impact·confidence·rationale (직접 인용 의무)
- OHLCV 60일 + 5d/1m/3m 변동률
- 외국인 5d/30d 누적 순매수
- PER·PBR·ROE·시총·외인 보유율
- 최근 30일 KRX 공시 (실적·자사주·신주발행)
- 사용자 화이트리스트 종목은 holdings 정보(평단·비중)도 함께 → 매수/매도/홀드 결정

**watchlist 우선 깊이 분석 + 나머지 universe 간단 분석** (사용자 결정 C).

## 데이터 source

- **FinanceDataReader** (`fdr.DataReader`): KOSPI/KOSDAQ OHLCV
- **pykrx** (`stock.get_market_*`): 외국인·기관 매매, 시총, 업종분류
- **DART API** (`dart-fss` 또는 직접 호출): 분기 재무제표, 공시

API key:
- DART API 무료, 사용자 발급 필요 (https://opendart.fss.or.kr/)
- 환경변수 `DART_API_KEY` 박기

## 섹터 매핑

**KRX 업종분류 → LAYER4 19섹터** (`sector_mapping.py`)

KRX 대분류는 33개 정도, LAYER4는 19섹터라 N:M 매핑.
- 1차: KRX 업종 → LAYER4 섹터 정적 dict
- 종목별 override (예: 삼성전자=전기전자 KRX → 반도체 LAYER4)
- override는 `sector_mapping.py` 안 `STOCK_OVERRIDES` dict

## 폴더 구조

```
LAYER5/
├── README.md                    # 이 파일
├── layer5_engine.py             # 메인 엔진 (run_layer5)
├── data_sources_stocks.py       # FDR + pykrx + DART fetcher
├── universe.py                  # 섹터별 시총 상위 5 + watchlist 결합
├── sector_mapping.py            # KRX 업종 → LAYER4 19섹터 매핑
├── watchlist.json               # 사용자 화이트리스트 (보유·관심)
├── scripts/
│   ├── generate_signals.py      # 매일 18:00 launchd
│   └── eval_signals.py          # 신호 vs 실제 평가 (1d/5d/1m)
└── output/
    └── target_YYYY-MM-DD_predicted_at_YYYY-MM-DD.json
```

## 의존성 (다음 회차 설치)

```bash
source ~/Projects/claude/LAYER1/venv/bin/activate
pip install finance-datareader pykrx dart-fss
```

- LAYER1·2·3·4 sibling
- venv: LAYER1과 공유
- Claude CLI (`/opt/homebrew/bin/claude`)
- DART API key (사용자 발급)

## 운영 흐름

```
매일 18:00 KST:
  LAYER1 → LAYER2 → LAYER3 → LAYER4 → LAYER5
                                          ├── universe = sector_top5 + watchlist
                                          ├── data_sources_stocks.fetch_all(universe)
                                          ├── sector_mapping.map_each(universe)
                                          ├── run_layer5(l4, stocks_data, watchlist)
                                          └── 저장 + Telegram 알림
```

이벤트 트리거: LAYER1 crisis_mode 또는 LAYER4 phase 변동 시 별 hook이 generate_signals.py 호출.

## 평가

- 매일 신호 → 1d/5d/1m 실제 변동 대조
- 정확도 통계 (signal == 강한매수 → 5d 수익률 분포 등)
- 월간 회고

## branch 전략

- `feature/layer5-poc` 별 branch
- 1주일 평가 후 main 머지 결정 (LAYER3·4 패턴 따름)

## Next (다음 회차)

1. 의존성 설치 + DART API key 환경변수 박기 (사용자 액션)
2. `sector_mapping.py` KRX 33 업종 → LAYER4 19섹터 정적 dict 박기
3. `data_sources_stocks.py` 4종 fetcher 작성 (OHLCV / 외인매매 / 재무 / 공시)
4. `universe.py` 섹터별 시총 상위 5 + watchlist 결합
5. `layer5_engine.py` LLM prompt builder + run_layer5 (LAYER3·4 패턴)
6. `watchlist.json` 사용자 보유·관심 종목 박기
7. PoC 시연 — 1종목 (예: 삼성전자) 1회 호출 + 결과 검증

## 변경 이력

- **2026-05-03 v0.0.1**: 디렉토리·README 골격 (사용자 결정 1.D / 2.C / 3.D / 4.C 반영)

---

# layer-app — Android 클라이언트 (README)

# Geostock — 토스증권 스타일 UI 재작업 (통합 zip)

본 zip 은 **Phase 1+2+3+4 누적**. 원본 프로젝트(`geostock-layer-app`)에 한 번만 덮어쓰면 끝.

## 적용

```bash
cd /path/to/geostock                                    # 원본 프로젝트 루트
unzip -o ~/Downloads/geostock-toss-redesign-complete.zip
./gradlew assembleDebug                                 # 빌드 검증
./gradlew installDebug                                  # 디바이스 설치
```

`unzip -o` 의 `-o` 는 기존 파일 덮어쓰기 (overwrite). 충돌 프롬프트 없이 진행.

---

## 변경 요약

### Phase 1 — 디자인 토큰 + 폰트 (foundation)

| 파일 | 역할 |
|---|---|
| `app/src/main/res/font/pretendard_{regular,medium,semibold,bold}.otf` | Pretendard 4 weights 번들 (~1.3MB) |
| `app/src/main/java/.../ui/theme/Color.kt` | `GeostockColors` 데이터클래스 + Light/Dark 컬러 시스템. 시멘틱 토큰: bgBase/bgCard/bgSubtle, textPrimary/Secondary/Tertiary, brand, brandSubtle, up/upSubtle (한국식 빨강), down/downSubtle (파랑), warning/success/error 등. |
| `app/src/main/java/.../ui/theme/Type.kt` | `GeostockTypography` — display/title/body/label + numeric (tabular nums) 스타일. Pretendard family 자동 매핑. |
| `app/src/main/java/.../ui/theme/Shape.kt` | `GeostockShapes` (4/8/12/16/20dp). |
| `app/src/main/java/.../ui/theme/Theme.kt` | `GeostockTheme` provider + `LocalGeostockColors`/`LocalGeostockTypography`/`LocalGeostockShapes`. Material3 fallback 도 동시 제공. |

### Phase 2 — 공통 컴포넌트 분리 (`ui/components/`)

| 파일 | 컴포넌트 |
|---|---|
| `Card.kt` | `TossCard(padding, radius, background, onClick)` |
| `TickerIcon.kt` | Toss CDN 로고 + 첫 글자 fallback |
| `ScreenHeader.kt` | `ScreenHeader(title, subtitle?)` (28sp Bold + 14sp gray 통일) |
| `FoldableSection.kt` | chevron 회전 애니메이션 폴딩 헤더 |
| `Pill.kt` | `Pill(label, selected, onClick)` 풀라운드 chip |
| `Numeric.kt` | `BigPrice(value)`, `ChangeText(pctText, prefix?, style)` (한국식 등락 색·부호 자동), `MetricRow(label, value)` |
| `States.kt` | `SkeletonCard`, `LoadingCard(message?)`, `AnalyzingCard(statusLabel, elapsedSec, extra?)`, `EmptyCard(message)`, `ErrorCard(message, label?, onRetry?)` |
| `StarToggle.kt` | 이모지 → Material Icons (Filled.Star / Outlined.StarBorder / Outlined.WorkOutline) |

### Phase 3 — 하단 네비 + edge-to-edge

| 파일 | 변경 |
|---|---|
| `app/build.gradle.kts` | `androidx.compose.material:material-icons-extended` 의존성 추가 |
| `MainActivity.kt` | `enableEdgeToEdge()`, `NavigationBar(tonalElevation=0.dp)`, `indicatorColor=Transparent`, 활성 brand color 처리. 4탭 모두 Material Icons (Home/StarBorder/Search/Schedule). 라우팅 로직(home/watchlist/search/history + detail/{jobId}/history/{file}/chart/{ticker}/{name}) 그대로 보존. |

### Phase 4 — 화면별 적용

| 화면 | 핵심 변경 |
|---|---|
| `HomeScreen.kt` | ScreenHeader, ForecastCard (국기 🇺🇸🇰🇷 유지), FoldableSection 보유/관심, HoldingRow/InterestRow (TickerIcon + ChangeText). polling 10s 그대로. |
| `WatchlistScreen.kt` | ScreenHeader, FoldableSection, HoldingCard/InterestCard. 안내문 톤다운. |
| `SearchScreen.kt` | ScreenHeader, AnalyzeButton primary, OutlinedTextField toss 컬러, StarToggle. 이모지(✅❌⚠️) → 텍스트/색 처리. |
| `DetailScreen.kt` | ScreenHeader, AnalyzingCard, ResultView (헤더 → 큰 시그널박스 → MetricRow 표 → SectionCard). signalColor 한국식 매수=up/매도=down. 이모지 모두 제거. |
| `HistoryScreen.kt` | ScreenHeader, 카드 폴딩 mutableStateMapOf 그대로. 국기 유지, 이모지 제거. |
| `ChartScreen.kt` | TickerIcon + 종목명 헤더, BigPrice + ChangeText, 통계 카드, period chip → Pill 컴포넌트, MAIN_PERIODS 6개 + DropdownMenu. CandleChart 호출에 토큰 컬러 주입. |
| `CandleChart.kt` | 하드코딩 컬러 → 파라미터로 추출 (`gridColor`, `labelColor`, `upColor`, `downColor`, `highlightColor`). 핀치 줌·팬 로직 그대로. |

---

## 디자인 원칙

- **한국 주식 등락 컬러**: 상승=Red(`up=#F04452`), 하락=Blue(`down=#3182F6`). `ChangeText` 컴포넌트가 자동 처리.
- **카드**: 큰 카드 padding=20dp/radius=20dp, 행 카드 padding=16dp/radius=16dp.
- **타이포**: 제목 displayMedium(28sp Bold) + 설명 bodyMedium(14sp gray). 숫자는 numeric* (tabular).
- **이모지 정책**: UI 이모지 모두 제거 → Material Icons. **국기 🇺🇸🇰🇷 만 예외**(시각적 정보 가치).

## 빌드 환경 가정

- compose-bom 2024.12.01
- compileSdk/targetSdk 35, minSdk 26
- Kotlin 1.9.x + compose plugin
- Note 20 Ultra (Android 13) 호환

## 컬러 시스템 빠른 참조

| 토큰 | Light | Dark |
|---|---|---|
| `bgBase` | `#F2F4F6` | `#181A20` |
| `bgCard` | `#FFFFFF` | `#222428` |
| `bgSubtle` | `#F2F4F6` | `#2A2D33` |
| `brand` | `#3182F6` | `#4493F8` |
| `up` (한국식 상승=빨강) | `#F04452` | `#FF5765` |
| `down` (한국식 하락=파랑) | `#3182F6` | `#4493F8` |
| `textPrimary` | `#191F28` | `#F2F4F6` |
| `textTertiary` | `#8B95A1` | `#6B7280` |

상세는 `ui/theme/Color.kt` 참조.

## 향후 마이그레이션 (옵션)

`ui/Theme.kt` 의 backward compat shim (`TossColors`, `GeostockTheme`, `TossCard`, `TickerIcon`)은 외부 진입점 호환용. 모든 화면이 이미 신규 위치(`ui.theme.*`, `ui.components.*`) 사용 중이므로 shim 제거 가능. 단 외부 코드 import 안전 확인 후 제거.
