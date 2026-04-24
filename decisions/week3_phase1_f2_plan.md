# F2: 데이터 품질 필터링 + 엔티티 중복 제거 (A+ 경로)

결정일: 2026-04-24
선행: `docs/decisions/week3_phase1_embedding.md` 정정 기록

> **주의**: 이전 "언어별 2단계화" 계획은 폐기. 본 문서가 F2의 단일 진실.

## 방향 전환 근거

F1 실패 원인은 **언어 분리 문제가 아니라 GKG 메타 데이터 품질 한계**. 언어 2단계화는 잘못된 진단 위에 세워진 계획이라 폐기.

증거 요약 (week3_phase1_embedding.md 정정 기록):
- NULL language 기사 15.5% — persons 메타 빈약 (39% 수준)
- top2 cluster 기사당 unique persons 3.14명 → 주제 일관성 없음
- 엔티티 mention 중복 미처리 → "Alienware 13회" 류 지배 신호
- 입력 빈약 극단치 (p10=154자, 최소 43자 사례)

## 필터링 설계 (순차 적용)

| 필터 | 기준 | 예상 drop율 |
|---|---|---|
| **F1** | `article.language` IS NOT NULL AND ≠ '' | ~15.5% |
| **F2** | `extras_raw->>'title'` NOT NULL AND `len(title.strip()) >= 10` | 추가 5~10% |
| **F3** | `|unique(persons ∪ orgs ∪ locations names)|` ≥ 3 (case-insensitive) | 추가 10~20% |

**예상 입력 규모**: 200,000 → **120,000~150,000** (Run 7 실측으로 확정).

filter_stats는 매 run system_log의 Phase 1 details JSONB에 `f2_filter_stats` 키로 기록:
```json
{
  "f2_filter_stats": {
    "total_input": 200000,
    "dropped_null_lang": 31050,
    "dropped_short_title": 12340,
    "dropped_few_entities": 18910,
    "total_output": 137700
  }
}
```

## 입력 함수 변경 (엔티티 중복 제거)

`_build_embedding_text`에 case-insensitive set 기반 중복 제거 추가.
- title 먼저 parts에 append
- persons/organizations/locations 순회 시 `name.lower().strip()` 키로 seen set 체크
- 중복 엔티티 이름은 최초 1회만 concat
- `TEXT_CHAR_CAP=2000` 유지
- 원문 형태(case 보존)는 그대로, seen 체크만 소문자 기준

## 알고리즘 유지 (단일 변인 원칙)

F1과 **완전히 동일**:
- 모델: BAAI/bge-m3, MPS, batch_size=64, max_seq_length=256, normalize_embeddings=True
- UMAP: n_components=50, n_neighbors=30, min_dist=0, metric=cosine, random_state=42, low_memory=True
- HDBSCAN: min_cluster_size=20, min_samples=5, metric=euclidean, cluster_selection_method=eom

파라미터 튜닝 **금지** (필터 + 중복 제거 효과 순수 측정 목적).

## 실패 모드 사전 명시

| 결과 | 판정 | 다음 조치 |
|---|---|---|
| max_size < 500 AND noise 10~25% | **성공** | main 머지 검토 |
| max_size 500~1200, noise 적정 | **부분 성공** | 파라미터 튜닝 여지 (min_cluster_size / eom→leaf) |
| max_size > 1500 | **데이터 품질로 해결 안 됨** | D 경로(알고리즘 재설계) 재검토 |
| 모집단 < 80,000 | **필터 과잉** | F3 임계 완화 (≥3 → ≥2) |

## 미완 사항 (Run 7 실측 후 결정)

- 87분 임베딩 시간 문제: 필터로 입력 200k → 135k 줄면 ~65분 단축. 여전히 1h 주기 목표엔 미달. D 경로에서 별도 해결.
- MAX_ARTICLES=200,000 상한 유지 여부: Run 7 속도 보고 결정.
- cluster_runs에 `details` JSONB 컬럼 신설 여부: 현재는 system_log.details 사용. 운영 쿼리 편의 필요시 migration 005 검토.

## 브랜치 / 커밋

- 브랜치: `feat/f1-bge-m3-hdbscan` 계속 사용 (F2는 F1의 확장이라 별도 브랜치 불요)
- F1 WIP commit `5ab8b92` 위에 F2 변경분 추가 커밋
- Run 7 결과는 본 문서 하단 "## Run 7 결과" 섹션으로 append

---

## Run 7 결과 (2026-04-24)

### 실행 환경
- 브랜치: `feat/f1-bge-m3-hdbscan` HEAD `2c6fba2`
- PID 97424, `[START] 06:41:43 KST`, `[DONE] 07:19:30 KST` (총 **37분 47초**)
- F2 필터 + 엔티티 중복 제거 첫 실측

### timings (Run 6 대비)

| 단계 | Run 6 | Run 7 | Δ |
|---|---|---|---|
| Encoding | 5,244.51s (87.4m) | **2,035.94s (33.9m)** | ↓61% |
| UMAP | 341.12s | **115.6s** | ↓66% |
| HDBSCAN | 28.98s | **9.44s** | ↓67% |
| save_clusters | ~108s | ~82s | |
| **총** | **96분** | **38분** | ↓60% |

필터로 입력이 200k→92,798로 축소되어 모든 단계가 자연 가속.

### 정량 지표

| 지표 | Run 6 | **Run 7** | 판정 기준 | 상태 |
|---|---|---|---|---|
| article_count (노이즈 제외) | 107,634 | 57,289 | — | |
| cluster_count | 1,459 | 817 | — | |
| noise_count | 92,290 | 35,509 | — | |
| **noise_ratio** | 46.16% | **38.26%** | >40% 실패 | ✅ 정상 (한계 바로 아래) |
| **max_size** | 2,160 | **1,602** | >1500 실패 | ❌ 실패 (경계선) |
| mega (≥1000) | 5 | 2 (280138, 280139) | ≥1 경고 | ⚠ |
| 모집단 (HDBSCAN 입력) | 199,924 | 92,798 | <80k 필터 과잉 | ✅ (80k 이상) |

