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
