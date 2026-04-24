<!--
════════════════════════════════════════════════════════════════════
SESSION_HANDOFF: news_intel 모듈
════════════════════════════════════════════════════════════════════
목적: Claude Code(로컬)가 작업 착수 전 반드시 처음부터 끝까지 읽는 문서

리라이트 이력:
  - 2026-04-20 v1.0 ~ 2026-04-22 v3.4: 이전 이력 생략 (각 버전 docs/ 보관)
  - 2026-04-23 v3.5: Week 2 P2 (클러스터링 + 스코어링 + 봇) 완료 시점 통합본
                     주요 변경:
                     - §1 정체성 강화: "공용 인프라" 본질 명시 (글로벌 광범위 수집,
                       카테고리 필터링은 상위 소비자 책임)
                     - §3-C: 미래 timestamp 자동 fallback 정책 추가
                     - §3-E #28: importance 공식 v2 (lang weight + persist cap)
                     - §3-E #31-a: 블랙리스트 0.01 유지 확정 (0.005 하향 불필요 결정)
                     - §3-J: 장애 복구 구현 완료 (SSD 마운트 + launchd)
                     - §6 스키마: clusters/daily_top_n 적용 완료
                     - §7: P2 완료 표시 + Week 3 (P3) 명세 신설
                     - §10: Week 2 운영 발견 사항 4개 기록
                     - §11: Week 2 P2 완료 기준 + 실측 결과
                     - §13: Week 3 검토 항목 정리
                     이것이 현재 유효한 최종 버전.

이 문서는 news_intel 공용 데이터 인프라와 그 첫 번째 소비자
geo-daily-digest-bot MVP 구축을 다룬다.
L3/Project A/Project B 스키마 계약은 이 문서 범위 외.
════════════════════════════════════════════════════════════════════
-->

# news_intel — 공용 뉴스 데이터 인프라

> **Claude Code 주의**: 이 문서는 작업 착수 전 반드시 처음부터 끝까지 읽을 것.
> 임의 판단으로 결정사항을 수정하지 말 것. 모든 확정사항(§3)은 세열님 최종 컨펌 완료.
> 변경 제안이 있으면 구현 전 세열님에게 보고.

---

## 0. MVP 용어 정의

| 용어 | 범위 | 완료 시점 |
|---|---|---|
| **데이터 레이어 MVP** | P1 (Week 1) — GDELT 이중 파이프라인 → articles 적재 | ✅ 2026-04-22 |
| **프로덕트 MVP** | P1 + P2 (Week 1~2) — 데이터 + 클러스터링 + `/digest` 봇 | ✅ 2026-04-23 |

단독 "MVP" = 프로덕트 MVP.

---

## 1. 정체성 (v3.5 강화)

**`news_intel`은 글로벌 이슈를 광범위하게 수집하는 공용 데이터 인프라이며, 카테고리별 필터링·취사선택은 상위 소비자 레이어의 책임이다.**

### 핵심 아키텍처 원칙 (v3.5 명시)

세열님 결정 (2026-04-23):
> "이 프로젝트의 근간은 글로벌 이슈. 너무 지엽적인 보도만 아니라면 어느 정도 여유 있게 수용. 각 상위 레이어에서 그 중에서 골라 쓰는 것을 고려."

**역할 분리**:

| 레이어 | 책임 | 필터 기준 |
|---|---|---|
| **news_intel (인프라)** | 글로벌 이슈 광범위 수집·클러스터링·다국어 매칭·1차 중요도 점수 | **글로벌성**(다국어, 다국가) |
| **다이제스트 봇 (소비자 A)** | 사용자 취향에 맞는 Top N 선별 | 카테고리 필터·개인 관심사 |
| **주가분석기 L3 (소비자 B)** | 시장 임팩트 이벤트 추출 | 경제·지정학·통화 |
| **Project A 지정학 (소비자 C)** | 국가관계 변화 신호 | 정상회담·제재·외교·군사 |
| **Project B 시장임팩트 (소비자 D)** | 자산가격 영향 가능 이벤트 | 통화·금리·원자재·기업 |

**즉, news_intel은 "넓게" 잡고, 소비자가 "좁게" 필터.**

이게 옳은 이유: 다이제스트 봇이 필터를 너무 강하게 하면 다른 소비자(주가분석기 L3 등)가 도킹할 때 정보 손실. "양봉가 한탄"은 다이제스트엔 노이즈여도 환경 ETF 분석엔 시그널일 수 있음.

```
┌─────────────────────────────────────────────────────────┐
│ news_intel  (공용 데이터 인프라 — 글로벌 광범위 수집)    │
│                                                          │
│  GDELT 2.0 이중 파이프라인 → articles                    │
│  → TF-IDF (72h 윈도우, 통합 벡터 공간) + 코사인 클러스터링│
│  → importance v2: log(size) + Σauth×(lang/3) + min(persist,72)/24 │
│  → /digest 수동 트리거 봇                                │
└────────────┬─────────────────────────────────────────────┘
             │
   ┌─────────┼──────────────────────────────┐
   ↓         ↓                              ↓
┌──────────┐  ┌───────────────────────────────────────┐
│ 소비자 A  │  │  후속 도킹 (이 프로젝트 범위 밖)        │
│ 다이제    │  │   - 주가분석기 L3                      │
│ 스트봇    │  │   - Project A 지정학                  │
│ 🟢 가동중 │  │   - Project B 시장 임팩트              │
└──────────┘  └───────────────────────────────────────┘
```

---

## 2. 주가분석기와의 관계

`stock_analyzer`는 **완전히 별도 리포·별도 프로젝트**.

```
~/Projects/claude/
├── news_intel/              ← 이 프로젝트
└── stock_analyzer/          ← 별도 (이 문서 범위 밖)
```

### 도킹 타이밍

```
Week 1~7: news_intel P1~P5 진행
Week 8+:  도킹 — NewsBundle 계약 문서 + 익스포터
          의존성 방향: stock_analyzer → news_intel
```

### 한국 국내 뉴스·증시 뉴스 경계

`news_intel`은 **국제 뉴스 인프라**. 한국 국내 세부 이슈·증시 전용 뉴스는 **주가분석기 L4의 영역**.