### 정성 (top10 + top1 샘플 + top3 entity overlap)

**top10 주제 요약**:
| id | size | 대표 주제 | 일관성 |
|---|---|---|---|
| 280138 | 1,602 | **UK/IE 지역 단신 catch-all** (샘플 20건: Bradford 베이커리, Oxford Mini, Swindon, Waterford, Herefordshire 등) | ❌ 주제 제각각 |
| 280139 | 1,421 | Indonesia 가사노동 합법화 (22년 투쟁) | ✅ 단일 |
| 280140 | 1,265 | 경희대 2026 스프링 하모니 | ✅ 단일 |
| 280141 | 951 | 북한 해커 crypto $300M 탈취 (글로벌 뉴스) | ✅ 단일 |
| 280142 | 838 | 2011 일본 쓰나미 진흙 해안 연구 | ✅ 단일 |
| 280143 | 661 | 캐나다 우주로켓 상업 발사 허가 | ✅ 단일 |
| 280144 | 645 | Eastpointe man murder plea | ✅ 단일 |
| 280145 | 590 | Rachel Roddy 아몬드 레시피 | ✅ 단일(국지) |
| 280146 | 581 | Oberursel 20년대 항공사진 | ✅ 단일(국지) |
| 280147 | 548 | Nigeria EndSars 경찰 억울수감 | ✅ 단일 |

**persons_per_article** (단일 주제 일관성 지표):
- 280138 (top1 catch-all): **2.09** (Run 6 유사 mixed bag 수준)
- 280139: **0.28** (단일 주제 강력 신호)
- 280140: **0.17** (단일 주제 강력 신호)

### 판정: **부분 성공 (경계선 실패)**

**근거**:
1. 사전 명시 실패 기준 `max_size > 1500` → **1,602로 초과** (정량 실패)
2. 그러나 top2~10은 **주제 일관성 대폭 확보** (persons_per_article 0.17~0.28). Run 6의 "전 cluster mixed bag"에서 "top1만 catch-all" 분화됨
3. top1(280138) 1,602건은 **UK/IE 지역 단신**이 임베딩 공간에서 한 덩어리로 뭉침. 필터·중복 제거가 닿지 못하는 잔여 실패 모드
4. 실행 시간 ↓60%, noise ↓17%, max_size ↓26%로 Run 6 대비 전반 개선

### 해석 요약

F2 설계(품질 필터 + 엔티티 중복 제거)는 **방향은 옳았으나 단독으로는 불충분**. top2~10은 실질 클러스터링 성공, top1만 지역 단신 catch-all 잔존.

### 다음 단계 후보 (세열 결정 영역)

A) **min_cluster_size 상향 (20→40)**: 1,600건 catch-all을 더 분해 가능성. 단 단일 변인 원칙 준수
B) **cluster_selection_method 'eom'→'leaf'**: 더 세분화, max_size 축소 기대
C) **지역 매체 필터 강화**: source_host tier 기반 블랙리스트 확장 (tier 7/8 일부 제거 또는 authority ≤0.3 제외)
D) **UMAP n_neighbors 축소 (30→15)**: 국소 구조 강조, 대형 catch-all 약화
E) **위 전부 거부 + D 경로 재검토**: TF-IDF+임베딩 혼합 등 아키텍처 재설계

파라미터 자동 조정 금지 원칙에 따라 현 상태에서 중단. 결정 대기.

### 관련 상태

- cluster_runs row id=7 저장 완료
- clusters 817개 저장 (noise 35,509건은 articles.cluster_id=NULL 유지)
- /digest는 이제 run_id=7 데이터 조회 가능 (품질은 위 판정대로)

---

## F 경로: Phase 2 재활용 (2026-04-24)

### 착수 근거

Run 7에서 top2~10 주제 일관성 확보 (persons_per_article 0.17~0.28), 단 top1 1,602건(UK/IE 지역 단신 catch-all)만 max_size 실패선 초과. **top1 1개 catch-all 해소**가 목표.

### 설계 선택

- **옵션 F 채택**: top1만 2차 클러스터링 (전체 재편 금지)
- Phase 2 조사 결과에서 **b) 부분 수정 후 재활용** 경로 확정
  - `_split_large_recursive` 자체는 **수정 없이 재활용** (TF-IDF 경로 보호)
  - `cluster_phase1_embedding` 반환에 `reduced` (UMAP 50d) 추가
  - `run_once` embedding 분기에 Phase 2 호출부 신설

### 확정 파라미터

| 항목 | 값 | 근거 |
|---|---|---|
| `PHASE2_SIZE_THRESHOLD` | **1,500** | Run 7 실패선(>1500) 일치. 환경변수 override 가능 |
| 입력 공간 | **bge-m3 UMAP 50d dense** | TF-IDF sparse 시절과 다름. sklearn MiniBatchKMeans가 둘 다 수용 |
| 알고리즘 | `MiniBatchKMeans` 재귀 분할 (기존 유지) | `sub_k=min(50, max(2, sqrt(n)))`, `batch_size=1024`, `n_init=3`, `random_state=42` |
| `max_depth` | 4 (기존) | |
| 노이즈(-1) 처리 | **제외** (Phase 2 대상 아님) | |
| 결과 병합 | **완전 교체** (첫 그룹 기존 cid, 나머지 새 cid) | 부모-자식 관계 없음 |
| 활성 스위치 | `NEWS_INTEL_ENABLE_PHASE2=true` | 기본 False 유지. 명시적 환경변수로만 활성 |

### 코드 변경

| 파일 | 변경 |
|---|---|
| `data/clusterer_embedding.py` | 반환 dict에 `"reduced"` (UMAP 50d numpy array) 추가 |
| `data/clusterer.py` | `ENABLE_PHASE2` 환경변수 전환, `PHASE2_SIZE_THRESHOLD` 상수 신설, embedding 분기에 Phase 2 블록 삽입 |
| `_split_large_recursive` | **수정 없음** (TF-IDF 경로 보호) |

### 실패 모드 사전 명시

