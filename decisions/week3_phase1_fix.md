# Week 3 T1 Phase 1 거대 클러스터 근본 수정

결정일: 2026-04-23
배경 이슈: Run 4 (04:05 UTC) importance 6236 이상, Day 6~7 운영 관찰

## 문제 진단 (Step 1-D)

run_id=3 top1 cluster(id=269156, 2211건) 샘플링 결과:
- 제목: You be the judge / Boys TV finale / 井手らっきょ / Bitcoin $79K / Nigeria 쿠데타 / Epstein / Schitt's Creek / Thunder / 유나이티드헬스그룹 혼재
- source_host 최상위 2.7% (와이어 복제 기각)
- 언어 en 85%, 9개 언어 혼합
- 35시간 지속, 하루 리듬 피크 (특정 이벤트 기각)

결론: Phase 1 KMeans가 의미가 아니라 "영어 뉴스체 공통 어휘 방향"으로 클러스터링.

## 가설 판정

| 가설 | 판정 |
|---|---|
| G1 와이어 복제 | 기각 |
| G2 k=√n 너무 작음 | 확정 (보조) |
| G3 MiniBatchKMeans batch 문제 | 보류 |
| G4 TF-IDF 공통 용어 지배 | 확정 (주범) |

## 수정 내용

| 파라미터 | 이전 | 이후 |
|---|---|---|
| TF-IDF max_df | 0.8 | 0.5 |
| TF-IDF min_df | 2 | 5 |
| TF-IDF sublinear_tf | True | True (유지) |
| Phase 1 k | √n | max(n/200, √n) |
| Phase 1 상한 | 5000 | 5000 (유지) |
| random_state | 42 | 42 (유지) |

## v3.5 §3 영향

- §3-D #23 변경 (max_df, min_df)
- §3-E #27-a 변경 (k 공식)
- 세열님 결정: SESSION_HANDOFF.md v3.6 리라이트는 추후. 본 문서가 결정 기록의 단일 진실.

## 수정 커밋

(커밋 해시는 commit 후 추가)

## 검증 결과

(수정 후 run_id=X 결과 첨부 예정)

---

## 검증 결과 (Step 1-E-verify, 2026-04-23)

### 정량 지표 (개선됨)

| 지표 | run_id=3 | run_id=4 | 변화 |
|---|---|---|---|
| max_size | 2,211 | 1,643 | ↓ 26% |
| mega(1000+) | 6 | 2 | ↓ 67% |
| huge(501~1000) | 6 | 0 | ↓ 100% |
| vocabulary_size | (미측정) | 20,000 | 목표 범위 내 |

### 의미 경계 (실패)

run_id=4 top1 cluster 274167 (1,643건):
- 제목 샘플: Law & Order SVU, Habitat Minden, Women's Lacrosse, IBM 키보드, Kevin Warsh Fed, Ethiopia Amnesty, Marvel Cinematic Universe, Haiti 1500 troops, Shedeur Sanders, Fastnet Film Festival
- top_entities persons: Bishop Amat, Sam Hanna, Otto Bock, J Sam Hanna Evens (무관)
- top_entities orgs: Nokia, IBM, Summit Hotel, Procter Gamble, Studio Ghibli (무관)
- source_host 최대 1.7%, en 85% (9개 언어)

run_id=4 top2 cluster 274168 (1,133건, canonical_title "Iranverhandlungen"):
- 실제 제목: 한국/중국 주식, 태국 국립공원, 헝가리 음식, 네덜란드 Tholen 등 (이란 협상 무관)
- canonical_title과 실제 내용 불일치 확인

### 결론

- E1+E2 수정은 크기 지표(max_size, mega 개수)를 개선했으나 의미 경계는 개선되지 않았음
- TF-IDF + KMeans의 근본 한계: 의미 표현 부재로 엔티티 간 의미 거리 구별 불가능
- 추가 파라미터 튜닝(max_df 0.3, k를 n/100)으로도 해결 어렵다고 판단 (증거: run_id=3과 run_id=4의 의미 경계 품질이 동일 패턴)

### 다음 결정

→ docs/decisions/week3_phase1_embedding.md 로 이어짐 (F1: 임베딩 도입 결정)
