# 지정학 → 주가 분석기 (Stock Analyzer)

`~/Projects/claude/{LAYER1..LAYER5,layer-app}` 프로젝트의 문서 미러.

> ⚠️ news_intel-docs 본래 용도는 news_intel 프로젝트 미러. 본 디렉토리는 별개
> 프로젝트(지정학→주가) 문서를 격리 보관용으로 두 번째 트랙으로 추가.

## 레이어 구조

| Layer | 역할 | 문서 |
|---|---|---|
| L1 | 지정학적 친화도 (4 차원 + KR macro lens) | [LAYER1_README.md](LAYER1_README.md), [PATCH_NOTES](LAYER1_PATCH_NOTES.md) |
| L2 | 데이터 소스 통합 + L1 snapshot 헬퍼 | [LAYER2_PATCH_NOTES.md](LAYER2_PATCH_NOTES.md) |
| L3 | RAG (이벤트 카드) | [LAYER3_README.md](LAYER3_README.md), [RAG_DESIGN](LAYER3_RAG_DESIGN.md) |
| L4 | 섹터별 dynamic weighting | [LAYER4_README.md](LAYER4_README.md) |
| L5 | 종목 단위 (universe + signal) | [LAYER5_README.md](LAYER5_README.md) |
| App | Android 클라이언트 | [layer-app_README.md](layer-app_README.md) |

## 동기화 정책

원본은 `~/Projects/claude/{LAYER*,layer-app}/`. 본 미러는 수기 머지로 갱신.
운영 스냅샷(`LAYER3/snapshots/*.md`)·코드·캐시는 미포함 — 설계·운영 가이드만.
