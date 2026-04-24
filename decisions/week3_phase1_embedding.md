# F1: Phase 1 임베딩 기반 클러스터링 도입 결정

결정일: 2026-04-23
결정자: 세열님
선행 문서: docs/decisions/week3_phase1_fix.md

## 결정

v3.5 §3-E #32 ("임베딩 도입 금지 MVP~P4") 재검토. Phase 1을 TF-IDF + KMeans에서 로컬 임베딩 + 밀도/근접도 기반 클러스터링으로 교체.

## 근거

### 증거 1: TF-IDF + KMeans의 의미 경계 실패
- run_id=3 top1 (2211건): Labour Party + Heineken + Bitcoin + Epstein + Thunder 혼재
- run_id=4 top1 (1643건): Law & Order + IBM + Nokia + Marvel + Haiti 혼재
- 파라미터 튜닝(E1+E2)으로 크기는 개선, 의미는 동일 실패

### 증거 2: §3-E #32 "Mac M4 리소스" 우려의 실측 해소
- bge-m3-ko가 법률봇에서 정상 운영 중 (Mac M4 16GB 환경)
- 재사용 가능한 인프라 존재

### 증거 3: Run 4 이상의 근본 원인 확정
- Phase 1의 의미 경계 부재가 mega-cluster의 본질
- Phase 2 재귀 분할은 대증요법에 불과

## 미정 사항 (다음 세션에서 결정)

| 항목 | 옵션 |
|---|---|
| 임베딩 모델 | bge-m3-ko 재사용 / 다국어 전용 모델 / 영어+번역 이중 |
| 클러스터링 알고리즘 | HDBSCAN / FAISS kNN + Louvain / 임베딩 기반 KMeans |
| 마이그레이션 방식 | TF-IDF 제거 후 교체 / 병렬 운영 후 점진 전환 |
| 72h 윈도우 유지 여부 | 임베딩 메모리 비용 감당 가능한지 벤치마크 후 결정 |
| Phase 2 재귀 분할 유지 여부 | 알고리즘 선택에 따라 |
| 기존 cluster_runs 로깅 스키마 유지 여부 | 유지 가정 |

## v3.5 §3 영향

- §3-E #32: 임베딩 도입 금지 → 해제 (구체 구현 후 v3.6 명문화)
- §3-E #27-a: Phase 1/Phase 2 알고리즘 정의 변경 예정
- §3-D #22-23: TF-IDF 파라미터 항목 제거 또는 rewrite 가능성
- §1: 정체성 문구는 유지 (광범위 수집 인프라)

## 보류된 Week 3 §7 작업

F1 구현 기간 동안 다음 작업은 Week 4로 밀림:
- /geo /market 카테고리 명령어
- 한국 매체 /kr
- 메시지 포맷 개선
- launchd plist 설치
- 5일 연속 안정 운영 검증 (F1 구현 후 재개)

## 운영 중 상태

- orchestrator 중단 (PID stale, 09:30 UTC 이후 자동 run 없음)
- /digest는 run_id=4 데이터 계속 조회 중 (잡탕 클러스터 기반)
- F1 구현 완료까지 /digest 품질 저하 상태 유지 수용
- 세열님이 필요 시 수동 run 요청

---

## Appendix: Run 6 결과 + F1 중단 결정 (2026-04-23)

### 실행 환경
- 브랜치: `feat/f1-bge-m3-hdbscan`
- run_id: 6 (run_id=5는 OOM 크래시: MPS `Invalid buffer size: 114.85 GiB`, bge-m3 기본 max_seq_length=8192 원인. `model.max_seq_length=256` + char cap 2000 적용 후 재실행이 Run 6)
- 입력: 199,924 docs (빈 76건 제외, 원본 200,000)

### 실측 timings

| 단계 | 추정 | 실측 |
|---|---|---|
| Encoding (bge-m3 MPS batch=64) | 10~25분 | **87.4분** (4.26s/batch × 3,124 batches) |
| UMAP (1024→50) | 5~15분 | **5.7분** |
| HDBSCAN | 2~5분 | **0.5분** |
| save_clusters | 3~5분 | 1.8분 |
| **총** | 20~50분 | **96분 (1h 36m)** |

### 정량 결과 (3지표 전부 실패선 초과)