| Run 8 결과 | 판정 | 다음 조치 |
|---|---|---|
| max_size > 1500 (효과 없음) | **실효 없음** | UMAP 공간에서 KMeans가 분할 못함. 재설계 필요 |
| max_size 500~1500 | **부분 성공** | threshold 추가 조정 여지 |
| max_size < 500 AND top2~10 유지 (persons_per_article<0.5 유지) | **성공** | main 머지 검토 |
| top2~10 분해 (persons_per_article 상승) | **과잉 분할** | size_threshold 상향 또는 sub_k 공식 조정 |
| cluster_count 급증 (예: 2000+) | **과분할 의심** | max_depth 축소 또는 size_threshold 상향 |

### Run 7 결과 보존

- run_id=7 cluster_runs row 유지 (삭제 없음)
- Run 8은 새 run_id=8 부여 예정. clusters 테이블은 `save_clusters`가 매 run에서 DELETE+INSERT하므로 Run 7 cluster 데이터는 **clusters 테이블엔 남지 않음** (cluster_runs 통계만 보존)
- /digest는 Run 8 완료 후 자동으로 run_id=8 데이터 조회 전환

### 승인 대기

- Run 8 실행: 세열 별도 승인 필요
- 예상 실행 시간: Run 7 38분과 유사 (Phase 2는 top1 1,602건 × O(k×iter) 미미)
- 실행 명령 예시:
  ```
  NEWS_INTEL_ENABLE_PHASE2=true ~/venv/news_intel/bin/python -u -c "from data import clusterer; print(clusterer.run_once())"
  ```

---

## Run 8 결과 (2026-04-24)

### 실행 환경
- 브랜치: `feat/f1-bge-m3-hdbscan` HEAD `d16698b`
- PID 99365, `[START] 08:04:15 KST`, `[DONE] 08:41:17 KST` (총 **37분 2초**)
- 활성 옵션: `NEWS_INTEL_ENABLE_PHASE2=true`, `PHASE2_SIZE_THRESHOLD=1500`

### 실측 timings

| 단계 | Run 7 | Run 8 |
|---|---|---|
| Encoding | 2,035.94s | **2,021.88s** (≈동일) |
| UMAP | 115.6s | **116.23s** (≈동일) |
| HDBSCAN | 9.44s | **10.32s** (≈동일) |
| Phase 2 | — | **< 0.1s (targets=0 스킵)** |
| save_clusters | ~82s | ~50s |
| **총** | 38분 | **37분** |

### 정량 지표 (Run 7 대비)

| 지표 | Run 7 | **Run 8** | 판정 기준 | 상태 |
|---|---|---|---|---|
| cluster_count | 817 | **833** | — | +16 (Phase 2 아님, UMAP 변동 기인) |
| article_count | 57,289 | 57,752 | — | |
| noise_ratio | 38.26% | 37.77% | >40% 실패 | ✅ |
| **max_size** | 1,602 | **1,424** | >1500 실패 | ✅ (실패선 아래) |
| mega(≥1000) | 2 | **5** | ≥1 경고 | ⚠ 오히려 증가 |
| 모집단 | 92,798 | 92,798 | ≥80k | ✅ (F2 필터 재현성) |

### Phase 2 통계 (중요: **작동 안 함**)

```
targets_count: 0
splits_attempted: 0
splits_skewed: 0
max_depth_reached: 0
cluster_count_after: 833 (= before)
max_size_after: 1424 (= before)
```

**원인**: HDBSCAN 단계에서 이미 max=1,424로 내려가 size_threshold=1,500 미달. Phase 2 진입 직전 필터에서 대상 cluster 0개로 판정 → 건너뜀.

**Run 7→Run 8 max_size 감소(1,602→1,424)는 Phase 2 덕분이 아니라 UMAP 비결정적 변동**:
- UMAP `random_state=42` 고정에도 `low_memory=True`+pynndescent 근사 kNN의 비결정성 존재
- 동일 파라미터, 동일 입력(F2 필터 92,798 재현)에도 출력 미세 차이 → HDBSCAN cluster 경계 달라짐

### 정성 관찰 (catch-all 지역 이동)

| id | size | 대표 주제 | 일관성 | persons/art |
|---|---|---|---|---|
| 280955 | 1,424 | **Indonesia 지역 단신 catch-all** (kompas/liputan6/tribunnews/viva/bisnis. 경찰·부패·스포츠·교통·환경·차량 파편) | ❌ 실패 | 0.28* |
| 280956 | 1,387 | **UK "Country diary" catch-all 재등장** (Run 7 top1) | ❌ 실패 | **2.04** |
| 280957 | 1,181 | 지구의 날 대만 고양이 보호 | ✅ | 0.45 |
| 280958 | 1,178 | Vingroup Taxi IPO (베트남) | ✅ | — |
| 280959 | 1,092 | 북한 crypto 해킹 $300M (글로벌) | ✅ | — |
| 280960 | 961 | 2011 일본 쓰나미 진흙 연구 | ✅ | — |
| 280961 | 839 | Rachel Roddy 아몬드 레시피 | ✅ | — |
| 280962 | 730 | Spain 이민 규제 | ✅ | — |
| 280963 | 550 | Nigeria EndSars 경찰 | ✅ | — |
| 280964 | 511 | Dingdong Cayman Ltd | ✅ | — |

\*280955 persons_per_article=0.28은 낮지만 실제 샘플은 인니 국내 뉴스 catch-all. GKG가 인니 기사에 persons 적게 부여하는 구조적 특성 영향.

**결정적 관찰**: Run 7의 top1 "UK/IE 지역 catch-all"이 Run 8에서 **top2로 밀려나고 새 top1에 "Indonesia 지역 catch-all" 등장**. 지역 단신 catch-all 문제의 본질은 해결되지 않고 위치만 이동.

### 판정: **부분 성공 (정량) / 실질 실패 (정성)**

- 정량 판정 기준(max_size 500~1500): **부분 성공**
- 정성 관찰: Phase 2 미작동 + catch-all 지역 이동 = **실질적 미해결**
- Phase 2 배선 자체는 OK (구현 검증 완료), 단 임계 1500이 이번 분포에 맞지 않음

