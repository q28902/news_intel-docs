# news_intel 운영 가이드

## 실행

```bash
cd ~/Projects/claude/news_intel
source ~/venv/bin/activate
python -m data.orchestrator
```

## 백그라운드 실행 (선택)

```bash
nohup python -m data.orchestrator > /dev/null 2>&1 &
echo $!  # PID 기록
```

## 중지

- **Foreground**: `Ctrl+C` (graceful shutdown → "Orchestrator stopped." 로그)
- **Background**: `kill <PID>` (SIGTERM → graceful shutdown)

## 로그 확인

```bash
# 실시간 스트림
tail -f ~/Projects/claude/news_intel/logs/pipeline_$(date -u +%Y%m%d).log

# DB 시스템 로그
docker exec -it news_pg psql -U news -d news_intel \
  -c "SELECT level, component, message, created_at FROM system_log ORDER BY created_at DESC LIMIT 20;"
```

## 메모리 사용량 측정

### 즉시 확인 (1회)

```bash
# Python 프로세스 RSS (MB)
ps aux | grep "data.orchestrator" | grep -v grep | awk '{print $6/1024 " MB"}'

# 상세 (PID, RSS, %MEM, 경과시간)
ps -p $(pgrep -f "data.orchestrator") -o pid,rss,%mem,etime 2>/dev/null
```

### 연속 모니터링 (30분 운영 테스트용)

```bash
# 10초 간격으로 RSS 기록 → CSV
while true; do
  echo "$(date -u +%H:%M:%S),$(ps -p $(pgrep -f data.orchestrator) -o rss= 2>/dev/null || echo 0)" \
    >> /tmp/mem_log.csv
  sleep 10
done

# 결과 확인: peak / average
awk -F, 'NR>0{sum+=$2; if($2>max)max=$2; n++} END{print "peak="max/1024"MB avg="sum/n/1024"MB"}' /tmp/mem_log.csv
```

### 기준값

| 항목 | 상한 | Day 4 실측 | Day 5 실측 |
|------|------|-----------|-----------|
| Python RSS (파싱 중) | 500 MB | 51 MB | ~50 MB |
| Python RSS (대기 중) | — | — | ~50 MB |
| Postgres (docker stats) | 500 MB | — | — |

500MB 넘으면 비정상 → `gkg_parser.py` 스트리밍 파싱 확인.

## 주요 상태 확인 쿼리

```sql
-- 파일 처리 상태 요약
SELECT file_type, status, COUNT(*) FROM gdelt_files GROUP BY file_type, status ORDER BY file_type;

-- 파이프라인별 적재량
SELECT source_file_type, COUNT(*) FROM articles GROUP BY source_file_type;

-- time_confidence 분포
SELECT source_file_type, time_confidence, COUNT(*),
       ROUND(100.0*COUNT(*)/(SUM(COUNT(*)) OVER (PARTITION BY source_file_type)),1) as pct
FROM articles GROUP BY source_file_type, time_confidence
ORDER BY source_file_type, time_confidence DESC;

-- 매체 Top 30
SELECT source_host, COUNT(*) FROM articles
GROUP BY source_host ORDER BY COUNT(*) DESC LIMIT 30;

-- 한국 매체 확인
SELECT source_host, COUNT(*) FROM articles
WHERE source_host LIKE '%.kr' OR source_host LIKE '%.co.kr'
GROUP BY source_host ORDER BY COUNT(*) DESC LIMIT 20;

-- URL 중복 체크 (0이어야 정상)
SELECT url, COUNT(*) FROM articles GROUP BY url HAVING COUNT(*) > 1;

-- 시간대별 수집량
SELECT date_trunc('hour', gdelt_added_at) as hour, source_file_type, COUNT(*)
FROM articles
GROUP BY hour, source_file_type
ORDER BY hour DESC LIMIT 20;

-- system_log 에러 확인
SELECT level, COUNT(*) FROM system_log GROUP BY level;
SELECT * FROM system_log WHERE level = 'ERROR' ORDER BY created_at DESC LIMIT 10;

-- 컴포넌트별 실행 횟수 (최근 30분)
SELECT component, COUNT(*) FROM system_log
WHERE created_at > NOW() - INTERVAL '30 minutes'
GROUP BY component;
```

## 운영 주기

| 컴포넌트 | 주기 | 설명 |
|---|---|---|
| masterfile_watcher | 5분 | lastupdate + lastupdate-translation 폴링 |
| downloader | 1분 | pending 파일 최대 5개 다운로드 |
| gkg_parser | 1분 | downloaded 파일 최대 3개 파싱 |

## 트러블슈팅