---

## 3. 확정 결정 (변경 금지)

### 3-A. 데이터 소스

| # | 결정 | 근거 |
|---|------|------|
| 1 | **Tier 1 = GDELT 2.0 이중 파이프라인** — 본체 GKG + Translation GKG | Day 1 실측 |
| 1-a | **본체 GKG**: 영어권 광역 (.com 78%, .uk 15%) | 24h 실측 |
| 1-b | **Translation GKG**: 비영어권 (.de 8.7%, .cn 6.7%, .kr 2.7%, .jp 2.2%) | Day 1 실측 |
| 1-c | **폴링 엔드포인트 2종**: `lastupdate.txt` + `lastupdate-translation.txt` | 비동기 |
| 2 | **BigQuery 유보** — 도킹 후 주가분석기 백테스팅용. GDELT BigQuery 공개 데이터셋은 2015년부터 전체 역사 접근 가능 (로컬 90일 롤링과 보완). 쿼리당 스캔량 과금 ($5/TB, 무료 1TB/월) | MVP 불필요 |
| 3 | **RSS는 MVP 범위 외** (P4에서 통신사 RSS 보강) | Translation GKG로 다국어 해결 |
| 4 | **상용 API 전면 배제** | GDELT 이중 파이프라인 상위호환 |
| 5 | **GDELT DOC 2.0 API도 배제** | Day 1 실측 품질·rate limit 문제 |
| 6 | **최근 90일 롤링 윈도우** | 외장 SSD 2TB 환경 |

### 3-B. 저장·처리

| # | 결정 | 근거 |
|---|------|------|
| 7 | Postgres 16 (Docker) | JSONB·GIN·동시 쓰기 |
| 8 | 데이터=외장 SSD 2TB, 코드=내장, pgdata=내장 (`~/docker_data/news_intel/pgdata/`) | Day 0 실측 |
| 9 | 원본 CSV 90일 보관, 양 파이프라인 동일 정책 | 재다운로드 가능 |
| 10 | 집계·스코어링 결과(`clusters`, `daily_top_n`) 영구 보관 | 재계산 비용 |
| 11 | 매체 권위 사전·엔티티 사전 영구 보관 | 노동 투입 자산 |
| 11-a | **용량 (v3.4 갱신)**: 24h 실측 약 340,000건/일 → 90일 약 30M 건. Postgres ~500GB, SSD 2TB 수용 | Day 1 예측 26배 |

### 3-C. 시간 처리 (v3.5: 미래 timestamp 가드 추가)

| # | 결정 | 근거 |
|---|------|------|
| 12 | 모든 시간 UTC 저장, 표시 시점 KST 변환 | 일관성 |
| 13 | 시간 필드 다중: `first_seen_at`, `gdelt_added_at`, `precise_pub_timestamp`, `url_path_date` | 교차 검증 |
| 14 | **`canonical_time` 우선순위**: ①`precise_pub_timestamp` (단, NOW() 이하) → ②`gdelt_added_at` | 24h 실측: 본체 52.7%, translation 44.4% precise 제공 |
| 15 | **`time_confidence` 2단계**: `1.0`=precise 유효 / `0.8`=fallback | 24h 합산 47.5% |
| 16 | URL 정규화 필수: `utm_*`, `fbclid`, `ref`, `#fragment`, `/amp/` 제거 | 중복 방지 |
| 16-a | **(v3.5 신규) 미래 timestamp 가드**: precise_pub_timestamp가 NOW()(UTC) 초과 시 자동으로 gdelt_added_at fallback + WARNING 로그. shared/time_utils.py compute_canonical_time()에 적용 | Day 5 발견: 일부 GDELT 메타데이터의 미래 시간 (예: 5개월 뒤). cluster persistence_hours 폭주 원인 |

### 3-D. 언어 처리

| # | 결정 | 근거 |
|---|------|------|
| 17 | **MVP v0.1 = 한·영 + translation 전 언어 저장** | 스코프 |
| 18 | translation의 다국어 articles 테이블에 저장(`language` 필드 구분), MVP는 한·영 중심 처리 | P4 단계적 활성화 |
| 19 | **통합 벡터 공간 + 언어 메타데이터 보존** (v3.2 원문에서 v3.3에 수정) | Week 1 실측: translation의 persons 엔티티가 영어 표기 ("Lee Jae" 등) — 영어권과 자연 매칭 |
| 19-a | **(v3.5 확증) 통합 벡터 공간 가설 입증**: Week 2 Day 3 v3 실측에서 multilang(language_count≥2) 클러스터 1,217개 확인. 통합 벡터 공간이 의도대로 작동 | Day 3 v3 검증 |
| 20 | 한국어 토크나이저 = **Kiwi** | Day 0 검증 |
| 21 | 영어 토크나이저 = sklearn 기본 + 뉴스 불용어 사전 | 경량 |
| 22 | **언어 자동 감지 후 Kiwi/sklearn 분기 → 통합 벡터 공간 merge** | langdetect + Unicode 휴리스틱 |
| 23 | `sublinear_tf=True`, `min_df=2`, `max_df=0.8` | 뉴스 TF-IDF 표준 |

### 3-E. 클러스터링·스코어링 (v3.5: 공식 v2 반영)

