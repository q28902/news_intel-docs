# L1 패치 노트 — 2026-04-17

테스트 브리핑에서 지적된 3건 이슈 수정.

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| `geopolitical_affinity/gdelt_query.py` | **신규 생성**. GDELT BigQuery 단일 진실 소스 |
| `geopolitical_affinity/bootstrap.py` | `_fetch_gdelt()` / `_try_fetch_gdelt()` → gdelt_query 위임 |
| `geopolitical_affinity/updater.py` | `update_gdelt()` → gdelt_query 위임 (SQL 중복 제거) |
| `geopolitical_affinity/layers/L1_structure.py` | dead code `load_and_compute()` 삭제 |
| `geopolitical_affinity/layers/L4_events.py` | `compute()` year_range 디폴트 None으로 변경 + config 참조 |
| `geopolitical_affinity/config.py` | `L4_YEAR_RANGE_START`, `L4_YEAR_RANGE_BUFFER`, `default_l4_year_range()` 추가 |
| `geopolitical_affinity/engine.py` | FBIC 실제 연도 범위를 L4에 주입 |

## 이슈별 상세

### 이슈 #1 — L4 year_range 하드코딩 (2000, 2027)

**문제**: `L4_events.compute(year_range=(2000, 2027))` 이 하드코딩되어 있어 2027년 이후 데이터 생성 시 잘림.

**해결**:
- `config.py`에 `L4_YEAR_RANGE_START=2000`, `L4_YEAR_RANGE_BUFFER=2` 추가
- `config.default_l4_year_range()` 함수로 현재 연도 + buffer 자동 계산
- `L4_events.compute(year_range=None)` 디폴트를 None으로 하고 실제 사용 시 config에서 가져옴
- `engine.init()`이 FBIC 실제 `year.min()` ~ `year.max() + 1 + buffer`를 계산해서 주입

**검증**:
```
config.default_l4_year_range() = (2000, 2028)  # 2026년 시점
```
내년에 실행하면 자동으로 2029로 확장.

### 이슈 #2 — L1.load_and_compute 반환값 불일치

**문제**: 
- 시그니처는 `-> pd.DataFrame`
- 실제로는 `return compute(fbic), fbic` — 튜플 반환
- engine.py는 이 함수를 **호출하지 않음** (직접 `pd.read_csv + L1_structure.compute` 사용)

**해결**: dead code이므로 삭제. L2/L4의 `load_and_compute`는 실제 사용되므로 유지.

### 이슈 #3 — GDELT SQL 중복

**문제**: 40줄짜리 BigQuery SQL이 `bootstrap.py`와 `updater.py`에 문자 단위로 복붙.

**해결**: `gdelt_query.py` 신설, 세 계층으로 분리:

| 함수 | 역할 |
|------|------|
| `build_query(countries)` | SQL 문자열 생성 (순수 함수, 테스트 가능) |
| `fetch_gdelt_dataframe(...)` | BigQuery 실행 → DataFrame 반환 |
| `fetch_and_save(path, ...)` | 위 둘 + CSV 저장 |

bootstrap/updater 모두 `fetch_and_save()` 한 줄 호출로 통일. 국가 코드 변경, WHERE 절 조건 수정, 기간 파라미터 추가 등 향후 진화가 한 곳에서만 처리됨.

## 하위 호환성

- `bootstrap._fetch_gdelt(project_id, output_path)` 시그니처 유지 (내부적으로 gdelt_query로 위임)
- `updater.update_gdelt(project_id)` 시그니처 유지
- `L4_events.compute(events, year_range)` 시그니처는 동일하나 year_range 디폴트만 변경 (기존 하드코딩 대체)
- **Breaking**: `L1_structure.load_and_compute` 삭제 — 하지만 프로젝트 내 호출처 없으므로 영향 없음

## 검증 결과

```
✅ 모든 모듈 import 성공
✅ CLI 모듈 import 성공
✅ py_compile 전체 통과
✅ gdelt_query.build_query 국가 코드 주입 동작
✅ L4.compute year_range 동적 디폴트 동작
✅ L1.load_and_compute 삭제 확인
```

실데이터(FBIC CSV, GDELT BigQuery)가 있는 환경에서 `engine.init()` 호출 테스트는
Mac mini 로컬 환경에서 세열님이 직접 확인 필요.
