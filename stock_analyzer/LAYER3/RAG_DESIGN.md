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