| # | 결정 | 근거 |
|---|------|------|
| 24 | 클러스터링 = **TF-IDF + 코사인** | 고유명사 정밀도 |
| 25 | **클러스터링 윈도우 = 72시간 슬라이딩** | 균형점 |
| 26 | **TF-IDF 입력 = 제목 + persons + organizations + locations 이름** | 제목 핵심 + 엔티티 앵커 |
| 27 | **통합 벡터 공간** | translation 엔티티가 영어 표기 |
| 27-a | **(v3.5 확정) 클러스터링 알고리즘**: KMeans(k=√n) Phase 1 + 재귀 KMeans Phase 2 (500건 초과 클러스터 분할, max_depth=4). data/clusterer.py 구현됨 | Day 3 v1·v2·v3 실험: AgglomerativeClustering 메모리 한계, 단일 KMeans는 메가 catch-all 발생, 재귀 분할이 균형점 |
| 28 | **(v3.5 갱신) importance 공식 v2**: `log(size+1) + Σ(authority) × (language_count/3.0) + min(persistence_hours, 72) / 24` | Day 4 검증: v1 (Σ만) 사용 시 단일국 가십 (스페인 Marc Giró 등)이 Top 진입. language_count 가중치로 글로벌성 반영. persist cap으로 미래 timestamp 흡수된 클러스터 방어 |
| 29 | **실측 후 추가 조정 가능**: Week 3 운영 중 themes 기반 카테고리 가중치 검토 | 세열님 직감 + 실측 |
| 30 | **매체 권위 사전 확장 시점** = Week 2 Day 1 ✅ 완료 | Week 1 24h 실측 기반 |
| 31 | **권위 점수 Tier**: Tier1(통신사) 1.0 / Tier2(1급 국제) 0.95 / Tier3(주류) 0.90 / Tier4(국가 대표) 0.85 / Tier5(주요 상업지) 0.80 / Tier6(지역 대표) 0.70 / Tier7(전문지·소형국) 0.65 / Tier8(미등재 기본값) 0.5 / 신뢰도 낮음 0.30 / **블랙리스트 0.01** | 가중치 0이 아닌 0.01: 지역 매체라도 수천 곳 동시 보도 시 신호 보존 |
| 31-a | **(v3.5 확정) 블랙리스트 authority 0.01 유지**: Day 4 검증에서 Iran #1 클러스터(856건)에 블랙리스트 12% 포함됐으나 Top 순위 영향 미미. 0.005 하향 불필요 결정 | Day 4 실측 |
| 31-b | sources 테이블 최종: 219개 매체 (Tier1=2, Tier2=8, Tier3=5, Tier4=7, Tier5=18, Tier6=48, Tier7=53, default=5, low=42, blacklist=31) | Day 1 Step 1-C 적용 |
| 32 | **임베딩 도입 금지** (MVP~P4) | Mac M4 리소스 |

### 3-F. LLM 사용

| # | 결정 | 근거 |
|---|------|------|
| 33 | **MVP(P1+P2)에서 LLM 호출 없음** | zero cost |
| 34 | LLM 검증 레이어는 **P5에 도입** | 법률봇과 동일 |
| 35 | LLM 호출 도입 후에도 news_intel 내부에서만 | 레이어 분리 |

### 3-G. 프로덕트화

| # | 결정 | 근거 |
|---|------|------|
| 36 | **소비자 A = 텔레그램 봇** | 법률봇 자매 |
| 37 | 웹 UI·회원가입·결제 **금지** (전 Phase) | 스코프 |
| 38 | **봇 = `/digest` 수동 트리거 전용. 자동 일일 스케줄 구현 금지 (전 Phase)** | 세열님 사용 패턴 |
| 39 | **Week 2 = `/digest` 기본**, **Week 3 = `/geo` `/market` 추가 + 포맷 개선** | 명령어 확장 |
| 40 | **클러스터링은 orchestrator 1시간 백그라운드** — `/digest` 호출 시 저장된 결과 조회 | 봇 즉시 응답 |
| 40-a | **(v3.5 확정) Week 2 Day 5 구현 완료**: orchestrator schedule에 clusterer+scorer 1h 주기 통합. SSD 마운트 가드 모든 작업에 적용 | Day 5 |
| 41 | 데이터 레이어와 봇 레이어 패키지 단위 분리 | 재사용성 |

### 3-H. 아키텍처

| # | 결정 | 근거 |
|---|------|------|
| 42 | 단일 리포 `~/Projects/claude/news_intel/` (내장) | 코드 성능 |
| 43 | 원본 CSV 외장 SSD `/Volumes/AIDRIVE/news_intel/data/raw/{gkg,translation_gkg}/` | 용량 |
| 44 | 패키지 분리: `data/` + `bot/` + `shared/` + `scripts/` + `migrations/` | 재사용 |
| 45 | API 키 `.env`만, `.gitignore` | 커밋 방지 |
| 46 | `.env` 로드 시 `override=True` 필수 | Day 2 실측 |
| 47 | 캐시 TTL 24h + `force_refresh` | 쿼터 보호 |

### 3-I. 운영·협업

| # | 결정 | 근거 |
|---|------|------|
| 48 | **Git = GitHub private** (`https://github.com/q28902/news_intel`) | 백업 |
| 49 | **모든 문서 = 리포 `docs/`** | 단일 진실 |
| 50 | **Claude Code 승인 = 모듈별 단계 리뷰** (Day별 독립 브랜치) | 법률봇 패턴 |
| 51 | 데이터 관찰 = Markdown 표 업로드 | 핸드오프 통합 |
| 52 | **Claude Code 프롬프트는 `cd <REPO_ROOT>/Projects/claude/news_intel && pwd` 박제로 시작** | 다른 프로젝트 디렉토리 시작 시 경로 오염 방지 |
| 53 | **프로젝트 루트 CLAUDE.md 배치** | 자동 로드로 맥락 오염 완화 |
| 54 | **세션 길어지면 Step 분리** | 컨텍스트 오염 방어 |
| 54-a | **(v3.5 신규) 프롬프트 작업량 ≥500줄 시 단계 분할 권장**: secretarybot 등 래퍼의 비용/시간 한도 회피. Day 5 5-A/5-B/5-C/5-D 4단계 분할이 효과적이었음 | Day 5 비용 한도 경험 |

### 3-J. 장애 복구·내구성 (v3.5: 구현 완료)

| # | 결정 | 근거 |
|---|------|------|
| 55 | 이미 구현된 복구: ON CONFLICT, Postgres 재시도, GDELT 타임아웃 WARN, 파일 단위 격리, graceful shutdown | Day 0~5 |
| 56 | **(v3.5 완료) 외장 SSD 마운트 체크**: `shared/health.py` `check_ssd_mount()` 구현. orchestrator의 watcher/downloader/parser/clusterer 모든 작업 직전에 체크. 실패 시 `system_log` 기록 + 작업 스킵 | Day 5 |
| 57 | **(v3.5 완료) launchd plist**: `scripts/launchd/com.newsintel.orchestrator.plist` 작성. KeepAlive=true로 자동 재시작. 설치는 세열님이 별도 시점 결정 (수동) | Day 5 |
| 58 | 헬스체크 watchdog 쓰레드 (P3 이후 검토) | Week 3+ |

