<!--
════════════════════════════════════════════════════════════════════
Week 1 작업 명세서 (Day 3~5)
════════════════════════════════════════════════════════════════════
대상: Claude Code (로컬)
작성일: 2026-04-21
버전: v1.0 (핸드오프 v3.2 기준)

이 문서는 docs/SESSION_HANDOFF.md v3.2를 전제로 한다.
Claude Code는 이 문서 작업 전 반드시 핸드오프 v3.2 전체를 읽을 것.

Day 3~5는 "모듈별 단계 리뷰" 방식 (§3-I #43).
각 Day 작업은 독립 브랜치로 제출하고, 세열님 승인 후 main에 머지한다.
세열님 승인 없이 다음 Day로 진행 금지.
════════════════════════════════════════════════════════════════════
-->

# Week 1 작업 명세서 — Day 3~5

## 전제 조건 (Day 0~2 완료 상태 확인)

작업 시작 전 다음이 모두 충족되어 있어야 한다:

- [x] Day 0: Docker Postgres 가동, Python venv + 11개 패키지, Kiwi 검증
- [x] Day 1: GDELT 구조 실측 확증, `translation` 파이프라인 필수 발견
- [x] Day 2: `migrations/001_init.sql` 적용, 4개 테이블 생성, 100행 수동 insert 검증

Day 2 성공 조건이 충족되지 않았다면 Day 3 진행 금지. 세열님에게 보고.

---

## 공통 규칙

### 절대 준수

1. **핸드오프 v3.2 §3 확정 결정 변경 금지** — 필요 시 세열님에게 보고
2. **핸드오프 §8 Do Not 리스트 22개 전부 엄수**
3. **본체 GKG와 translation GKG 양쪽 처리 필수** (v3.2 핵심)
4. **모든 시간은 UTC 저장** (§3-C #12)
5. **단일 파일 > 500MB 메모리 로드 금지** — 스트리밍 파싱 필수 (§8 #12)
6. **API 키 하드코딩 금지** — `.env`만 사용 (§8 #3)
7. **main 브랜치 직접 푸시 금지** — Day별 독립 브랜치 사용 (§8 #16)

### 브랜치 전략

| Day | 브랜치명 |
|---|---|
| Day 3 | `day3-watcher-downloader` |
| Day 4 | `day4-parser` |
| Day 5 | `day5-orchestrator` |

각 Day 작업 완료 시:
1. 해당 브랜치에 커밋·push
2. Pull Request 생성 (또는 브랜치 상태로 세열님 리뷰 요청)
3. 세열님 승인 후 `main`에 머지

### 보고 형식 (각 Day 종료 시)

다음 형식으로 세열님에게 보고:

```
## Day [N] 완료 보고

### 작성된 파일
- 파일 목록 (신규/수정)

### 핵심 동작 검증
- 각 모듈의 주요 기능이 작동함을 보인 증거
- 예: 로그 출력 스크린샷, DB 쿼리 결과

### 발견된 이슈
- 설계와 다르게 구현한 부분이 있으면 이유와 함께 보고
- 성능 문제, 예외 케이스 등

### 다음 Day 전제 조건
- 이 Day가 성공해야 다음 Day 진행 가능한 상태인지 확인

### 브랜치·커밋 상태
- 브랜치명, 최근 커밋 3개
```

---

## Day 3: masterfile_watcher + downloader

### 목표
GDELT의 두 엔드포인트(`lastupdate.txt`, `lastupdate-translation.txt`)를 폴링하여 새 파일을 감지하고, `gdelt_files` 테이블에 기록한다. 승인된 파일을 순차 다운로드해 외장 SSD에 저장한다.

### 환경 변수 (`.env` 활용)

```
DATABASE_URL=postgresql://news:...@localhost:5432/news_intel
NEWS_DATA_ROOT=/Volumes/AIDRIVE/news_intel
GDELT_LASTUPDATE_URL=http://data.gdeltproject.org/gdeltv2/lastupdate.txt
GDELT_LASTUPDATE_TRANSLATION_URL=http://data.gdeltproject.org/gdeltv2/lastupdate-translation.txt
GDELT_POLL_INTERVAL_SECONDS=300
LOG_LEVEL=INFO
```

`.env`에 누락된 키는 추가하되, 기존 키는 수정하지 말 것.

### 파일 1: `shared/config.py`

- `python-dotenv`로 `.env` 로드
- **필수**: `dotenv.load_dotenv(PROJECT_ROOT / ".env", override=True)` 사용
  - `override=True`는 필수. Day 2 실측에서 셸에 다른 프로젝트의 `DATABASE_URL`이
    export되어 있을 때 `.env` 값이 무시되는 환경변수 오염 문제가 확증됨.
    `override=True`로 프로젝트 `.env`가 항상 셸 환경변수보다 우선하도록 보장.
  - 참고: `scripts/manual_insert_test.py`에도 동일 패턴 적용 완료 (Day 2 검증 시 수정)
- 모든 환경 변수를 상수로 노출:
  - `DATABASE_URL`, `NEWS_DATA_ROOT`(Path 객체), `GDELT_LASTUPDATE_URL`, `GDELT_LASTUPDATE_TRANSLATION_URL`, `GDELT_POLL_INTERVAL_SECONDS`(int), `LOG_LEVEL`
- `NEWS_DATA_ROOT / "data" / "raw" / "gkg"` 와 `NEWS_DATA_ROOT / "data" / "raw" / "translation_gkg"` 경로 상수도 제공
- 로드 시점에 필수 키 누락이면 명확한 에러 메시지로 실패

### 파일 2: `shared/db.py`

- `psycopg2.pool.ThreadedConnectionPool`로 커넥션 풀 (min=1, max=5)
- 컨텍스트 매니저 `@contextmanager def get_conn()` 제공 (자동 commit/rollback)
- 풀 초기화는 지연(lazy) 방식 권장 (import 시점에 Postgres 미기동이어도 모듈 로드는 되도록)

### 파일 3: `shared/logging_util.py`

- Python `logging` 기반
- 콘솔 + 파일 핸들러 (파일: `~/Projects/claude/news_intel/logs/pipeline_YYYYMMDD.log`, 일별 rotation)
- 포맷: `%(asctime)s | %(levelname)s | %(name)s | %(message)s` (UTC)
- 로그 레벨은 `config.LOG_LEVEL` 기반
- `system_log` 테이블에도 INFO 이상 기록하는 핸들러 함수 제공 (선택적 사용)

### 파일 4: `data/masterfile_watcher.py`

**책임**: 두 `lastupdate` 엔드포인트 폴링 → `gdelt_files` 테이블에 신규 파일 등록

**구현 상세**:

- 함수 `fetch_latest_files() -> list[dict]`:
  - `GDELT_LASTUPDATE_URL`과 `GDELT_LASTUPDATE_TRANSLATION_URL` 각각 requests.get (timeout=30s)
  - 각 응답은 3줄 (events/mentions/gkg)
  - 본체 응답의 gkg 라인만 추출 → `file_type='gkg'`
  - translation 응답의 gkg 라인만 추출 → `file_type='translation_gkg'`
  - events/mentions는 MVP에서 무시 (Do Not §8 #11, Phase P4 이후)
  - 반환: `[{'file_url': ..., 'file_type': ..., 'file_timestamp': datetime, 'file_size': int, 'md5_expected': str}, ...]`
  - URL에서 타임스탬프(YYYYMMDDHHMMSS) 추출 → UTC datetime
  
- 함수 `register_new_files(files: list[dict]) -> int`:
  - `gdelt_files` 테이블에 `INSERT ... ON CONFLICT (file_url) DO NOTHING`
  - 새로 삽입된 행 수 반환

- 함수 `run_once()`:
  - `fetch_latest_files()` → `register_new_files()`
  - 실패 시 system_log에 WARN 기록, 예외 던지지 말고 0 반환
  - 성공 시 INFO 로그: `"Registered N new files (gkg=X, translation_gkg=Y)"`

- 엔트리포인트: `if __name__ == "__main__": run_once()` (일회성 실행도 가능하도록)

### 파일 5: `data/downloader.py`

**책임**: `gdelt_files` 테이블에서 `status='pending'` 파일을 가져와 외장 SSD에 다운로드, `status='downloaded'`로 갱신

**구현 상세**:

- 함수 `download_one(file_row) -> bool`:
  - `file_type`에 따라 저장 경로 결정:
    - `gkg` → `{NEWS_DATA_ROOT}/data/raw/gkg/`
    - `translation_gkg` → `{NEWS_DATA_ROOT}/data/raw/translation_gkg/`
  - 파일명: URL의 마지막 segment (예: `20260421024500.gkg.csv.zip`)
  - requests stream=True로 청크 다운로드 (chunk_size=8192)
  - 다운로드 후 MD5 검증 (expected와 비교). 불일치 시 파일 삭제 + status='failed'
  - 성공: status='downloaded', downloaded_at=NOW()
  - 실패: retry_count += 1, error_message 기록. retry_count ≥ 3이면 status='failed', 아니면 'pending' 유지

- 함수 `run_once(max_downloads: int = 5)`:
  - `SELECT ... WHERE status='pending' ORDER BY file_timestamp DESC LIMIT {max_downloads}`
  - 각 파일을 `download_one()`으로 처리 (순차, 동시 다운로드 금지)
  - 전체 결과 로그: `"Downloaded N files (success=X, failed=Y)"`

- 엔트리포인트: `if __name__ == "__main__": run_once()`

### Day 3 검증

작업 완료 후 수동 실행:

```bash
cd ~/Projects/claude/news_intel
source venv/bin/activate

python -m data.masterfile_watcher
python -m data.downloader
```

검증 쿼리:

```sql
-- 신규 파일 등록 확인
SELECT file_type, COUNT(*) FROM gdelt_files GROUP BY file_type;

-- 다운로드 상태
SELECT status, COUNT(*) FROM gdelt_files GROUP BY status;

-- 최근 처리 5건
SELECT file_url, file_type, status, downloaded_at 
FROM gdelt_files ORDER BY created_at DESC LIMIT 5;
```

외장 SSD 확인:

```bash
ls -la /Volumes/AIDRIVE/news_intel/data/raw/gkg/ | head
ls -la /Volumes/AIDRIVE/news_intel/data/raw/translation_gkg/ | head
```

### Day 3 완료 기준

- [ ] 4개 파일 작성 (`shared/config.py`, `shared/db.py`, `shared/logging_util.py`, `data/masterfile_watcher.py`, `data/downloader.py`)
- [ ] `masterfile_watcher` 실행 시 gdelt_files 테이블에 본체+translation 각각 최소 1건 등록
- [ ] `downloader` 실행 시 외장 SSD의 `gkg/`와 `translation_gkg/` 각각에 최소 1개 파일 저장
- [ ] MD5 검증 통과
- [ ] 로그 파일 생성 (`logs/pipeline_YYYYMMDD.log`)
- [ ] system_log 테이블에 INFO 레벨 로그 있음
- [ ] 중복 파일 재등록되지 않음 (ON CONFLICT 작동)

---

## Day 4: gkg_parser + url_utils + xml_utils

### 목표
`status='downloaded'` 파일을 스트리밍 파싱하여 `articles` 테이블에 적재한다. 본체와 translation 양쪽 공통 파서로 처리하되 `source_file_type` 필드로 구분한다.

### 파일 1: `shared/url_utils.py`

- 함수 `normalize_url(raw_url: str) -> str`:
  - 쿼리스트링 제거: `utm_*`, `fbclid`, `ref`, `source`, `ref_src`, `mc_cid`, `_ga`
  - 프래그먼트(`#...`) 제거
  - `/amp/` 제거 (끝 또는 경로 내 둘 다)
  - 트레일링 슬래시 통일 (없는 쪽으로)
  - `http` → `https`
  - 정규화 실패 시 raw_url 원본 반환 (예외 던지지 않음)

- 함수 `extract_host(url: str) -> str`:
  - `tldextract`로 `{domain}.{suffix}` 반환 (예: `reuters.com`)
  - 서브도메인 포함이 필요한 경우만 (예: `world.kbs.co.kr`은 별도 매체) 예외 처리 규칙 명시

- 테스트 케이스 (최소 10개, `if __name__ == "__main__":`로 실행 가능):
  ```python
  cases = [
      ("https://www.reuters.com/article?utm_source=rss#top", "https://www.reuters.com/article"),
      ("http://example.com/amp/news/", "https://example.com/news"),
      ...
  ]
  ```

### 파일 2: `shared/xml_utils.py`

- 함수 `parse_extras_xml(xml_str: str) -> dict`:
  - V2ExtrasXML는 루트 태그 없는 XML 조각. 정규식 기반 추출이 더 견고
  - 추출 대상 태그:
    - `PAGE_PRECISEPUBTIMESTAMP` → datetime (UTC) 또는 None
    - `PAGE_AUTHORS` → str 또는 None
    - `PAGE_LINKS` → list[str] (세미콜론 split)
    - `PAGE_TITLE` → str 또는 None
    - `PAGE_ALTURL_AMP` → str 또는 None
  - 반환 dict: `{"precise_pub_timestamp": datetime|None, "authors": str|None, "links": list, "title": str|None, "alt_url_amp": str|None, "raw_xml": str}`
  - 파싱 실패한 필드는 None, 전체 raw_xml은 항상 보존

- 타임스탬프 파싱: `PAGE_PRECISEPUBTIMESTAMP`는 `YYYYMMDDHHMMSS` 14자리. 형식 불일치면 None

### 파일 3: `shared/time_utils.py`

- 함수 `parse_gdelt_date(yyyymmddhhmmss: str) -> datetime`:
  - 14자리 문자열 → UTC datetime
  - 형식 오류 시 `ValueError` 발생

- 함수 `compute_canonical_time(gdelt_added_at: datetime, precise_pub_timestamp: datetime | None) -> tuple[datetime, float]`:
  - precise 있으면 `(precise, 1.0)`
  - 없으면 `(gdelt_added_at, 0.8)`
  - 반환: (canonical_time, time_confidence)

- 함수 `extract_url_path_date(url: str) -> date | None`:
  - URL path에서 날짜 추출 시도: `/YYYY/MM/DD/`, `/YYYYMMDD/`, `/YYYY-MM-DD/` 패턴
  - 없으면 None

### 파일 4: `data/gkg_parser.py`

**책임**: 다운로드된 GKG zip 파일을 스트리밍 파싱하여 articles 적재

**핵심 원칙**:
- **스트리밍 필수**: `zipfile.ZipFile` → `TextIOWrapper`로 한 줄씩 읽기. pandas.read_csv로 전체 로드 금지
- **메모리 상한**: 단일 파일 처리 중 Python 프로세스 메모리 500MB 미만 유지
- **한 행 파싱 실패가 파일 전체 중단시키지 않음**: try/except로 개별 행 skip + system_log 기록

**구현 상세**:

- GKG 27개 컬럼 매핑 상수 `GKG_COLS` 정의 (v3.2 문서 참조)

- 함수 `parse_and_load(file_row) -> tuple[int, int]`:
  - file_row는 gdelt_files 한 행 (status='downloaded')
  - 로컬 경로 구성: `{NEWS_DATA_ROOT}/data/raw/{file_type}/{filename}`
  - zipfile로 압축 해제 스트림 오픈
  - `csv.reader(delimiter='\t')` 한 행씩 처리
  - 각 행 매핑:
    ```
    gkg_record_id    = row[0]
    gdelt_added_at   = parse_gdelt_date(row[1])
    source_host      = row[3] if row[3] else extract_host(row[4])
    url_raw          = row[4]
    url              = normalize_url(url_raw)
    
    extras           = parse_extras_xml(row[26])
    precise          = extras["precise_pub_timestamp"]
    canonical_time, time_confidence = compute_canonical_time(gdelt_added_at, precise)
    
    url_path_date    = extract_url_path_date(url)
    
    # V1.5Tone (col 15): "tone,pos,neg,pol,act,self,wc" 쉼표 구분
    tone_values = row[15].split(",") if row[15] else []
    tone = float(tone_values[0]) if tone_values and tone_values[0] else None
    
    # JSONB 필드들 (세미콜론 split, 각 엔티티는 쉼표)
    themes        = parse_themes(row[8])         # "theme1,off1;theme2,off2" → [{"theme":..., "offset":...}]
    locations     = parse_locations(row[10])     # 더 복잡한 파이프 구분. FIPS, ADM1, lat, lon 포함
    persons       = parse_entities(row[12])
    organizations = parse_entities(row[14])
    
    # source_file_type은 file_row.file_type 그대로 ('gkg' 또는 'translation_gkg')
    source_file_type = file_row.file_type
    
    # language: MVP는 본체=en, translation=NULL로 두고 P3에서 실제 감지
    language = "en" if source_file_type == "gkg" else None
    ```
  - INSERT ... ON CONFLICT (url) DO NOTHING, RETURNING id
  - 개별 행 실패 시 skip 카운트 증가, system_log에 WARN
  - 반환: `(inserted, skipped)`

- 파일 완료 후:
  - gdelt_files.status = 'parsed'
  - gdelt_files.parsed_at = NOW()
  - gdelt_files.row_count = inserted
  - 로그: `"Parsed file {filename}: inserted={inserted}, skipped={skipped}"`

- 함수 `run_once(max_files: int = 3)`:
  - `SELECT ... WHERE status='downloaded' ORDER BY file_timestamp ASC LIMIT {max_files}`
  - 각 파일을 parse_and_load 호출
  - 전체 요약 로그

- 엔트리포인트

### Day 4 검증

수동 실행:

```bash
cd ~/Projects/claude/news_intel
source venv/bin/activate
python -m data.gkg_parser
```

검증 쿼리:

```sql
-- 파일 처리 상태
SELECT status, COUNT(*) FROM gdelt_files GROUP BY status;

-- 파이프라인별 적재량
SELECT source_file_type, COUNT(*) FROM articles GROUP BY source_file_type;

-- time_confidence 분포
SELECT source_file_type, time_confidence, COUNT(*) 
FROM articles GROUP BY source_file_type, time_confidence
ORDER BY source_file_type, time_confidence DESC;

-- 한국·영어 매체 샘플
SELECT source_host, COUNT(*) 
FROM articles 
WHERE source_host ~ '\.(com|kr|co\.kr|uk|de|jp|cn)$'
GROUP BY source_host 
ORDER BY COUNT(*) DESC LIMIT 30;

-- JSONB 구조 확인
SELECT url, persons->0, organizations->0, themes->0 
FROM articles 
WHERE persons IS NOT NULL AND jsonb_array_length(persons) > 0
LIMIT 3;

-- URL 중복 체크 (0이어야 정상)
SELECT url, COUNT(*) FROM articles GROUP BY url HAVING COUNT(*) > 1;

-- 한글 깨짐 체크
SELECT url, article_authors FROM articles 
WHERE source_host LIKE '%.kr' OR url LIKE '%korea%' LIMIT 10;
```

### Day 4 완료 기준

- [ ] 4개 파일 작성 (`shared/url_utils.py`, `shared/xml_utils.py`, `shared/time_utils.py`, `data/gkg_parser.py`)
- [ ] url_utils 테스트 10개 전부 통과
- [ ] 본체 GKG 파일 1개와 translation GKG 파일 1개 파싱 성공 (articles 테이블에 source_file_type 양쪽 모두 존재)
- [ ] 한글 인코딩 깨짐 없음
- [ ] URL 중복 0건
- [ ] precise_pub_timestamp 파싱 비율: 본체 60~70%, translation 25~35% 범위
- [ ] 파싱 중 파일 전체 중단 없음 (개별 행 실패 허용)
- [ ] 메모리 사용 500MB 이하 확인 (`docker stats` + `ps aux | grep python`)

---

## Day 5: orchestrator + cleanup + 24시간 운영 준비

### 목표
Day 3·4 모듈을 스케줄링으로 묶어 **24시간 무중단 운영**이 가능한 상태로 완성한다. 90일 롤링 삭제는 구조만 만들고 활성화는 Week 4 이후.

### 파일 1: `data/cleanup.py`

**책임**: 90일 초과 데이터 삭제 (구조만, 활성화는 나중)

- 함수 `cleanup_old_files(days: int = 90, dry_run: bool = True)`:
  - 외장 SSD `raw/gkg/`와 `raw/translation_gkg/`에서 mtime이 90일 이상 된 파일 목록 반환
  - dry_run=True면 로그만, False면 실제 unlink
  - 삭제 전 gdelt_files.status='parsed'인지 확인 (미처리 파일은 보존)

- 함수 `cleanup_old_articles(days: int = 90, dry_run: bool = True)`:
  - `DELETE FROM articles WHERE canonical_time < NOW() - INTERVAL '{days} days' AND cluster_id IS NOT NULL`
  - cluster_id IS NULL인 행은 보존 (아직 클러스터링 안 됨)
  - dry_run=True면 COUNT만

**주의**: Day 5에서는 `dry_run=True`가 기본값. 자동 활성화 금지. Week 4에서 세열님이 활성화 결정.

### 파일 2: `data/orchestrator.py`

**책임**: schedule 라이브러리로 Day 3·4 모듈을 주기 실행

**구현 상세**:

```python
import schedule
import time
import signal
import sys

from data import masterfile_watcher, downloader, gkg_parser
from shared.logging_util import get_logger

logger = get_logger(__name__)
stop_flag = False

def handle_shutdown(signum, frame):
    global stop_flag
    logger.info(f"Received signal {signum}, shutting down gracefully...")
    stop_flag = True

signal.signal(signal.SIGINT, handle_shutdown)
signal.signal(signal.SIGTERM, handle_shutdown)

def run_watcher():
    try:
        masterfile_watcher.run_once()
    except Exception as e:
        logger.error(f"Watcher failed: {e}", exc_info=True)

def run_downloader():
    try:
        downloader.run_once(max_downloads=5)
    except Exception as e:
        logger.error(f"Downloader failed: {e}", exc_info=True)

def run_parser():
    try:
        gkg_parser.run_once(max_files=3)
    except Exception as e:
        logger.error(f"Parser failed: {e}", exc_info=True)

def main():
    logger.info("Orchestrator starting...")
    
    # 주기 설정 (§3-A 실측: GDELT 지연 일상적이므로 5분은 충분)
    schedule.every(5).minutes.do(run_watcher)
    schedule.every(1).minutes.do(run_downloader)
    schedule.every(1).minutes.do(run_parser)
    
    # 시작 시 즉시 1회 실행
    run_watcher()
    run_downloader()
    run_parser()
    
    logger.info("Entering main loop (Ctrl+C to stop)")
    while not stop_flag:
        schedule.run_pending()
        time.sleep(1)
    
    logger.info("Orchestrator stopped.")

if __name__ == "__main__":
    main()
```

### 파일 3: 운영 가이드 문서 `docs/operations.md`

간단한 운영 매뉴얼 (약 1~2페이지):

```markdown
# news_intel 운영 가이드

## 실행
```bash
cd ~/Projects/claude/news_intel
source venv/bin/activate
python -m data.orchestrator
```

## 백그라운드 실행 (선택)
nohup python -m data.orchestrator > /dev/null 2>&1 &

## 상태 확인
- 로그: tail -f logs/pipeline_YYYYMMDD.log
- DB: docker exec -it news_pg psql -U news -d news_intel -c "SELECT ..."

## 중지
- Foreground: Ctrl+C (graceful shutdown)
- Background: kill PID

## 주요 쿼리
(§11의 관찰 쿼리 세트 복붙)
```

### Day 5 검증 — 24시간 운영 예행 연습

Day 5 당일에는 전체 24시간 운영은 불가. 대신:

1. **30분 운영 테스트**:
   ```bash
   python -m data.orchestrator
   # 30분 대기
   # Ctrl+C
   ```

2. **graceful shutdown 확인**:
   - Ctrl+C 후 "Orchestrator stopped." 로그 확인
   - 프로세스 잔존 없음 확인

3. **운영 중 쿼리**:
   ```sql
   -- 주기적 실행 증거
   SELECT component, COUNT(*) FROM system_log 
   WHERE created_at > NOW() - INTERVAL '30 minutes'
   GROUP BY component;
   
   -- 진행 중 파일 처리
   SELECT status, COUNT(*) FROM gdelt_files
   WHERE created_at > NOW() - INTERVAL '30 minutes'
   GROUP BY status;
   ```

### Day 5 완료 기준

- [ ] 3개 파일 작성 (`data/cleanup.py`, `data/orchestrator.py`, `docs/operations.md`)
- [ ] orchestrator 30분 실행 중 에러 없음
- [ ] graceful shutdown 작동 (Ctrl+C → "stopped" 로그)
- [ ] 30분 내 watcher는 6회 이상, downloader·parser는 30회 이상 실행됨
- [ ] cleanup 모듈은 dry_run=True 기본값 확인
- [ ] `docs/operations.md` 작성 완료

---

## Week 1 최종 완료 기준 (Day 6~7 세열님 담당)

Day 5까지 완료 후, 세열님이 24시간 연속 운영한다. 핸드오프 v3.2 §11 P1 완료 기준:

- [ ] 24시간 동안 발행된 모든 GDELT 파일을 누락 없이 처리 (본체+translation 양쪽)
- [ ] articles 테이블 누적 행 수 25,000+
- [ ] 중복 URL 0건
- [ ] 한국어 주요 매체 20개 중 8개 이상 수집 확인
- [ ] 영어 주요 매체 15개 중 12개 이상 수집 확인
- [ ] 파일 처리 실패율 5% 미만
- [ ] 메모리 1GB 미만 상시 유지
- [ ] ERROR 로그 없음 (파일 부재 경고 제외)
- [ ] time_confidence=1.0 비율 40% 이상
- [ ] source_file_type 분포 gkg 약 55% / translation_gkg 약 45%

세열님은 `docs/observations/day6_7_observation.md`로 결과 정리하여 다음 Claude 세션에 공유한다.

---

## 예외 처리·피드백 루프

### Claude Code가 작업 중 막힐 때

다음 상황에서는 **독단 진행 금지**, 세열님에게 보고:

- 핸드오프 v3.2 §3 확정 결정과 충돌하는 요구사항 발견
- GDELT 파일 구조가 v3.2 §6 스키마와 근본적으로 불일치
- 메모리/성능 상한을 초과하는 구조적 문제
- 스키마 변경이 필요하다고 판단되는 경우

### 세열님에게 보고할 때 포함할 것

- 구체적 에러 메시지 또는 관찰
- 영향 받는 파일·테이블
- 가능한 해결 방향 2~3개와 각각의 trade-off
- 제안하는 선택과 근거

---

**문서 상태**: v1.0 (2026-04-21 작성)
**다음 업데이트 시점**: Week 1 완료 후 회고 또는 Week 2 명세 작성 시
