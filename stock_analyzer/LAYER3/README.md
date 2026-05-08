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