### 다음 단계 후보 (세열 결정)

A) **`PHASE2_SIZE_THRESHOLD` 하향 (1500→1200 또는 1000)**: Run 8의 top1(1,424), top2(1,387) 둘 다 Phase 2 대상에 포함. 단일 변인 실험.
B) **Phase 2 임계 유지, 지역 매체 블랙리스트 강화**: source_host tier 기반 필터. authority<0.3 제외 등. F2 필터에 F4 추가.
C) **재실행 여러 번 평균화**: UMAP 비결정성 관찰. `random_state` 외 다른 재현성 보장 방법 필요.
D) **D 경로 재검토**: TF-IDF+임베딩 하이브리드, 또는 다른 임베딩 모델(multilingual-e5).
E) **Phase 2 알고리즘 재설계**: KMeans 대신 HDBSCAN recursive (embedding 공간에 더 적합).

파라미터 자동 조정 금지 원칙 준수, 중단 후 결정 대기.

### 관련 상태

- cluster_runs id=8 저장, Run 7 row 유지
- clusters 테이블은 Run 8 결과로 교체 (save_clusters DELETE+INSERT)
- /digest는 run_id=8 데이터 조회 전환

---

## Run 9 결과 (C 경로 — UMAP 비결정성 진단)

### 실행 환경
- 브랜치: `feat/f1-bge-m3-hdbscan` HEAD `dc74fdc`
- PID 1618, `[START] 2026-04-24 09:21:56 KST`, `[DONE] ~09:59 KST` (총 **약 37분**)
- 활성 옵션: **Run 8과 완전 동일** — `NEWS_INTEL_ENABLE_PHASE2=true`, `PHASE2_SIZE_THRESHOLD=1500`
- 목적: Run 7/8 max_size 차이(1,602→1,424, Δ178)가 UMAP `random_state=42` 고정에도 발생하는 비결정성인지 확인

### 실측 timings

| 단계 | Run 7 | Run 8 | **Run 9** | 변동 범위 |
|---|---|---|---|---|
| Encoding | 2,035.94s | 2,021.88s | **2,032.43s** | ±0.7% |
| UMAP | 115.6s | 116.23s | **117.5s** | ±0.8% |
| HDBSCAN | 9.44s | 10.32s | **8.84s** | ±8% |
| save_clusters | ~82s | ~50s | ~57s | |
| **총** | 38분 | 37분 | **~37분** | |

timings는 사실상 동일. 입력(F2 필터 후 92,798)도 재현.

### 3-way 지표 비교

| 지표 | Run 7 | Run 8 | **Run 9** | min-max range | 판정 |
|---|---|---|---|---|---|
| article_count | 57,289 | 57,752 | 56,202 | 1,550 | 소폭 변동 |
| cluster_count | 817 | 833 | **839** | 22 | 소폭 변동 |
| noise_count | 35,509 | 35,046 | 36,596 | 1,550 | 소폭 변동 |
| noise_ratio | 38.26% | 37.77% | **39.44%** | 1.67%p | ✅ <40% 유지 |
| **max_size** | **1,602** | **1,424** | **1,416** | **186** | ⚠ 중간 영역 |
| multilang | 353 | 377 | 363 | 24 | 소폭 변동 |
| large(≥10) | 817 | 833 | 839 | 22 | 소폭 변동 |

**max_size 편차 186**: 지시서 진단 기준의 **중간 영역** (<100=비결정성 작음, >200=구조적 반복).

### Phase 2 통계

Run 9 max_size=1,416 < PHASE2_SIZE_THRESHOLD=1,500 → Run 8과 동일하게 **Phase 2 targets=0 스킵**. Phase 2는 이번 분포에서 3회 연속 미작동.

### top10 정성 (Run 9)

| rank | id | size | 대표 주제 | 일관성 |
|---|---|---|---|---|
| 1 | 281788 | **1,416** | **UK/IE "Country diary" 지역 단신 catch-all** (independent.ie, theargus.co.uk, swindonadvertiser.co.uk, irishtimes.com, dorsetecho.co.uk, lancashiretelegraph.co.uk …) | ❌ mixed bag |
| 2 | 281789 | 1,227 | 경희대 2026 스프링 하모니 | ✅ 단일 |
| 3 | 281790 | 1,046 | 북한 해커 crypto $300M (글로벌) | ✅ 단일 |
| 4 | 281791 | 1,022 | 2011 일본 쓰나미 진흙 해안 연구 | ✅ 단일 |
| 5 | 281792 | **893** | **Indonesia 가사노동 합법화** (Run 8의 top1 catch-all이 top5로 이동) | ✅ 단일 주제지만 사이즈↓ |
| 6 | 281793 | 720 | Rachel Roddy 아몬드 레시피 | ✅ 단일 |
| 7 | 281794 | 615 | Spain 이민 규제 | ✅ 단일 |
| 8 | 281795 | 526 | 터키 학교 상담 시스템 | ✅ 단일 |
| 9 | 281796 | 516 | Steam/Epic 무료 게임 | ✅ 단일 |
| 10 | 281797 | 504 | Nigeria EndSars | ✅ 단일 |

### top3 entity overlap

| id | size | uniq_persons | uniq_orgs | **persons/article** |
|---|---|---|---|---|
| 281788 | 1,416 | 4,066 | 3,348 | **2.871** |
| 281789 | 1,227 | 474 | 6,559 | 0.386 |
| 281790 | 1,046 | 755 | 4,687 | 0.722 |

281788의 persons/article 2.87은 Run 6 mixed bag 수준. Run 7 top1(2.09)보다 **더 심한 분산** — UK/IE 지역 단신이 실제 여러 주제의 집합임을 재확인.

### top1 catch-all 정체 — 3회 비교

| Run | top1 catch-all | size | top2~5 catch-all 잔존 여부 |
|---|---|---|---|
| 7 | **UK/IE "Country diary"** | 1,602 | Indonesia는 별도 top 미등장 |
| 8 | **Indonesia 지역 단신** | 1,424 | UK "Country diary" 1,387 (top2) |
| 9 | **UK/IE "Country diary"** | 1,416 | Indonesia 893 (top5) |