- **"No new GKG files found"**: GDELT 지연은 정상 (수 분~수 시간). §4 참조.
- **MD5 mismatch**: 자동 재시도 (최대 3회). 3회 초과 시 status='failed'.
- **메모리 초과**: 파서가 500MB 넘으면 비정상. `gkg_parser.py`의 스트리밍 깨짐 의심.
- **한글 깨짐**: `xml_utils.py`의 `html.unescape()` 확인.

---

## launchd 자동 재시작 설정 (선택)

macOS 재부팅 시 orchestrator 자동 시작 + 프로세스 다운 시 자동 재시작.

### 설치

```bash
cp scripts/launchd/com.newsintel.orchestrator.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.newsintel.orchestrator.plist
```

### 상태 확인

```bash
launchctl list | grep newsintel
```

### 중지 (재부팅 시 자동 시작 비활성화)

```bash
launchctl unload ~/Library/LaunchAgents/com.newsintel.orchestrator.plist
rm ~/Library/LaunchAgents/com.newsintel.orchestrator.plist
```

### 로그 확인

```bash
tail -f logs/launchd_stdout.log
tail -f logs/launchd_stderr.log
```

### 주의

- venv 경로(`<REPO_ROOT>/venv/bin/python`)와 작업 디렉토리가 정확해야 함
- 환경변수가 필요하면 plist에 `EnvironmentVariables` 키 추가
- 설치 전 orchestrator 수동 실행 검증 권장

---

## 장시간 작업 표준 보고 체계 (2026-04-24 확정)

5분 이상 소요되거나 여러 단계를 포함하는 작업은 본 체계에 따라 보고한다. news_intel Run 실행, 빌드, 마이그레이션, 배치 처리 등 모든 유사 작업에 적용.

### 이중화

- **Telegram 알림**: secretarybot (채널 환경변수로 결정)
- **Claude Code 세션 출력**: 동일 내용 병행 — 세션 로그에 남김

### 보고 시점 6개 포인트

| 이모지 | 시점 | 포함 항목 |
|---|---|---|
| 🚀 | 작업 시작 | 작업명, PID, 예상 소요 시간, 활성 옵션(env/플래그) |
| ✅ | 단계 완료 (Phase/Task 단위 반복) | 단계명, 소요 시간, 핵심 지표(수량·비율) |
| 🔄 | 다음 단계 진입 | 단계명, 입력 요약(무엇을 처리) |
| ✅ | 단계 완료 (반복) | — |
| 📊 | 최종 판정 | 결과 요약(표/bullet), 판정(성공/부분성공/실패), 다음 단계 제안(결정은 사용자) |
| ❌ | 에러 발생 시 | stage명, 1~2줄 에러 요약. **stack trace는 세션 출력에만, Telegram에는 포함 금지** |

### 보안 원칙 (엄수)

- 토큰 / chat_id 값은 메시지·로그·문서 어디에도 **출력 금지**
- **환경변수 참조만** 사용 (변수명 언급은 가능, 값은 절대 금지)
- 발송 스크립트의 토큰 하드코딩 **금지**

### 발송 헬퍼

`scripts/notify_telegram.sh` 사용. 환경변수 참조:

| 우선 | 대체 |
|---|---|
| `SECRETARY_BOT_TOKEN` | `TELEGRAM_BOT_TOKEN` |
| `SECRETARY_CHAT_ID` | `TELEGRAM_CHAT_ID` |

사용 예:
```bash
scripts/notify_telegram.sh "🚀 Run 8 시작
PID 99365
예상 38~45분
옵션: NEWS_INTEL_ENABLE_PHASE2=true, PHASE2_SIZE_THRESHOLD=1500"

scripts/notify_telegram.sh "✅ Phase 1 완료 (34분 5초)
n=92,798, clusters=817, noise=35,509 (38.26%)"

scripts/notify_telegram.sh "❌ Run 8 실패
stage: Phase 1 encoding
원인: MPS Invalid buffer size"
# stack trace는 /tmp/run8.log 또는 세션 출력에만 남기고 Telegram 메시지에는 포함 금지
```

HTML parse_mode 기본. 볼드/링크 사용 시 `<b>`, `<a href="...">` 가능. 메시지 내 `<`, `>`, `&`는 적절히 이스케이프.

### 적용 범위

- news_intel Run (38~96분 규모)
- F 경로, F2, 향후 F+ 경로 등 모든 클러스터링 실험
- 임베딩 재계산, DB 마이그레이션, 대용량 배치
- 제외: 5분 미만 단일 작업은 최종 결과만 보고 (6포인트 불필요)

### 에러 처리 상세

- 크래시/OOM/Traceback → ❌ 메시지 1회
- 재시도/재실행은 사용자 승인 받기 전까지 자동 금지 (현 news_intel 원칙과 동일)
- `anomaly_reason` 같은 DB 기록 필드가 있으면 해당 필드에도 원인 텍스트 저장
