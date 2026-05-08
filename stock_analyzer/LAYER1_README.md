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