**결정적 관찰**: UK/IE "Country diary"와 Indonesia 지역 단신 두 catch-all이 **매 run 반복 등장**하며, UMAP 비결정성에 따라 **상대 크기만 swap**. 단순 "무작위 지역이 top1에 오른다"가 아니라 **특정 2대 구조가 안정적으로 형성**되는 패턴.

### 편차 해석 → 진단 결론

지시서 매핑:
- max_size 편차 186 → **중간 영역**
- catch-all 정체 "매번 다른 지역"인가? → **부분적으로만 그러함** (UK↔Indonesia swap, 완전 새 지역 아님)

**진단**: **UMAP 비결정성은 작음~중간 (top1/top2 rank swap에 영향), 그러나 catch-all 구조 자체는 구조적 반복**.

- UMAP `random_state=42` + `low_memory=True` + pynndescent 조합이 완전 결정적이지 않음 (실제 편차 관찰됨)
- 단 편차의 크기는 "rank swap" 수준이지 "구조 붕괴" 수준 아님
- 핵심 문제는 임베딩 공간 자체에서 UK 지역 신문 피드와 인도네시아 지역 매체 피드가 각각 거대한 근접 덩어리를 형성한다는 **데이터·임베딩 구조 원인**

### 다음 단계 후보 (세열 결정)

| 후보 | 설명 | 기대 | 리스크 |
|---|---|---|---|
| **B (권장)** | **source tier 필터 (F4 레이어)** — 기생 피드성 `.co.uk` 지역 신문 군집 / 인도네시아 지역 매체 authority 하향 또는 제외 | 두 catch-all 입구 차단, 본질 해결 | 일반 사용자 관심 소재도 과도하게 배제 위험 |
| A | `PHASE2_SIZE_THRESHOLD` 1,500→1,200 | UK·Indonesia catch-all 모두 Phase 2 대상 진입, 기계적 분할 | 사후 처리라 근본 해결 아님 |
| A+B 병행 | 입구(B)+출구(A) 동시 | 가장 효과 클 가능성 | 단일 변인 원칙 위배 |
| D | 아키텍처 재설계 (multilingual-e5, 하이브리드) | 임베딩 공간 자체 개선 | 비용 큼 |
| G | Run 9 수용 + main 머지 | /digest 운영 현실화 | top1 catch-all 품질 낮음 |

**구조적 권고**: C 진단은 "UMAP 비결정성"이 원인의 주된 부분이 아님을 확증. **다음은 B 경로 우선 검토** (원인 지향).

### 금지 준수

- main 머지 없음
- orchestrator 재개 없음
- 파라미터 자동 조정 없음 (Phase 2 threshold 변경 없음)
- SESSION_HANDOFF 수정 없음

### 관련 상태

- cluster_runs id=9 저장, Run 7/8 row 유지
- clusters 테이블은 Run 9 결과로 교체 (DELETE+INSERT)
- /digest는 run_id=9 데이터 조회 전환 (UK catch-all이 다시 top1 위치)

---

## 정정 기록 (2026-04-24, Step B-0 발견)

### top2의 실제 정체

Run 9 top2 (cluster 281789, 1,227건):
- `canonical_title`: "경희대 2026 스프링 하모니" (한국어 라벨 함정)
- 실제 구성: **베트남 국내 뉴스 catch-all**
  (baomoi.com 228건 + .vn 매체 15+ 도메인 합 ~780건)
- 샘플 주제: V-League 축구, Sacombank 주총, 고속도로 공사, 부동산 사기, 솔라·풍력 손실 등 베트남 국내 잡다 단신

### Run 9 실제 catch-all은 3종

| rank | cluster | size | 실제 정체 |
|---|---|---|---|
| 1 | 281788 | 1,416 | UK/IE "Country diary" 지역 단신 |
| 2 | 281789 | 1,227 | **베트남 국내 단신** (한국어 canonical_title에 가려짐) |
| 5 | 281792 | 893 | Indonesia 지역 단신 |

### Run 7/8 판정의 함정 가능성

"Run 7 top2~10 주제 일관성 확보 (persons_per_article 0.17~0.28)" 판정은 canonical_title만 보고 내려진 것. 실제로는 베트남/인도네시아/영어 지역 단신 catch-all이 이미 있었을 가능성.

persons_per_article 낮은 값은 "주제 일관"이 아니라 해당 언어권 기사에 인물 엔티티가 원래 적을 뿐이었을 가능성 확인됨.

### 추가 구조적 문제

GKG language 오태깅 확인:
- baomoi.com 베트남 기사가 `language='en'`으로 태그됨
- F2 필터 "NULL language 제외"를 통과해도 언어 태그 틀린 기사가 엉뚱한 임베딩 공간에 진입. 별도 구조적 문제로 인식 (F4 범위 밖).

### persons_per_article 지표 재정의 (2026-04-24)

**기존 사용**:
- 해당 cluster의 주제 일관성 판정 지표
- 낮으면 "주제 일관", 높으면 "mixed bag"

**문제점**:
- 언어권별 entity 풍부도 차이 무시 (베트남 기사 본래 persons 엔티티 적음)
- canonical_title과 무관한 raw count 기반이라 catch-all 발견 실패
- 지표 단독으로 성공 판정 근거로 사용 불가

**보완**:
- 주제 일관성 판정은 canonical_title + 실제 기사 title 샘플링 병행 필수
- persons_per_article은 참고 지표로만 사용, 단독 판정 근거 금지
- 언어별 정규화(해당 언어권 평균 대비 상대값) 추후 도입 고려

---

## B 경로: F4 source tier 필터 (2026-04-24)

### 착수 근거

Run 9 Step B-0 진단으로 catch-all 3종 구조 확정:
- UK/IE 지역 체인 (~30 도메인, independent.ie / theargus.co.uk / swindonadvertiser.co.uk / irishtimes.com …)
- Vietnam 매체 (baomoi.com + `.vn` 15+ 도메인: danviet.vn / vietnamnet.vn / stockbiz.vn / vov.vn / vietnamnews.vn …)
- Indonesia 지역 (kompas.com / tribunnews.com / antaranews.com)