---

## 4. 경계와 오해 방지

| 헷갈림 | 정답 |
|---|------|
| `news_intel`이 주가분석기 서브모듈? | 아니다. 독립 프로젝트. P5 후 도킹 |
| MVP 범위? | 단독 = 프로덕트 MVP (P1+P2) |
| 본문 요약·임베딩? | 안 함. 제목 + GKG 메타 + description |
| 한국 주식 뉴스 다루나? | 아니다. L4 영역 |
| 한국 국내 세부 이슈? | 아니다. L4 영역 |
| 다이제스트 봇이 판단? | 아니다. 1차 점수만. 해석은 사용자 |
| 상용 뉴스 API? | 안 된다. GDELT 이중 파이프라인 |
| BigQuery 언제? | MVP 불필요. 도킹 후 백테스팅 |
| GDELT 15분 정확? | Week 1 24h 실측 100% 15분 이내. 시간대·날짜 따라 편차 |
| V2ExtrasXML JSONB만? | 아니다. 핵심 태그는 별도 컬럼 평탄화 |
| 본체 GKG만? | 아니다. Translation GKG 함께 |
| GDELT DOC 2.0 API? | 안 된다 |
| 언어별 독립 클러스터링? | 아니다. 통합 벡터 공간 |
| 클러스터링 범위? | 72h 슬라이딩 |
| 블랙리스트 완전 차단? | 아니다. 0.01 유지 (Day 4 검증으로 0.005 하향 불필요 확정) |
| 자동 발송? | 안 한다. `/digest` 수동만 (전 Phase) |
| 클러스터링 언제? | orchestrator 1h 백그라운드. `/digest`는 조회만 |
| WSJ·FT·AP 부재? | Week 1 실측 6/12 Tier-1만 잡힘. P4 RSS 보강 (방법 미정) |
| **Top 10에 가십 섞이면? (v3.5)** | **인프라 단계는 광범위 수용. 카테고리 필터링은 상위 소비자 책임 (§1)** |
| **importance 공식? (v3.5)** | **v2: log(size+1) + Σauth × (lang/3) + min(persist,72)/24** |
| **persistence_hours 이상치 (5개월 등)?** | **scorer가 72h cap 적용. 미래 timestamp는 time_utils가 자동 fallback (§3-C #16-a)** |
| **봇 어떻게 가동?** | nohup 백그라운드 (operations.md 참조). launchd 자동 재시작은 선택 |

---

## 5. 폴더·파일 구조

### 코드 (내장)

```
~/Projects/claude/news_intel/
│
├── CLAUDE.md                 # Claude Code 자동 로드
├── .env                      # 커밋 금지
├── .gitignore
├── README.md
├── requirements.txt
│
├── docs/
│   ├── SESSION_HANDOFF.md    # ⭐ 이 문서 (v3.5)
│   ├── week1_tasks.md        # Week 1 완료
│   ├── week2_tasks.md        # Week 2 완료
│   ├── week3_tasks.md        # Week 3 (작성 예정)
│   ├── operations.md         # launchd 가이드 포함
│   ├── observations/
│   │   ├── day1_gdelt_structure.md
│   │   ├── day6_7_observation.md       # Week 1 24h
│   │   ├── day4_scorer_results.md      # Day 4 검증
│   │   └── day5_completion.md          # Week 2 Day 5 완료
│   └── decisions/
│
├── shared/
│   ├── config.py             # .env (override=True)
│   ├── db.py
│   ├── schemas.py
│   ├── url_utils.py
│   ├── time_utils.py         # 미래 timestamp 가드 적용
│   ├── xml_utils.py
│   ├── logging_util.py
│   ├── lang_detect.py        # Week 2 Day 2
│   ├── tokenizer.py          # Kiwi + sklearn
│   └── health.py             # SSD 마운트 체크
│
├── data/
│   ├── masterfile_watcher.py
│   ├── downloader.py
│   ├── gkg_parser.py         # csv.field_size_limit(1MB)
│   ├── clusterer.py          # 72h KMeans + 재귀 Phase 2
│   ├── scorer.py             # importance v2
│   ├── cleanup.py
│   └── orchestrator.py       # clusterer/scorer 1h 주기 + SSD 가드
│
├── bot/
│   ├── sender.py             # Telegram API 래퍼
│   ├── renderer.py           # HTML 다이제스트
│   └── commands.py           # /digest, /start, /help (수동)
│
├── scripts/
│   ├── init_db.py
│   ├── seed_sources.py       # 초기 20개
│   ├── seed_sources_v2.py    # 200개 확장
│   ├── source_authority_v2.csv
│   ├── analyze_top_sources.py
│   ├── output/
│   │   └── top_sources_analysis.md
│   └── launchd/
│       └── com.newsintel.orchestrator.plist
│
├── migrations/
│   ├── 001_init.sql          # gdelt_files, articles, sources, system_log
│   └── 002_clusters.sql      # clusters, daily_top_n
│
└── logs/
```

### 데이터 (외장 SSD 2TB)

```
/Volumes/AIDRIVE/news_intel/
├── data/raw/
│   ├── gkg/                   # 90일 롤링
│   ├── translation_gkg/
│   ├── events/                # P5+
│   ├── mentions/
│   └── translation_events/
└── logs/
```

---

## 6. 스키마 (현재 운영 중)

### 적용된 마이그레이션

- `001_init.sql` (Week 1): gdelt_files, articles, sources(219개), system_log
- `002_clusters.sql` (Week 2 Day 2): clusters, daily_top_n

### clusters 테이블

```sql
CREATE TABLE clusters (
    id                    BIGSERIAL PRIMARY KEY,
    canonical_title       TEXT,
    first_article_time    TIMESTAMPTZ NOT NULL,
    last_article_time     TIMESTAMPTZ NOT NULL,
    persistence_hours     NUMERIC(6,2),
    article_count         INT NOT NULL,
    language_count        INT,
    country_count         INT,
    importance_score      NUMERIC(8,3) NOT NULL,
    language_distribution JSONB,
    top_entities          JSONB,
    computed_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    window_start          TIMESTAMPTZ NOT NULL,
    window_end            TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_clusters_importance ON clusters(importance_score DESC);
CREATE INDEX idx_clusters_window ON clusters(window_end DESC);
```

### daily_top_n 테이블

```sql
CREATE TABLE daily_top_n (
    id              BIGSERIAL PRIMARY KEY,
    run_date        DATE NOT NULL,
    rank            INT NOT NULL,
    cluster_id      BIGINT REFERENCES clusters(id),
    importance_score NUMERIC(8,3),
    sent_at         TIMESTAMPTZ,
    UNIQUE (run_date, rank)
);
```

---

## 7. Phase 로드맵

| Phase | 범위 | 상태 |
|---|---|---|
| **P1 (Week 1)** | 데이터 레이어 MVP | ✅ **완료 2026-04-22** |
| **P2 (Week 2)** | 클러스터링·스코어링 + 수동 봇 | ✅ **완료 2026-04-23** |
| **P3 (Week 3)** | 운영 안정화 + `/geo` `/market` + 튜닝 | 🔵 다음 |
| **P4 (Week 4~5)** | 품질 고도화: 엔티티 정규화, RSS 보강, 다국어 활성화 | |
| **P5 (Week 6~7)** | LLM 검증 레이어 | |
| **도킹 (Week 8+)** | 주가분석기 L3 연결 | 범위 외 |

### Week 3 상세 (P3) — 검토 항목

| 작업 | 우선순위 |
|---|---|
| 5일 연속 안정 운영 검증 | 🔴 필수 |
| `/geo` `/market` 카테고리 명령어 | 🟡 중요 |
| themes 기반 카테고리 가중치 (가십 분별) | 🟡 검토 |
| 한국 매체 활용 (`/kr` 등) | 🟢 선택 |
| 메시지 포맷 개선 (이모지, 그룹핑) | 🟢 선택 |
| importance 공식 추가 조정 (실측 결과 기반) | 🟢 선택 |
| launchd plist 실제 설치 | 🟢 선택 |

Week 3 명세서는 별도 `docs/week3_tasks.md` 작성 예정 (Day 6~7 운영 결과 보고).

---

## 8. Do Not 리스트 (엄수)

1. 주가분석기 리포 코드·문서 수정 금지
2. 임베딩 모델 도입 금지 (MVP~P4)
3. API 키 하드코딩 금지
4. 한국 주식 시장 전용 뉴스 파이프라인 금지
5. 한국 국내 세부 이슈 중심 파이프라인 금지
6. LLM 호출 금지 (MVP~P4)
7. §3 확정사항 임의 변경 금지
8. 도킹 스키마(`NewsBundle`) 지금 만들지 금지
9. 웹 UI·회원가입·결제 금지
10. 상용 뉴스 API 금지
11. BigQuery MVP 단계 세팅 금지
12. RSS 도입 금지 (MVP~P3)
13. 단일 파일 >500MB 메모리 로드 금지
14. 독·중·일 클러스터링 MVP(P2)에 활성화 금지
15. 원본 CSV 내장 저장 금지
16. 심링크 사용 금지
17. `main` 직접 푸시 금지
18. 외부 단독 문서 작성 금지
19. V2ExtrasXML JSONB 덩어리만 저장 금지
20. GDELT 파일 누락 ERROR 처리 금지
21. 본체 GKG만 수집 금지
22. GDELT DOC 2.0 API 도입 금지
23. 본체·translation 같은 file_type 처리 금지
24. 언어별 독립 클러스터링 금지 (통합 벡터 공간)
25. 블랙리스트 완전 차단(authority=0) 금지 (0.01 유지, Day 4 실측으로 0.005 하향 불필요 확정)
26. `.env` `override=False` 금지
27. **자동 발송 스케줄 구현 금지 (전 Phase)** — `/digest` 수동만
28. 클러스터링 윈도우 24h/7일 선택 금지 (72h)
29. Claude Code 프롬프트 `cd <REPO_ROOT>/Projects/claude/news_intel && pwd` 박제 생략 금지
30. CLAUDE.md 임의 작성 금지 (Week 1 완료 시 배치됨)
31. **(v3.5 신규) 다이제스트 카테고리 필터링을 news_intel 코어에 구현 금지** — 인프라는 광범위 수집. 카테고리 필터는 봇 명령어 또는 상위 소비자 레이어
32. **(v3.5 신규) precise_pub_timestamp 미래 시간을 그대로 canonical_time에 사용 금지** — time_utils.py가 자동 fallback. 우회 시 클러스터 persistence_hours 폭주
33. **(v3.5 신규) clusterer Phase 2를 단순 해체(각 기사 = 독립 클러스터) 처리 금지** — 재귀 KMeans 분할 사용. 단순 해체 시 99% singleton 발생 (Day 3 v1 사례)

---

## 9. 작업 환경

| 항목 | 값 |
|------|-----|
| 하드웨어 | Mac mini M4 / 16GB RAM |
| OS | macOS |
| Python | 3.11 (venv `~/venv/`) |
| DB | Postgres 16 (Docker, container `news_pg`) |
| pgdata | `~/docker_data/news_intel/pgdata/` |
| 원본 데이터 | `/Volumes/AIDRIVE/news_intel/data/` |
| 코드 | `~/Projects/claude/news_intel/` |
| Git | `https://github.com/q28902/news_intel` |
| 시계 | UTC 저장 / KST 표시 |

### Week 2 종료 시점 검증 완료

| 검증 | 결과 |
|------|-----|
| Docker Postgres | ✅ 안정 |
| GDELT 이중 파이프라인 | ✅ |
| articles 적재 (P1 24h) | ✅ 339,905건 |
| 클러스터링 작동 (P2) | ✅ 4,646 clusters / multilang 1,217 |
| 메모리 (clusterer) | ✅ 3.3GB peak |
| 메모리 (orchestrator+bot 가동) | ✅ 각 52MB |
| 스코어링 작동 | ✅ avg=23.9, max=1154.7 |
| `/digest` 응답 | ✅ Top 10 정상 |
| Tier-1 영어 매체 | 🟡 6/12 (P4 RSS 보강 예정) |
| 한국 매체 | ✅ 20개 (Week 1) / 클러스터 299개 기여 (Week 2) |
| graceful shutdown | ✅ |
| SSD 마운트 가드 | ✅ Day 5 |

---

## 10. 운영 중 발견된 이슈

### 10-A. csv.field_size_limit ✅ 해결

- 증상: gkg 파일 3건 파싱 실패 (`field larger than field limit (131072)`)
- 조치: `data/gkg_parser.py`에 `csv.field_size_limit(1_000_000)` 추가
- 적용: 커밋 d184505

### 10-B. 매체 권위 사전 시드 부적합 ✅ 해결

- 증상: 실측 Top 50의 198/200이 시드 미등재
- 조치: 200개 매체 분석 + 219개 등재 + 31개 블랙리스트
- 적용: Day 1 Step 1-C, scripts/seed_sources_v2.py

### 10-C. 블랙리스트 0.01 검증 ✅ 유지 결정

- 우려: 500+ 복제 시 Top 역전 위험
- 실측: Day 4에서 Iran #1(856건)에 블랙리스트 12% 포함됐으나 영향 미미
- 결정: 0.01 유지 (0.005 하향 불필요)

### 10-D. 미래 timestamp + cluster persistence 폭주 ✅ 해결

- 증상: Bundeswehr 클러스터 persistence_hours=3,476h(144일), Lourdes=1,463h(60일)
- 원인: 일부 GDELT 메타데이터의 precise_pub_timestamp가 미래 시간(예: 5개월 뒤). 클러스터의 last_article_time을 오염시켜 persistence 폭주
- 영향: 615건 미래 timestamp 기사 발견 (대부분 UTC/KST 시차 정상, 진짜 이상치 4건)
- 조치: 
  1. 615건 일괄 정정: `canonical_time = first_seen_at, time_confidence = 0.5`
  2. clusters 테이블 `last_article_time` 재계산
  3. **shared/time_utils.py 영구 패치**: `compute_canonical_time()`에 미래 시간 가드 추가
  4. **scorer.py 영구 가드**: `min(persistence_hours, 72)` cap
- 적용: 커밋 615aa1a

### 10-E. importance 공식 v1 → v2 ✅ 적용

- 증상: Σ 방식이 단일국 가십(스페인 Marc Giró 등)을 Top 진입시킴
- 분석: language_count=1 클러스터가 Top 10의 30% 차지
- 조치: importance 공식에 `× (language_count / 3.0)` 가중치 추가
- 효과: 단일국 노이즈 자연 탈락, Iran #1 점수 1154.7로 압도적 1위
- 적용: 커밋 615aa1a

### 10-F. clusterer Phase 2 알고리즘 진화 (Day 3 v1→v2→v3)

- v1 (AgglomerativeClustering): 메모리 O(n²) — 1M 건 처리 불가
- v2 (단일 KMeans k=√n + 단순 Phase 2 해체): 99.7% singleton 또는 메가 catch-all
- **v3 (KMeans + 재귀 KMeans Phase 2)**: ✅ 채택. 4,646 clusters, multilang 1,217, max 856
- 교훈: §3-E #27-a에 알고리즘 결정 명문화

---

## 11. P1·P2 완료 기준 (실측 결과)

### P1 (Week 1) — 데이터 레이어 MVP

| 기준 | 목표 | 24h 실측 | 달성 |
|------|------|---------|------|
| articles 누적 | 25,000+ | **339,905** | ✅ 13.6배 |
| 중복 URL | 0 | 0 | ✅ |
| 한국어 매체 | 8개+ | **20개** | ✅ 2.5배 |
| 영어 Tier-1 | 12개+ | 6개 (Guardian, BBC×2, Reuters, Bloomberg, NYT) | 🟡 P4 RSS |
| 파일 실패율 | 5%↓ | 1.6% | ✅ |
| 메모리 peak | 1GB↓ | 40MB | ✅ |
| ERROR (파일 부재 외) | 0 | csv 3건(패치) + GDELT 타임아웃 WARN | ✅ |
| time_confidence 1.0 | 40%+ | 47.5% | ✅ |
| source_file_type | gkg 55/trans 45 (예상) | gkg 36/trans 64 | ✅ translation 우세 |

### P2 (Week 2) — 프로덕트 MVP

| 기준 | 목표 | 실측 | 달성 |
|------|------|---------|------|
| clusters 생성 | 정상 분포 (메가 catch-all 없음) | **4,646 clusters** (max 856, median 15) | ✅ |
| Singleton 비율 | <30% | 10.3% | ✅ |
| Multilang 클러스터 | >100 | **1,217** | ✅ 12배 |
| importance 공식 | log+Σauth+persist | **v2: + ×(lang/3) + cap72** | ✅ 갱신 |
| `/digest` 응답 | <30초 | 즉시 (저장 결과 조회) | ✅ |
| `/digest` Top 10 의미 | 글로벌 이슈 다수 | Top 10에 5개 강한 임팩트 + 5개 글로벌 묶임 | ✅ MVP 수용 |
| 봇 + orchestrator 가동 | 정상 | 각 52MB 메모리 | ✅ |
| SSD 마운트 가드 | 모든 작업 | watcher/downloader/parser/clusterer 적용 | ✅ |
| launchd plist | 작성 | scripts/launchd/ | ✅ (설치 보류) |
| 한국 매체 클러스터 기여 | 측정 | 299개 클러스터, 1,268건 (72h) | ✅ |

### Week 1·2 도출 설계 피드백

- translation_gkg 비중 64% (예상 45%) — 다국어 커버리지 풍부
- GDELT 발행 주기 100% 15분 이내 (24h 샘플)
- Tier-1 영어 매체 6/12 — P4 RSS 보강 필요 (방법 미정)
- 통합 벡터 공간 가설 입증: multilang 1,217 클러스터
- 단일국 가십 노이즈 패턴 → language 가중치로 해결
- 다국어 가십 잔존 → 카테고리 필터는 상위 소비자 책임 (§1)

---

## 12. 문서 업데이트 정책

- 모든 문서 리포 `docs/` 버전 관리
- 중대 변경 시 상단 주석에 버전 번호·날짜
- Claude Code는 이 문서 직접 수정 불가, 변경 제안은 세열님 보고
- Phase 완료 시 §7 기록
- 의사결정 기록은 `docs/decisions/`

---

## 13. 나중 결정 예정 항목 (보류)

| 항목 | 결정 시점 |
|------|-----------|
| 텔레그램 봇 최종 이름 | 운영 정착 후 |
| 영어 Tier-1 매체 P4 보강 방법 (RSS / OG 스크래핑 / 상용 API) | P4 착수 시 실측 재평가 |
| themes 기반 카테고리 가중치 (가십 분별) | Week 3 운영 결과 기반 |
| `/geo` `/market` 명령어 카테고리 정의 | Week 3 |
| 한국 매체 활용 패턴 (`/kr` 등) | Week 3 운영 결과 |
| 엔티티 정규화 세부 구현 | P4 |
| 독·중·일 클러스터링 활성화·토크나이저 | P4 (jieba, fugashi, 독일어 spaCy) |
| LLM 검증 프롬프트 | P5 |
| BigQuery 쿼리 설계 | 도킹 후 |
| 다이제스트 봇 추가 명령어 (`/week`, `/entity`) | MVP 운영 후 |
| `NewsBundle` 계약 스키마 | Week 8+ 도킹 |
| Translation Events/Mentions 파이프라인 | P5+ |
| 한국 국내 뉴스 RSS (주가분석기 L4) | L4 착수, news_intel 밖 |
| 헬스체크 watchdog 쓰레드 | P3+ |
| launchd plist 실제 설치 | 봇 운영 정착 후 |
| importance 공식 추가 조정 (Σ→MAX 또는 다른 가중치) | Week 3 실측 후 |

---

**문서 상태**: v3.5 (2026-04-23 Week 2 P2 완료 + 모든 결정 통합)
**Week 1·2 산출물**: 14개 파이프라인 모듈 + 219 매체 사전 + 4,646 클러스터링 작동 + `/digest` 봇 가동
**다음 변경 트리거**: Day 6~7 운영 관찰 결과 또는 Week 3 진입 시

---

## 14. Week 3 T1 진행 상태 (2026-04-23 추가, v3.6 리라이트 전 임시)

### 14-A. 완료된 것

- **Step 1-C-2**: cluster_runs 테이블 + run_id FK 도입 (커밋 `5e1d3ef`, branch `feature/cluster-runs-history`)
- **Step 1-E**: TF-IDF 파라미터 튜닝 (max_df 0.8→0.5, min_df 2→5, k 공식) — 크기 지표 완화, **의미 경계 실패** 재확인 (커밋 `6ad88d0`)
- **F1 착수**: bge-m3 + UMAP + HDBSCAN 파이프라인 구현 (branch `feat/f1-bge-m3-hdbscan`)
- **migration 004**: cluster_runs에 noise_count / noise_ratio 컬럼 추가 (HDBSCAN 통계용)

### 14-B. F1 중단 (2026-04-23 밤)

- **run_id=6 실측**: max_size 2,160 / mega 5개 / noise 46% — **3지표 전부 실패선 초과**
- **언어 catch-all 문제**: top5 중 4개 cluster가 단일 언어 집중(ru/ja/es/en), Run 4의 "무관 혼재"가 "언어별 catch-all"로 자리만 바꿈
- **임베딩 소요 87분** (1h 주기 운영 부적합)
- 상세: `docs/decisions/week3_phase1_embedding.md` Appendix + `week3_phase1_f2_plan.md`

### 14-C. F2 방향 전환 (2026-04-24 오전)

- 기존 "언어 catch-all" 진단이 부분 오판으로 확인 (`week3_phase1_embedding.md` 정정 기록)
- 실제 원인: 데이터 품질 한계 + 엔티티 mention 중복 + 입력 빈약
- "언어 2단계화" 폐기 → **A+ 경로**: F2 품질 필터 3종 + `_build_embedding_text` 중복 제거 (커밋 `01cdfff`, `2c6fba2`)
- F 경로: Phase 2 재활용 (embedding 공간 배선, 커밋 `14fcc23`)

### 14-D. Run 7~8 진행 기록 (2026-04-24)

**Run 7** (HEAD `9f6d43d`, F2 첫 실측):
- F2 품질 필터 + entity dedup 적용, Phase 2 비활성
- 정량: 총 38분 / article 57,289 / cluster 817 / noise 38.26% / max_size **1,602** / mega 2
- 정성: top2~10 주제 일관성 확보 (persons_per_article 0.17~0.28)
- top1(1,602건)만 UK/IE 지역 단신 catch-all 잔존
- 판정: **부분 성공**

**Run 8** (HEAD `7be5637`, F 경로 추가):
- F2 + Phase 2 활성 (`NEWS_INTEL_ENABLE_PHASE2=true`, threshold=1500)
- 정량: 총 37분 / article 57,752 / cluster 833 / noise 37.77% / max_size **1,424** / mega 5
- **Phase 2 미작동**: targets=0, splits=0 (HDBSCAN 단계에서 이미 1,424로 내려가 threshold 1500 미달)
- max_size 개선은 **UMAP 비결정성 기인**, Phase 2 효과 아님
- catch-all 정체 이동: **UK(Run 7 top1) → Indonesia(Run 8 top1)**, 본질 미해결
- 판정: **정량 부분 성공 / 정성 실질 실패**

### 14-E. 대기 중인 결정 (다음 세션)

- **C) 재실행 평균**: UMAP 비결정성 진단 (~70분 × 2회)
- **B) source tier 필터**: F4 레이어 신설 (authority<0.3 제외 등)
- **G) Run 8 수용 + main 머지**: `/digest`는 이미 Run 8 기반 응답 중
- **D) 아키텍처 재설계**: 다른 임베딩 모델 / 하이브리드
- **E) Phase 2 threshold 1200으로 낮춰 재시도** (A 계열)