| 지표 | Run 4 (TF-IDF E1+E2) | Run 6 (F1) | 실패 기준 | 판정 |
|---|---|---|---|---|
| max_size | 1,643 | **2,160** | >1000 | ❌ |
| mega(≥1000) | 2 | **5** | ≥1 | ❌ |
| noise_ratio | N/A | **46.16%** | >40% | ❌ |
| cluster_count | 4,512 | 1,459 | — | — |
| singleton | 548 | 0 | — | 노이즈가 산발 뉴스 흡수 |

### 정성 관찰 (Run 6 top5 cluster)

| id | size | 대표 언어 | 대표 제목 |
|---|---|---|---|
| 278679 | 2,160 | **ru 집중** | ロシア共産党指導者… (러시아 국내 정치·경제·사고·교통 혼재) |
| 278680 | 1,941 | **ja 집중** | 牧野ＦＴＯＢ… 日本政府が外資ファンド… |
| 278681 | 1,892 | en | Rachel Roddy's recipe for almond and lemon spiced treacle tart |
| 278682 | 1,272 | **es 집중** | ¿De verdad hoy todo es neurodivergencia? |
| 278683 | 1,047 | en | Taxpayers' money given to help lonely veterans… |

top1 샘플 20건: 모두 러시아 국내 뉴스, 주제는 파편(법안·연금·날씨·교통·사건). Run 4의 "Labour Party + Heineken + Bitcoin + Epstein" 식 무관 혼재는 **사라짐**, 대신 **언어 단위 catch-all** 등장.

### 구조적 판정: F1 중단

- **실패 유형 전환**: TF-IDF의 "공통 뉴스체 어휘 방향 집계"는 해결됐으나, bge-m3 다국어 단일 벡터 공간이 **언어 매니폴드**를 의미 이슈보다 먼저 잡음. Run 4 혼재 문제가 Run 6 언어 catch-all 문제로 자리 바꿈.
- 파라미터 튜닝(min_cluster_size 20→15, eom→leaf, UMAP n_neighbors 30→15)으로는 구조적 한계 해결 불가. 언어 축이 UMAP 단계에서 이미 dominant.
- 임베딩 87분 소요 → 1h 주기 운영 부적합.

**결정**: F1 중단. 파라미터 재조정 경로 폐기. 아키텍처 재설계(F2)로 전환.

### F2로 이관되는 미해결 과제

1. 언어 catch-all 분해: 언어별 1차 분리 후 언어 내 cluster, 이후 cross-lingual 병합 2단계 구조 (→ `week3_phase1_f2_plan.md`)
2. 1h 주기 운영 가능한 임베딩 속도 (87분은 불가)
3. 단일 언어 내부 의미 혼재 잔존 (예: 영어 디저트 레시피 278681)
4. HDBSCAN noise 46% 해석: 진짜 산발 뉴스인지, min_cluster_size 부적절인지 분리 분석
5. MAX_ARTICLES=200,000 유지 여부 (465k → 200k 잘림)

---

## 정정 기록 (2026-04-24)

진단 재검증 (D1~D4 SQL) 결과, 기존 "언어 catch-all" 진단은 부분 오판으로 확인됨.

**실제 실패 원인 (증거 기반)**:

1. **데이터 품질 한계** — NULL language 15.5% (71,906건)의 persons 평균 길이가 NOT NULL 그룹의 39% 수준 (40 vs 103). 메타데이터 빈약.
2. **의미 신호 부족** — Run 6 top2 클러스터(1,941건)에 unique persons 6,088명 (기사당 3.14명). 주제 일관성 0, 완전 mixed bag.
3. **엔티티 mention 중복 미처리** — GKG는 mention 단위로 entity 기록. 실측 사례: "Alienware 13회, Nvidia 4회" 중복이 그대로 concat되어 임베딩에서 지배 신호화. TF-IDF는 `sublinear_tf`로 자연 방어, 임베딩은 방어 없음.
4. **입력 빈약 극단치** — p10=154자. 사례: `id=508657` 43자 `"Help! I'm in a Secret Relationship! Ed Lynn"` 유형이 mega-cluster에 흡수됨.

이전 "언어 catch-all" 관찰은 top1 샘플링 편향(러시아어 우세)에 의한 오진. 실제 top30은 대부분 다국어 mixed bag (D3 결과: top1 5언어, top2 10언어, top3 7언어).

→ F2 방향을 "언어별 2단계화"에서 **"데이터 품질 필터링 + 엔티티 중복 제거"**로 전환. 상세는 `week3_phase1_f2_plan.md` 재작성본 참조.
