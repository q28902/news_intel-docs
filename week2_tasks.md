<!--
════════════════════════════════════════════════════════════════════
Week 2 작업 명세서 (Day 1~7)
════════════════════════════════════════════════════════════════════
대상: Claude Code (로컬)
작성일: 2026-04-22
버전: v1.0 (핸드오프 v3.4 기준)

이 문서는 docs/SESSION_HANDOFF.md v3.4를 전제로 한다.
Claude Code는 작업 전 반드시 핸드오프 v3.4 전체를 읽을 것.

Week 2는 P2 Phase — 클러스터링·스코어링 + 수동 트리거 봇.
모듈별 단계 리뷰 방식 (§3-I #50).
각 Day 작업은 독립 브랜치로 제출, 세열님 승인 후 main 머지.
════════════════════════════════════════════════════════════════════
-->

# Week 2 작업 명세서 — Day 1~7

## 전제 조건 (Week 1 완료 상태)

작업 시작 전 다음이 모두 충족되어 있어야 한다:

- [x] Day 0~5 Claude Code 구현 완료
- [x] 24시간 무중단 운영 완료 (articles 339,905건)
- [x] `docs/SESSION_HANDOFF.md` v3.4 main에 반영됨
- [x] `CLAUDE.md` 프로젝트 루트 배치 완료
- [x] `csv.field_size_limit(1_000_000)` 패치 적용 (week2-prep → main 머지 완료)
- [x] failed 3건 → pending 리셋 완료
- [x] `docs/observations/day6_7_observation.md` 작성 완료
- [x] `docs/week2_tasks.md` (이 문서) main에 반영됨

Week 1 완료 기준 미충족 시 Day 1 진행 금지.

---

## 공통 규칙

### 절대 준수

1. **핸드오프 v3.4 §3 확정 결정 변경 금지** — 필요 시 세열님에게 보고
2. **핸드오프 §8 Do Not 리스트 30개 엄수**
3. **`cd <REPO_ROOT>/Projects/claude/news_intel && pwd` 박제 블록으로 시작** (§3-I #52)
4. **CLAUDE.md 자동 로드 확인** (§3-I #53)
5. **모든 시간은 UTC 저장** (§3-C #12)
6. **통합 벡터 공간 (§3-D #19), 72h 윈도우 (§3-E #25), 자동 스케줄 금지 (§3-G #38)**
7. **`main` 브랜치 직접 푸시 금지** — Day별 독립 브랜치 (§8 #17)
8. **API 키 하드코딩 금지** — `.env`만, `load_dotenv(override=True)` 필수

### 브랜치 전략

| Day | 브랜치명 |
|---|---|
| Day 1 | `day1-source-dict` |
| Day 2 | `day2-tokenizer-schema` |
| Day 3 | `day3-clusterer` |
| Day 4 | `day4-scorer` |
| Day 5 | `day5-bot-resilience` |

각 Day 완료 시: 브랜치 push → PR/브랜치 상태 → 세열님 리뷰 → 승인 후 main 머지.

### 보고 형식 (각 Day 종료 시)

```
## Day [N] 완료 보고

### 작성된 파일 (신규/수정)
### 핵심 동작 검증 (로그 또는 DB 쿼리 결과)
### 발견된 이슈 (있으면)
### 다음 Day 전제 조건 충족 여부
### 브랜치·커밋 상태
```

---

## Day 1: 매체 권위 사전 실측 기반 확장

### 목표

Week 1 24h 실측 Top 200 매체를 분석하여 매체 권위 사전을 확장한다. 세열님이 직접 리뷰한 리스트를 `sources` 테이블에 적용.

**중요**: Day 1은 Claude Code 자동 진행 금지. **중간에 세열님 리뷰·확정 단계**가 있다.

### 단계

#### Step 1-A (Claude Code): 실측 Top 200 분석 리포트

`scripts/analyze_top_sources.py` 작성:

- articles 테이블에서 Week 1 24h 데이터 기반 Top 200 source_host 집계
- 각 매체별:
  - 24h 건수
  - source_file_type 분포
  - 첫·마지막 기사 시각
  - 대표 URL 샘플 (최근 3개)
  - 기존 sources 테이블 등재 여부
- 출력: `scripts/output/top_sources_analysis.md` 파일
- Markdown 표로 정리 (세열님이 바로 리뷰 가능하게)

추가 자동 분류 힌트:
- URL 도메인 패턴으로 "aggregator/라디오 네트워크/블로그" 추정 (iheart, zazoom 등)
- TLD 기반 국가 언어 추정

**⚠️ authority 점수는 자동 부여 금지.** 분석만. 세열님 리뷰 대기.

#### Step 1-B (세열님): 리뷰 및 확정

세열님이 `top_sources_analysis.md`를 보고:
- 각 매체를 Tier 1~8 중 하나로 분류 (§3-E #31)
- 블랙리스트 매체 선정 (authority=0.01)
- 결과를 `scripts/source_authority_v2.csv` 형식으로 작성:
  ```
  host,display_name,country,language,authority,category,notes
  reuters.com,Reuters,Global,en,1.0,tier1_wire,Global news agency
  iheart.com,iHeartRadio,US,en,0.01,blacklist,"Radio network aggregator, 7554 articles/24h"
  ...
  ```

#### Step 1-C (Claude Code): sources 테이블 적용

`scripts/seed_sources_v2.py` 작성:

- `scripts/source_authority_v2.csv` 읽기
- `INSERT ... ON CONFLICT (host) DO UPDATE SET authority=EXCLUDED.authority, display_name=EXCLUDED.display_name, ...`
- 적용 후 검증 쿼리 실행:
  ```sql
  SELECT 
    CASE 
      WHEN authority >= 0.95 THEN 'tier1_2'
      WHEN authority >= 0.85 THEN 'tier3_4'
      WHEN authority >= 0.65 THEN 'tier5_7'
      WHEN authority >= 0.30 THEN 'low'
      WHEN authority <= 0.05 THEN 'blacklist'
      ELSE 'default'
    END AS tier,
    COUNT(*)
  FROM sources GROUP BY tier ORDER BY tier;
  ```

### Day 1 완료 기준

- [ ] `scripts/analyze_top_sources.py` 작성 및 리포트 생성
- [ ] `scripts/source_authority_v2.csv` 세열님 확정본 존재
- [ ] `scripts/seed_sources_v2.py` 작성 및 적용 성공
- [ ] sources 테이블에 50~70개 능동 등재 + 10~15개 블랙리스트
- [ ] 24h 실측 Top 50 매체 전부 sources 테이블에 분류됨

### Day 1 보고 시 포함

- Tier별 매체 수 분포
- Top 10 매체의 확정된 authority 값
- 블랙리스트 매체 리스트
- 세열님 리뷰가 완료된 `source_authority_v2.csv` 경로

---

## Day 2: 언어 감지 + 토큰화 + 스키마

### 목표

언어 감지·토큰화 공통 모듈 작성 + `clusters`/`daily_top_n` 테이블 생성.

### 파일 1: `shared/lang_detect.py`

- 함수 `detect_language(text: str) -> str`:
  - 반환: ISO 639-1 코드 ('ko', 'en', 'ja', 'de', 'zh', 'other')
  - 1차: Unicode 범위 휴리스틱 (한글 U+AC00~U+D7AF, 일본어 히라가나/카타카나, 중국어 CJK 등)
  - 2차 (필요 시): `langdetect` 라이브러리 fallback
  - 빈 문자열·None은 'other' 반환

- 함수 `detect_languages_batch(texts: list[str]) -> list[str]`:
  - 배치 버전, 효율 위주

- 테스트 케이스 10개 이상

### 파일 2: `shared/tokenizer.py`

- 함수 `tokenize(text: str, language: str = None) -> list[str]`:
  - language 미지정 시 `detect_language()` 호출
  - `'ko'`: Kiwi 사용, 명사·동사 어근·고유명사만 추출 (조사·어미 제거)
  - `'en'`: sklearn 기본 + 뉴스 불용어 사전 (`"said", "report", "according", ...`)
  - `'ja'/'de'/'zh'/'other'`: 공백 + 구두점 기준 단순 토큰화 (MVP)
  - 토큰 정규화: 소문자, 길이 1 토큰 제거, 숫자만 있는 토큰 제거

- 함수 `tokenize_for_clustering(article) -> list[str]`:
  - `article.extras_raw['title']` + persons/organizations/locations의 name 추출
  - 언어 감지 후 토큰화
  - `persons/organizations/locations`의 name은 그대로 토큰으로 사용 (토큰화 안 함, 고유명사 앵커 역할)

### 파일 3: `migrations/002_clusters.sql`

`v3.4 §6` 스키마 그대로 적용:

- `clusters` 테이블
  - 인덱스: importance_score DESC, window_end DESC
- `daily_top_n` 테이블
  - UNIQUE (run_date, rank)

### Day 2 검증

수동 실행:

```bash
python -c "from shared.lang_detect import detect_language; print(detect_language('한국어 테스트'))"
python -c "from shared.lang_detect import detect_language; print(detect_language('English test'))"
python -c "from shared.tokenizer import tokenize; print(tokenize('미 연준이 금리를 동결했다', 'ko'))"
python -c "from shared.tokenizer import tokenize; print(tokenize('The Fed held rates steady', 'en'))"
```

```sql
\dt  -- clusters, daily_top_n 추가 확인
\d clusters
\d daily_top_n
```

### Day 2 완료 기준

- [ ] 3개 파일 작성
- [ ] lang_detect 한/영/일/독/중 샘플 정확 감지
- [ ] tokenize 한국어 조사 제거, 영어 불용어 제거 확인
- [ ] clusters·daily_top_n 테이블 생성 + 인덱스 확인

---

## Day 3: 클러스터링 (72h 윈도우, 통합 벡터 공간)

### 목표

72h 슬라이딩 윈도우로 articles 대상 TF-IDF 벡터화 + 코사인 유사도 기반 클러스터링.

### 파일: `data/clusterer.py`

**구현 상세**:

- 함수 `load_window_articles(window_hours: int = 72) -> list[Article]`:
  - `SELECT * FROM articles WHERE canonical_time > NOW() - INTERVAL '{window_hours} hours'`
  - 한·영 기사만 (language IN ('ko', 'en') OR source_file_type='gkg')
  - 반환: Article dataclass 리스트

- 함수 `vectorize_articles(articles) -> (sparse_matrix, vectorizer)`:
  - 각 기사 → `tokenize_for_clustering()` → 공백 join한 문서 스트링 리스트
  - `TfidfVectorizer(sublinear_tf=True, min_df=2, max_df=0.8, tokenizer=lambda x: x.split())`
  - 반환: TF-IDF sparse 행렬, vectorizer 객체

- 함수 `cluster_articles(matrix, threshold: float = 0.7) -> list[int]`:
  - 옵션 A (권장): `sklearn.cluster.AgglomerativeClustering(metric='cosine', linkage='average', distance_threshold=1-threshold, n_clusters=None)`
  - 옵션 B (대안): 자체 구현 — 각 기사를 순회하며 기존 클러스터와 코사인 유사도 측정, threshold 초과하면 해당 클러스터에 편입, 아니면 새 클러스터
  - 메모리 고려: 120만 건이면 옵션 A는 메모리 초과 가능. 배치 처리 필요할 수 있음
  - 반환: article별 cluster_id 리스트

- 함수 `build_cluster_records(articles, cluster_ids) -> list[Cluster]`:
  - 같은 cluster_id 기사들 그룹핑
  - 각 클러스터의:
    - `canonical_title`: 클러스터 내 최다 매체 권위 기사의 제목 (동점이면 최신)
    - `first_article_time`, `last_article_time`, `persistence_hours`
    - `article_count`, `language_count`, `country_count`
    - `language_distribution`: JSONB `{"en": 32, "ko": 8, ...}`
    - `top_entities`: persons/organizations 빈도 Top 5씩
    - `window_start`, `window_end`

- 함수 `save_clusters(clusters, articles_with_cluster_ids)`:
  - clusters 테이블에 INSERT (기존 레코드 DELETE 후 재삽입 — 매 실행마다 전체 재계산)
  - articles 테이블 `cluster_id` 업데이트 (UPDATE articles SET cluster_id=? WHERE id IN ...)

- 함수 `run_once()`:
  - 위 함수들 순차 호출
  - 로그: `"Clustered N articles into M clusters (window: 72h)"`

- 엔트리포인트: `if __name__ == "__main__": run_once()`

### 성능 가이드

- 120만 건 × 어휘 10,000 TF-IDF는 sklearn으로 Mac M4에서 5~10분 내 처리 가능
- 메모리 초과 시 윈도우 내 기사 수를 제한 (예: 가장 최신 500,000건만)하거나 배치로 처리
- 클러스터 수가 수만 개 되면 정상 (긴 꼬리 지역 뉴스)

### Day 3 검증

수동 실행:

```bash
python -m data.clusterer
```

검증 쿼리:

```sql
-- 클러스터 수, 평균 크기
SELECT COUNT(*) AS cluster_count, AVG(article_count) AS avg_size, MAX(article_count) AS max_size
FROM clusters;

-- Top 10 대형 클러스터
SELECT id, canonical_title, article_count, language_count, persistence_hours
FROM clusters ORDER BY article_count DESC LIMIT 10;

-- 언어 분포 있는 클러스터 샘플
SELECT id, canonical_title, language_distribution 
FROM clusters 
WHERE language_count >= 2 
ORDER BY article_count DESC 
LIMIT 5;

-- 클러스터링 안 된 기사 비율
SELECT 
  COUNT(*) FILTER (WHERE cluster_id IS NULL) AS unclustered,
  COUNT(*) FILTER (WHERE cluster_id IS NOT NULL) AS clustered
FROM articles
WHERE canonical_time > NOW() - INTERVAL '72 hours';
```

### Day 3 완료 기준

- [ ] `data/clusterer.py` 작성
- [ ] 72h 윈도우 대상 클러스터링 성공
- [ ] clusters 테이블에 M개 레코드, articles 테이블 cluster_id 업데이트 확인
- [ ] language_count >= 2 클러스터 존재 (다국어 매칭 작동)
- [ ] 처리 시간 15분 이내, 메모리 2GB 이내

---

## Day 4: 중요도 스코어링

### 목표

각 클러스터의 importance_score 계산 + Top N 기록.

### 파일: `data/scorer.py`

**구현 상세**:

- 함수 `compute_importance(cluster) -> float`:
  ```python
  import math
  
  score = (
      math.log(cluster.article_count + 1)
      + sum_media_authority(cluster)  # Σ 방식 (v3.4 §3-E #28)
      + cluster.persistence_hours / 24.0
  )
  return round(score, 3)
  ```

- 함수 `sum_media_authority(cluster) -> float`:
  - 클러스터에 속한 기사들의 source_host → sources 테이블 조회 → authority 합산
  - 미등재 매체는 기본값 0.5 (§3-E #31 Tier8)
  - SQL:
    ```sql
    SELECT COALESCE(SUM(s.authority), 0) AS total_auth
    FROM articles a
    LEFT JOIN sources s ON s.host = a.source_host
    WHERE a.cluster_id = ?
    ```
  - 미등재 매체는 `COALESCE(s.authority, 0.5)` 처리

- 함수 `update_cluster_scores()`:
  - 모든 clusters 순회하며 importance_score 계산·업데이트
  - 로그: `"Updated N cluster scores (avg=X, max=Y)"`

- 함수 `record_daily_top_n(n: int = 20)`:
  - daily_top_n 테이블에 오늘 날짜(KST 기준) 기록
  - `INSERT ... ON CONFLICT (run_date, rank) DO UPDATE`
  - SELECT clusters ORDER BY importance_score DESC LIMIT n

- 함수 `run_once()`:
  - `update_cluster_scores()` → `record_daily_top_n()`
  - 로그 기록

### Day 4 검증

수동 실행:

```bash
python -m data.scorer
```

검증 쿼리:

```sql
-- Top 20 클러스터의 점수 구성 확인
SELECT 
  c.id,
  c.canonical_title,
  c.article_count,
  c.persistence_hours,
  c.importance_score,
  LEFT(c.language_distribution::text, 80) AS lang_dist
FROM clusters c
ORDER BY c.importance_score DESC
LIMIT 20;

-- daily_top_n 기록 확인
SELECT rank, cluster_id, importance_score 
FROM daily_top_n 
WHERE run_date = CURRENT_DATE 
ORDER BY rank;

-- 매체 authority 적용 샘플 (특정 클러스터)
SELECT a.source_host, COALESCE(s.authority, 0.5) AS auth, COUNT(*) AS cnt
FROM articles a 
LEFT JOIN sources s ON s.host = a.source_host
WHERE a.cluster_id = [Top1 cluster_id]
GROUP BY a.source_host, s.authority
ORDER BY cnt DESC;
```

### Day 4 완료 기준

- [ ] `data/scorer.py` 작성
- [ ] 모든 clusters importance_score 계산됨
- [ ] daily_top_n 테이블 오늘 날짜 레코드 20건
- [ ] Top 1 클러스터의 점수 구성 (cluster_size / sum_auth / persistence) 확인 가능

---

## Day 5: 텔레그램 봇 + 장애 복구 + orchestrator 통합

### 목표

`/digest` 수동 트리거 봇 + 장애 복구 + orchestrator에 clusterer/scorer 1h 주기 통합.

### 파일 1: `bot/sender.py`

- Telegram Bot API 래퍼
- `.env`에서 `TELEGRAM_BOT_TOKEN` 로드 (없으면 명확한 에러)
- 함수 `send_message(chat_id, text, parse_mode='HTML')`:
  - HTTP POST to `https://api.telegram.org/bot{TOKEN}/sendMessage`
  - 메시지 4096자 초과 시 분할 전송
  - 재시도 로직 (429 rate limit 대응)

### 파일 2: `bot/renderer.py`

- 함수 `render_digest(top_clusters: list[Cluster]) -> str`:
  - Top 10 클러스터를 Markdown/HTML 형식으로 렌더링
  - 템플릿:
    ```
    📰 news_intel 다이제스트 — 2026-04-XX 13:45 KST
    (최근 72시간 기준, 총 [N]개 클러스터)
    
    1. [언어 분포 이모지] [canonical_title]
       📊 [article_count]개 기사 · [country_count]개국 · [persistence_hours]h 지속
       🎯 importance: [score]
       🔗 [대표 URL]
    
    2. ...
    ```
  - 언어 분포 이모지 매핑:
    - en 다수: 🌐
    - ko 다수: 🇰🇷  
    - 다국어 (>=2): 🌍

- 함수 `render_digest_html(top_clusters) -> str`:
  - Telegram HTML 파싱용

### 파일 3: `bot/commands.py`

- Telegram bot 폴링 루프 (python-telegram-bot 라이브러리 사용)
- 명령어 등록:
  - `/digest`: 현재 clusters 테이블의 Top 10 조회 → renderer → sender
  - `/start`: 간단 안내 메시지
  - `/help`: 명령어 목록

- 세열님 chat_id만 허용 (보안)
  - `.env`에 `TELEGRAM_ALLOWED_CHAT_IDS` (쉼표 구분 리스트)
  - 다른 사용자 요청 무시

**주의**: `bot/scheduler.py` **작성 금지**. 자동 발송 폐기 (§8 #27).

### 파일 4: `data/cleanup.py` 보강 — 외장 SSD 마운트 체크

`cleanup.py` 또는 신규 `shared/health.py`:

- 함수 `check_ssd_mount() -> bool`:
  - `Path(NEWS_DATA_ROOT).exists()` 체크
  - False면 logger.error + system_log에 기록
  - 5분 대기 후 재체크하는 로직은 호출자 쪽에서 처리

### 파일 5: `data/orchestrator.py` 보강

기존 스케줄에 추가:

```python
# 기존
schedule.every(5).minutes.do(run_watcher)
schedule.every(1).minutes.do(run_downloader)
schedule.every(1).minutes.do(run_parser)

# Week 2 Day 5 추가
schedule.every(1).hours.do(run_clusterer_and_scorer)

def run_clusterer_and_scorer():
    # SSD 마운트 체크
    if not check_ssd_mount():
        logger.warning("SSD not mounted, skipping clustering")
        return
    try:
        clusterer.run_once()
        scorer.run_once()
    except Exception as e:
        logger.error(f"Clustering/scoring failed: {e}", exc_info=True)
```

### 파일 6: launchd plist (`scripts/launchd/com.newsintel.orchestrator.plist`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.newsintel.orchestrator</string>
    
    <key>ProgramArguments</key>
    <array>
        <string><REPO_ROOT>/venv/bin/python</string>
        <string>-m</string>
        <string>data.orchestrator</string>
    </array>
    
    <key>WorkingDirectory</key>
    <string><REPO_ROOT>/Projects/claude/news_intel</string>
    
    <key>KeepAlive</key>
    <true/>
    
    <key>RunAtLoad</key>
    <true/>
    
    <key>StandardOutPath</key>
    <string><REPO_ROOT>/Projects/claude/news_intel/logs/launchd_stdout.log</string>
    
    <key>StandardErrorPath</key>
    <string><REPO_ROOT>/Projects/claude/news_intel/logs/launchd_stderr.log</string>
</dict>
</plist>
```

설치 가이드를 `docs/operations.md`에 추가:

```markdown
## launchd 자동 재시작 설정 (선택)

cp scripts/launchd/com.newsintel.orchestrator.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.newsintel.orchestrator.plist

중지 (재부팅 시 자동 시작 비활성):
launchctl unload ~/Library/LaunchAgents/com.newsintel.orchestrator.plist
```

### Day 5 검증

#### 선행: Telegram Bot 생성 (세열님 수동)

- @BotFather에서 봇 생성 → TOKEN 받기
- `.env`에 `TELEGRAM_BOT_TOKEN=...` 추가
- 세열님 chat_id 확인 (@userinfobot 등 사용) → `.env`에 `TELEGRAM_ALLOWED_CHAT_IDS=...` 추가

#### 봇 동작 테스트

```bash
python -m bot.commands  # 폴링 시작
```

세열님이 Telegram에서 `/digest` 송신 → Top 10 메시지 수신 확인.

#### orchestrator 테스트

```bash
python -m data.orchestrator
# 1시간 이상 실행
```

검증:
- 1시간 주기 clusterer 실행 로그
- clusters 테이블 갱신 시각 확인
- SSD 마운트 해제 시뮬레이션 (외장 SSD 잠시 뺐다 연결) → "SSD not mounted" 경고 후 재개

### Day 5 완료 기준

- [ ] 5개 파일 작성 (bot 3개 + cleanup/health 1개 + orchestrator 수정 + plist)
- [ ] `/digest` 명령 응답 작동
- [ ] 허용되지 않은 chat_id 요청 무시 확인
- [ ] orchestrator 1h 주기 clusterer/scorer 실행 로그
- [ ] SSD 마운트 체크 경고 동작 확인
- [ ] launchd plist 파일 작성 (설치 안 해도 됨)
- [ ] `docs/operations.md`에 launchd 설정 가이드 추가

---

## Day 6~7: 수동 품질 검증 (세열님 담당)

### 목표

세열님이 직접 `/digest` 호출하며 다이제스트 품질 평가 + Week 3 튜닝 포인트 식별.

### 체크 항목

| 항목 | 확인 방법 |
|---|---|
| Top 10에 국제 주요 이슈 포함 | 주관 판단 — 금리, 정상회담, 전쟁, 대형 경제지표 등 |
| 지역·가십 노이즈 비율 | iheart·지역 뉴스가 Top에 오르는가 |
| 블랙리스트 authority 0.01 유효성 | 블랙리스트 매체 대형 복제 패턴이 Top 역전 시켰는가 |
| 다국어 매칭 작동 | `language_count >= 2` 클러스터가 Top 10에 포함되는가 |
| 중요도 공식 재조정 필요성 | Σ / MAX / threshold 조정 중 어떤 방향 |
| 응답 시간 | `/digest` 치고 몇 초 안에 응답하는가 |

### 블랙리스트 0.01 vs 0.005 판단

- Top 10에 iheart·zazoom 지역 뉴스가 2개 이상 올라오면 → 0.005로 하향 권장
- 그렇지 않으면 → 0.01 유지

판단 후 SQL 실행:
```sql
UPDATE sources SET authority=0.005 WHERE authority=0.01;
-- scorer.run_once() 재실행하여 반영
```

### Week 2 완료 기준

- [ ] `/digest` 5회 이상 호출하며 Top 10 샘플 수집
- [ ] 주관 품질 평가 메모 작성 (`docs/observations/week2_day6_7_eval.md`)
- [ ] 블랙리스트 0.01 vs 0.005 결정
- [ ] 중요도 공식 조정 필요 여부 결정
- [ ] Week 3 튜닝 작업 리스트 도출

---

## 예외 처리·피드백 루프

### Claude Code가 독단 진행 금지 상황

- 핸드오프 v3.4 §3 확정 결정과 충돌 요구사항
- 메모리/성능 상한 초과 구조적 문제
- 스키마 변경 필요 판단
- 클러스터링 성능이 2GB/15분 상한 초과
- 매체 권위 사전 Tier 분류 판단 모호

이 경우 세열님 보고 + 대안 2~3개 제시.

### 세열님 보고 포함 내용

- 구체 에러/관찰
- 영향 받는 파일·테이블
- 가능한 해결 방향과 각 trade-off
- 제안하는 선택과 근거

---

**문서 상태**: v1.0 (2026-04-22 작성, 핸드오프 v3.4 기준)
**다음 업데이트**: Week 2 완료 후 회고 또는 Week 3 명세 작성 시