### 14-F. 구조적 관찰 (다음 세션 참고)

Run 3/4/6/7/8 전반에서 **"큰 catch-all의 정체만 바뀌고 존재는 유지"** 패턴. Phase 2 같은 사후 처리보다 **Phase 1 단계의 source tier 필터 또는 재학습**이 원인 지향일 가능성 높음.

### 14-G. 운영 중 상태

- orchestrator: **중단 유지** (자동 1h clustering 없음)
- bot (`/digest`): 가동 중, **run_id=8** 기반 응답 (TF-IDF Run 4에서 Run 8로 자동 전환됨)
- cluster_runs row 8개 존재 (1~4 TF-IDF, 5 OOM, 6 F1, 7 F2, 8 F경로)
- 브랜치: `feat/f1-bge-m3-hdbscan`, HEAD `7be5637` (+ SESSION_HANDOFF 커밋 예정)
- main 머지 승인 없음
- SESSION_HANDOFF **v3.6 리라이트 금지** (별도 승인 사항)

### 14-H. 인프라 / 표준

- **venv 전환 완료**: `~/venv/news_intel/` (Python 3.12.13) — SESSION_HANDOFF §9의 "Python 3.11" 기재와 불일치, v3.6에서 동기화 예정
- **장시간 작업 표준 보고 체계**: 6포인트 + Telegram/세션 이중화, `scripts/notify_telegram.sh` + `docs/operations.md` (커밋 `d16698b`)
- **ScheduleWakeup 자동 예약 규칙**: `/Volumes/AIDRIVE/CLAUDE.md`에 명문화 (≤15분→15분 1회 / ≤30분→30분 1회 / >30분→15분 간격). 다음 세션부터 자동 로드

