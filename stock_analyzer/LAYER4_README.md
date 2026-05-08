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