추가로 한국 경제·정치 매체군 (heraldcorp.com / fnnews.com / edaily.co.kr / koreaherald.com / ohmynews.com / metroseoul.co.kr) = top3 cluster 281790(1,046)에 집중. 일반 뉴스 가치는 있으나 catch-all 기여.

+ A 범주 공시/자동 피드 매체 2개:
- `cfi.net.cn` (중국 상장기업 공시 자동 피드, 제목 패턴 "종목코드(XXXXXX):공시제목-CFi.CN 中财网")
- `dailypolitical.com` (미국 주식 target price 자동 피드, "Ticker (EXCHANGE:SYMBOL) Price Target Raised to $X")

### 설계 선택: F4c (A+B 조합)

- **블랙리스트 (A)**: `cfi.net.cn`, `dailypolitical.com` — 자동 피드성 반복 포맷, 뉴스 가치 없음
- **Per-cluster source cap (B)**: `<세열 확정값>` (B-1-3 측정 분위수 기반)
- **적용 시점**: post-clustering (HDBSCAN + Phase 2 후). 임베딩 공간엔 영향 없음, 최종 labels 단계에서만 정리.
- **초과분 처리**: noise(-1)로 재할당 (articles.cluster_id=NULL 저장)
- **결정적 처리**: source별 cap 초과분은 `article.id` 오름차순으로 유지, 큰 id 쪽부터 drop → 재실행 시 동일 결과

### 환경변수

- `NEWS_INTEL_F4_SOURCE_CAP=<N>` — 정수. 미설정 시 F4 비활성.
- `NEWS_INTEL_F4_BLACKLIST="host1,host2"` — 쉼표 구분. 공백 strip, 빈 문자열 방어. 미설정 시 기본값 `cfi.net.cn,dailypolitical.com`.
- `NEWS_INTEL_ENABLE_PHASE2`와 독립 (둘 다 on 가능).

### 실패 모드 사전 명시

| Run 10 결과 | 판정 | 다음 조치 |
|---|---|---|
| max_size > 1,000 여전 | **cap 미효과** | 임계 하향 검토 (N-5) 또는 block 단위 확장 |
| noise_ratio > 50% | **cap 과강** | 임계 상향 (N+5) 또는 적용 범위 축소 |
| 모집단 < 80,000 | **필터 과잉** | cap 상향 또는 블랙리스트 축소 |
| catch-all 정체만 교체 (예: UK→독일/일본) | **블랙리스트 확장 또는 cap 근본 재설계** 필요 | Step B-0 재조사 |
| max_size 500~1,000, top2~10 주제 일관 | **성공 후보** | 정성 검증(title 샘플링 병행) 후 main 머지 검토 |

### 측정 근거 기록

Task B-1-3 SQL 결과 (Run 9 정상 cluster size 20~500 대상, 30,836 rows):

| 분위수 | 값 |
|---|---|
| p50 | 1 |
| p75 | 1 |
| p90 | 2 |
| p95 | 3 |
| p99 | **11** (cap 임계 결정 근거 — "정상 분포 99% 범위" 경계) |
| max | 338 (size 500 근접 cluster의 이상치) |

Big cluster(>500) 내 단일 source 점유율:
- top1 UK(1,416): 최대 `independent.ie` 71 (5.0%) — 분산형
- top2 VN(1,227): **`baomoi.com` 228 (18.6%)** — 집중형, p99의 20배
- top3 KR(1,046): `heraldcorp.com` 113 (10.8%) — 집중형 (신규 발견)

### 세열 확정값 (2026-04-24)

- **cap 값: 10** (p99=11 기반 공격적 시작 → Run 10 실측 후 완화/강화 재검토)
- **top3 KR 처리: A (cap만 적용)** — heraldcorp 등 화이트리스트 예외 없음. 일관성 우선, Run 10 실증 후 예외 재검토
- **적용 시점: post-clustering (사후)**
- **블랙리스트: `cfi.net.cn`, `dailypolitical.com` 적용**

### 실행 환경변수 (Run 10 용)

```
NEWS_INTEL_ENABLE_PHASE2=true
NEWS_INTEL_F4_SOURCE_CAP=10
NEWS_INTEL_F4_BLACKLIST="cfi.net.cn,dailypolitical.com"
```

cap 후보별 산술 상한 (실제 HDBSCAN 재배치 전 기계적 상한):

| cluster | 현재 | cap=10 | cap=15 | cap=20 |
|---|---|---|---|---|
| top1 UK | 1,416 | ~1,086 | ~1,176 | ~1,276 |
| top2 VN | 1,227 | ~707 | ~755 | ~837 |
| top3 KR | 1,046 | ~763 | ~784 | ~836 |

### 범위 밖 인지 사항

- **GKG language 오태깅** (baomoi.com VN 기사 `language='en'` 태그) → 별도 과제. F2 필터의 "NULL language 제외" 통과시키지만 실제 언어 다름. 임베딩 공간 오배치 원인 중 하나로 의심. 별도 설계 필요.
- **자동 탐지 룰**: `cfi.net.cn` 같은 공시 포맷 제목 정규식 탐지 → 블랙리스트 확장 시 검토. 현재는 수동 entry만.
- **한국 일반 경제매체 화이트리스트 예외 처리 여부**: top3 KR cluster의 heraldcorp/fnnews/edaily는 일반 뉴스라 cap 적용 시 정당한 경제 이슈도 약화 우려. 세열 확정값과 별도로 결정 대기.
- **사전 필터(임베딩 전 제외) vs 사후 필터(cluster 배정 후 noise)**: 본 설계는 **사후**. 사전 적용 시 임베딩 공간 자체 재구성 효과까지 기대 가능하나 재학습 비용 큼. 추후 변경 여지 있음.

### 구현 상태