---

## Run 9~10 진행 + D 옵션 결정 (2026-04-24)

### Run 9 (HEAD `0c83b8c`) — C 경로 UMAP 비결정성 진단
- 설정: Run 8과 완전 동일 (재실행 1회)
- 결과: max_size 1,416 / noise 39.44% / Phase 2 targets=0 (3회 연속 미작동)
- 3-way 편차 (Run 7/8/9): max_size range 186, noise range 1.67pp
- top1 catch-all 정체: UK/IE "Country diary" 재등장 (Run 8 Indonesia와 rank swap)
- 결론: UMAP 비결정성은 rank swap 수준, catch-all 구조는 데이터·임베딩의 구조적 반복 패턴 확정
- 정정 기록: top2(1,227)가 canonical_title "경희대" 뒤에 숨은 베트남 국내 catch-all이었음 확인

### Step B-0 (Source tier 진단)
- Run 9 기준 정상 cluster 분위수: p50=1 / p95=3 / p99=11 / max=338
- 대형 cluster 집중도:
  - top1 UK(1,416): `independent.ie` 71(5.0%) — 분산형
  - top2 VN(1,227): `baomoi.com` 228(18.6%) — 집중형
  - top3 KR(1,046): `heraldcorp.com` 113(10.8%) — 신규 catch-all 발견