- **코드: 구현 완료** (2026-04-24, B-1-4)
  - 파일: `data/clusterer.py`
  - 신규 상수: `F4_SOURCE_CAP`, `F4_BLACKLIST`, `F4_ENABLED` (line 53~66)
  - 신규 함수: `_apply_f4_source_cap(articles_aligned, labels, per_source_cap, blacklist) -> (new_labels, stats)` (line 90~)
  - 호출 위치: `run_once()` embedding 분기 내 Phase 2 블록 직후, `valid_indices` 추출 전 (line 808 직후)
  - 통계 기록: `system_log.details`에 `phase="f4"` 로 `log_to_db` (Phase 1·2 동일 패턴 — `cluster_runs`에 `details` 컬럼 없어 기존 관례 계승)
- **Phase 1/2 코드 불변**: `cluster_phase1_embedding`, `_split_large_recursive` 본체 수정 없음
- **TF-IDF 경로 영향 없음**: F4는 `CLUSTERING_BACKEND == "embedding"` 분기 내부에서만 호출
- **결정성 보장**: `article.id` 오름차순 정렬 후 cap 초과분 drop
- **단위 테스트 통과 (2026-04-24)**: 결정성 / 블랙리스트 / cap 경계 / 원본 labels 불변 4항목 확인

---

## Run 10 결과 (F4 첫 실전, 2026-04-24)

### 실행 환경

- 브랜치: `feat/f1-bge-m3-hdbscan` HEAD `54dfe4f`
- PID 4015, `[START] 10:54:48 KST`, `[DONE] ~11:32 KST` (총 **~37분**, Run 7/8/9 평균과 동일)
- 활성 옵션: `NEWS_INTEL_ENABLE_PHASE2=true`, `NEWS_INTEL_F4_SOURCE_CAP=10`, `NEWS_INTEL_F4_BLACKLIST="cfi.net.cn,dailypolitical.com"`

### timings

| 단계 | 소요 | Run 9 대비 |
|---|---|---|
| Encoding | 2,034.48s (33.9m) | ≈ (Run 9 2,032.43s) |
| UMAP | 114.83s | ≈ |
| HDBSCAN | 9.19s | ≈ |
| Phase 2 | < 0.1s (targets=0, 임계 1,500 미달) | 3회 연속 미작동 |
| F4 | 즉시 (Python set/dict 연산) | 신규 |
| save_clusters | ~60s | ≈ |
| **총** | **~37분** | 동일 |

### 4-way 지표 비교

| 지표 | Run 7 | Run 8 | Run 9 | **Run 10** | Δ vs Run 9 |
|---|---|---|---|---|---|
| article_count (배정) | 57,289 | 57,752 | 56,202 | **47,118** | **-9,084** (F4 drop 효과) |
| cluster_count | 817 | 833 | 839 | **826** | -13 |
| noise_count | 35,509 | 35,046 | 36,596 | **37,251** | +655 |
| noise_ratio | 38.26% | 37.77% | 39.44% | **40.14%** | +0.70pp |
| **max_size** | **1,602** | **1,424** | **1,416** | **1,063** | **-353 (-25%)** |
| multilang | 353 | 377 | 363 | 367 | +4 |
| large(≥10) | 817 | 833 | 839 | 824 | -15 |

### F4 통계 (system_log.details, phase='f4')

| 필드 | 값 |
|---|---|
| per_source_cap | 10 |
| blacklist | `["cfi.net.cn", "dailypolitical.com"]` |
| blacklist_dropped | **662** |
| cap_dropped | **7,767** |
| clusters_affected | **183** |
| cluster_count_after | 826 |
| max_size_after | **1,063** |
| pre-F4 max_size (HDBSCAN 직후) | **1,489** |

**F4 단독 효과**: 1,489 → 1,063 (**-28%**). Phase 2는 targets=0으로 이번에도 미작동이므로 감소 전량이 F4 기여.

### SQL 3: top1 source 분포 (cap 작동 검증)

cluster **282627 (1,063, UK/IE "Country diary")**:
- **39개 `.co.uk`/`.ie` 도메인이 정확히 각 10건씩** (1개만 9건 = dailypost.co.uk)
- pct 모두 0.9% (1,063 중 10건)
- **cap=10 완벽 작동** — 원본 Run 9라면 independent.ie 71/theargus 45/swindonadvertiser 37 등 cap 없이 그대로였을 것

**근거**: cap이 기계적으로 작동했음에도 top1이 여전히 1,063건인 이유는 **도메인 수 자체가 많아** (40+ UK/IE 체인), 각 10건 × 40 = 400건 + 10건 미만 소규모 도메인들이 나머지를 구성.

### top10 정성 (canonical_title + 실제 정체)

| rank | id | size | canonical_title / 실제 정체 | 판정 |
|---|---|---|---|---|
| 1 | 282627 | **1,063** | UK/IE "Country diary" 지역 단신 (883건 .co.uk/.ie) | ❌ catch-all, 도메인 너무 많음 |
| 2 | 282628 | **649** | "Mud-rich 2011 Japan tsunami" canonical_title ← 함정. 실제 **일본 매체 catch-all** (mainichi/asahi/sankei/kyodo/sankei/tokyo-np 등 20+ `.jp` 도메인 + 중국 매체 10+). persons/subject 혼재 | ❌ 새 catch-all 발견 |
| 3 | 282629 | 636 | "경희대 스프링 하모니" canonical_title ← 여전 함정. **VN 추적 446건** = 베트남 catch-all 잔존 | ⚠ 축소됐으나 잔존 (Run 9 1,227→636, -48%) |
| 4 | 282630 | 617 | Spain 이민 규제 | ✅ 정당한 글로벌 뉴스 |
| 5 | 282631 | 535 | 북한 해커 crypto $300M — KR 매체 151건 참여, 글로벌 주제 | ✅ 정당 (KR catch-all 해소됨) |
| 6 | 282632 | 443 | **Oberursel 항공사진** — language_count=1 독일 지역 단신 catch-all 의심 | ⚠ 새 catch-all 가능성 |
| 7 | 282633 | 437 | Venezuelan sociologist 이민 | ✅ |
| 8 | 282634 | 340 | Rachel Roddy 아몬드 레시피 | ✅ |
| 9 | 282635 | 305 | Stranger Things 스핀오프 리뷰 | ✅ |
| 10 | 282636 | 290 | 트럼프-이란 휴전 (마케도니아어 canonical) | ✅ |