- A 범주 공시/피드: `cfi.net.cn`, `dailypolitical.com`

### Run 10 (HEAD `df40a73`) — F4 필터 첫 검증
- 설정: cap=10 + 블랙리스트(`cfi.net.cn`, `dailypolitical.com`)
- 결과: max_size 1,063 (Run 9의 1,416 대비 **-25%**) / noise 40.14%
- F4 통계: blacklist_dropped 662 / cap_dropped 7,767 / clusters_affected 183
- 집중형 catch-all 해소:
  - VN 1,227 → 636 (**-48%**, `baomoi.com` cap 직격)
  - KR 1,046 → 535 (**-49%**, 북한 crypto 글로벌 주제로 분산)
- 분산형 catch-all 잔존: UK `.co.uk` 체인 40+ 도메인 각 10건 × 40 = 400+ (cap 우회, 사전 예측 실패 모드 #1)
- 새 catch-all 등장: 일본 매체군 top2(649), 독일 Oberursel top6(443)
- 판정: **부분 성공**

### `persons_per_article` 지표 재정의
- 기존: cluster 주제 일관성 판정 지표
- 문제: 언어권별 entity 풍부도 차이 무시 (베트남 기사 본래 적음)
- 재정의: 참고 지표만, 단독 판정 근거 금지

### canonical_title 함정 4회 반복 확인
Run 7 "Country diary", Run 8 Indonesia 라벨, Run 9 "경희대", Run 10 "Japan tsunami" — 모두 라벨과 실제 내용 불일치. **라벨/샘플/source 3축 검증 원칙** 재확인.

### LLM 하네스 최적화 평가 (54/100)
- 노이즈 75 / 응집도 55 / 라벨 20 / 토큰 45 / 결정성 95 / 확장성 60
- 규칙 기반 천장: 10일 투자 시 85점 (라벨은 50점 천장)
- 라벨 신뢰도 20점이 최대 병목으로 확정

### D 옵션 결정
- canonical_title 생성에만 Claude Haiku 4.5 투입
- 다른 영역은 규칙 기반 유지
- 예상 효과: 54 → 74점 (3~4일 투자)
- Phase L-1(최소 구현) / L-2(캐시+최적화) / L-3(catch-all 신호 활용)

### 다음 세션 착수 지점
1. Run 10 수용 + main 머지 (선행 안정화) ← 본 세션 종료 시점에 진행
2. D 옵션 Phase L-1 설계 착수
3. 쟁점 7개 세열 최종 결정 필요 (호출 범위, 입력 선정, 프롬프트, 실패 처리, 캐싱, 배치, 환경변수)

### 범위 밖 미결 과제
- UK `.co.uk` 분산형 catch-all (도메인 그룹화 필요, Week 4+)
- GKG language 오태깅 (`baomoi.com` VN→en 등)
- Phase 2 재설계 (3회 연속 미작동, size_threshold 1500 경계값 문제)
- Wakeup 인프라 신뢰도 (2회 연속 fire 실패)
- /digest LLM 소비 계층 (Week 5+)