### Run 9 3종 catch-all 추적 (SQL 4)

| 지역 | Run 10에서 가장 몰린 cluster | size | 지역 기사 수 | Run 9 대비 판정 |
|---|---|---|---|---|
| **VN** (`.vn`/baomoi) | 282629 (top3) | 636 | 446 (70% 점유) | ✅ 축소 성공 (Run 9 top2 1,227 → Run 10 top3 636, -48%) |
| **UK/IE** (`.co.uk`/`.ie`) | 282627 (top1) | **1,063** | **883** (83% 점유) | ❌ 잔존 (Run 9 top1 1,416 → Run 10 top1 1,063, -25%. 여전히 top1) |
| **KR** (`.kr`/heraldcorp/newspim) | 282631 (top5) | 535 | 151 (28% 점유) | ✅ **해소** (Run 9 top3 1,046의 KR 주도 → Run 10에선 글로벌 crypto 뉴스의 보조 매체로 분산) |

### 판정: **부분 성공**

지시서 판정 기준 매핑:
- max_size 1,063 ∈ **1,000~1,500** → **부분 성공**
- noise_ratio 40.14% — "50% 과강" 미달, "35% 이하 성공" 초과 → 중간
- 모집단 47,118 (F2 후 92,798의 **51%** cluster 배정, F4 drop 9k 포함) — 80k 기준에서 cluster 배정률 기준 부족
- top3 내 단일 source 모두 ≤10 → **F4 cap 완벽 작동 확증**
- canonical_title 함정 재발: top2("일본 쓰나미" 라벨이 일본 매체 catch-all), top3("경희대" 라벨이 VN catch-all) — **라벨과 실제 정체 3축 검증 원칙 재확인**

### 성공·실패 상세

**성공한 부분** (F4 설계 의도대로 작동):
1. **VN 집중형 catch-all 축소**: baomoi.com 228 → cap 10 직격. Run 9 top2 1,227 → Run 10 top3 636 (-48%).
2. **KR catch-all 해소**: heraldcorp 113/fnnews 58/edaily 55 등 cap 적용 → 한국 매체가 tier 3 이하로 흩어짐. Run 9 top3 1,046 → Run 10 내 KR 주도 cluster 535 (그것도 글로벌 주제의 보조 매체로).
3. **블랙리스트 작동**: cfi.net.cn / dailypolitical.com 662건 완전 제거.
4. **cap 183 cluster 영향**: 대부분 정상 cluster는 p99=11 기준 cap 10 도달 전이라 무영향. 183개만 감소 → 정상 분포 대비 "상위 예외군" 타격 정확.

**미해결 / 새 문제**:
1. **UK 분산형 catch-all 여전 top1**: 지시서 사전 예측 실패 모드 #1 확증 — 40+ `.co.uk` 도메인 각 10건 × 40 = 400+이 cap으로 막히지 않음. **도메인 그룹화 필요**.
2. **새 catch-all 노출**: top2 (649) 일본 매체군 — Run 9에선 1,022건 "Mud-rich Japan tsunami" cluster였는데 cap이 잘라내고 나니 649로 남고, 이게 **실제론 일본 국내 잡다 뉴스의 집합**임이 드러남 (블랙리스트 2종엔 없었음).
3. **독일 Oberursel 443 (top6)**: language_count=1 단일 언어 지역 catch-all 패턴 의심. 작지만 유사 구조.

### 다음 단계 후보 (세열 결정, 자동 진행 금지)

| 옵션 | 설명 | 기대 | 리스크 |
|---|---|---|---|
| **B.1** | `.co.uk` 체인 **도메인 그룹화** (모든 `.co.uk`/`.ie` 도메인을 "uk_local_chain" 가상 source로 매핑해서 cap 10 적용) | UK catch-all 1,063 → 10 수준으로 직격 | BBC/Guardian 등 정당한 전국지도 .co.uk라 함께 타격. whitelist 필요 |
| **B.2** | 블랙리스트 확장: 일본 매체군 (mainichi.jp, asahi.com, sankei.com 등 20개) + 독일 oberursel | top2, top6 해소 | 무한 확장 리스크, 정당한 뉴스 손실 |
| **A.1** | `PHASE2_SIZE_THRESHOLD` 1,500 → **1,000**으로 하향 | top1 1,063 Phase 2 분할 대상 진입, KMeans 재귀 분할 작동 | 사후 처리라 근본 해결 아님, 기계 분할의 주제 일관성 보장 못함 |
| **B+A 병행** | B.1 (도메인 그룹화) + A.1 (Phase 2 threshold) | 입구+출구 동시 | 단일 변인 원칙 위배 |
| **F4 수용 + main 머지** | 현 상태가 Run 9보다 /digest 품질 개선 (max_size -25%, VN/KR 해소) | 운영 현실화 | UK catch-all 잔존 |
| **D** | 아키텍처 재설계 (multilingual-e5 등) | 임베딩 공간 자체 개선 | 비용 큼 |

**구조적 권고**: VN/KR은 F4로 충분히 해소됨을 확증. 남은 문제는 **UK 체인 분산형** 단 하나의 구조적 패턴 — **B.1 (도메인 그룹화)** 가 원인 지향. 단 whitelist 설계 필요.

### 금지 준수

- main 머지 없음 / orchestrator 재개 없음 / 파라미터 자동 조정 없음 / Run 11 자동 실행 없음 / cap 값 자동 조정 없음 / 블랙리스트 자동 확장 없음 / SESSION_HANDOFF 수정 없음

### 관련 상태

- cluster_runs id=10 저장
- clusters 테이블 Run 10 결과 (826 clusters)
- /digest는 run_id=10 데이터 조회 전환
- Run 7/8/9 cluster row는 이미 소실 (save_clusters DELETE+INSERT), 통계는 cluster_runs + system_log에 보존
