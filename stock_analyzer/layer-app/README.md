# Geostock — 토스증권 스타일 UI 재작업 (통합 zip)

본 zip 은 **Phase 1+2+3+4 누적**. 원본 프로젝트(`geostock-layer-app`)에 한 번만 덮어쓰면 끝.

## 적용

```bash
cd /path/to/geostock                                    # 원본 프로젝트 루트
unzip -o ~/Downloads/geostock-toss-redesign-complete.zip
./gradlew assembleDebug                                 # 빌드 검증
./gradlew installDebug                                  # 디바이스 설치
```

`unzip -o` 의 `-o` 는 기존 파일 덮어쓰기 (overwrite). 충돌 프롬프트 없이 진행.

---

## 변경 요약

### Phase 1 — 디자인 토큰 + 폰트 (foundation)

| 파일 | 역할 |
|---|---|
| `app/src/main/res/font/pretendard_{regular,medium,semibold,bold}.otf` | Pretendard 4 weights 번들 (~1.3MB) |
| `app/src/main/java/.../ui/theme/Color.kt` | `GeostockColors` 데이터클래스 + Light/Dark 컬러 시스템. 시멘틱 토큰: bgBase/bgCard/bgSubtle, textPrimary/Secondary/Tertiary, brand, brandSubtle, up/upSubtle (한국식 빨강), down/downSubtle (파랑), warning/success/error 등. |
| `app/src/main/java/.../ui/theme/Type.kt` | `GeostockTypography` — display/title/body/label + numeric (tabular nums) 스타일. Pretendard family 자동 매핑. |
| `app/src/main/java/.../ui/theme/Shape.kt` | `GeostockShapes` (4/8/12/16/20dp). |
| `app/src/main/java/.../ui/theme/Theme.kt` | `GeostockTheme` provider + `LocalGeostockColors`/`LocalGeostockTypography`/`LocalGeostockShapes`. Material3 fallback 도 동시 제공. |

### Phase 2 — 공통 컴포넌트 분리 (`ui/components/`)

| 파일 | 컴포넌트 |
|---|---|
| `Card.kt` | `TossCard(padding, radius, background, onClick)` |
| `TickerIcon.kt` | Toss CDN 로고 + 첫 글자 fallback |
| `ScreenHeader.kt` | `ScreenHeader(title, subtitle?)` (28sp Bold + 14sp gray 통일) |
| `FoldableSection.kt` | chevron 회전 애니메이션 폴딩 헤더 |
| `Pill.kt` | `Pill(label, selected, onClick)` 풀라운드 chip |
| `Numeric.kt` | `BigPrice(value)`, `ChangeText(pctText, prefix?, style)` (한국식 등락 색·부호 자동), `MetricRow(label, value)` |
| `States.kt` | `SkeletonCard`, `LoadingCard(message?)`, `AnalyzingCard(statusLabel, elapsedSec, extra?)`, `EmptyCard(message)`, `ErrorCard(message, label?, onRetry?)` |
| `StarToggle.kt` | 이모지 → Material Icons (Filled.Star / Outlined.StarBorder / Outlined.WorkOutline) |

### Phase 3 — 하단 네비 + edge-to-edge

| 파일 | 변경 |
|---|---|
| `app/build.gradle.kts` | `androidx.compose.material:material-icons-extended` 의존성 추가 |
| `MainActivity.kt` | `enableEdgeToEdge()`, `NavigationBar(tonalElevation=0.dp)`, `indicatorColor=Transparent`, 활성 brand color 처리. 4탭 모두 Material Icons (Home/StarBorder/Search/Schedule). 라우팅 로직(home/watchlist/search/history + detail/{jobId}/history/{file}/chart/{ticker}/{name}) 그대로 보존. |

### Phase 4 — 화면별 적용

| 화면 | 핵심 변경 |
|---|---|
| `HomeScreen.kt` | ScreenHeader, ForecastCard (국기 🇺🇸🇰🇷 유지), FoldableSection 보유/관심, HoldingRow/InterestRow (TickerIcon + ChangeText). polling 10s 그대로. |
| `WatchlistScreen.kt` | ScreenHeader, FoldableSection, HoldingCard/InterestCard. 안내문 톤다운. |
| `SearchScreen.kt` | ScreenHeader, AnalyzeButton primary, OutlinedTextField toss 컬러, StarToggle. 이모지(✅❌⚠️) → 텍스트/색 처리. |
| `DetailScreen.kt` | ScreenHeader, AnalyzingCard, ResultView (헤더 → 큰 시그널박스 → MetricRow 표 → SectionCard). signalColor 한국식 매수=up/매도=down. 이모지 모두 제거. |
| `HistoryScreen.kt` | ScreenHeader, 카드 폴딩 mutableStateMapOf 그대로. 국기 유지, 이모지 제거. |
| `ChartScreen.kt` | TickerIcon + 종목명 헤더, BigPrice + ChangeText, 통계 카드, period chip → Pill 컴포넌트, MAIN_PERIODS 6개 + DropdownMenu. CandleChart 호출에 토큰 컬러 주입. |
| `CandleChart.kt` | 하드코딩 컬러 → 파라미터로 추출 (`gridColor`, `labelColor`, `upColor`, `downColor`, `highlightColor`). 핀치 줌·팬 로직 그대로. |

---

## 디자인 원칙

- **한국 주식 등락 컬러**: 상승=Red(`up=#F04452`), 하락=Blue(`down=#3182F6`). `ChangeText` 컴포넌트가 자동 처리.
- **카드**: 큰 카드 padding=20dp/radius=20dp, 행 카드 padding=16dp/radius=16dp.
- **타이포**: 제목 displayMedium(28sp Bold) + 설명 bodyMedium(14sp gray). 숫자는 numeric* (tabular).
- **이모지 정책**: UI 이모지 모두 제거 → Material Icons. **국기 🇺🇸🇰🇷 만 예외**(시각적 정보 가치).

## 빌드 환경 가정

- compose-bom 2024.12.01
- compileSdk/targetSdk 35, minSdk 26
- Kotlin 1.9.x + compose plugin
- Note 20 Ultra (Android 13) 호환

## 컬러 시스템 빠른 참조

| 토큰 | Light | Dark |
|---|---|---|
| `bgBase` | `#F2F4F6` | `#181A20` |
| `bgCard` | `#FFFFFF` | `#222428` |
| `bgSubtle` | `#F2F4F6` | `#2A2D33` |
| `brand` | `#3182F6` | `#4493F8` |
| `up` (한국식 상승=빨강) | `#F04452` | `#FF5765` |
| `down` (한국식 하락=파랑) | `#3182F6` | `#4493F8` |
| `textPrimary` | `#191F28` | `#F2F4F6` |
| `textTertiary` | `#8B95A1` | `#6B7280` |

상세는 `ui/theme/Color.kt` 참조.

## 향후 마이그레이션 (옵션)

`ui/Theme.kt` 의 backward compat shim (`TossColors`, `GeostockTheme`, `TossCard`, `TickerIcon`)은 외부 진입점 호환용. 모든 화면이 이미 신규 위치(`ui.theme.*`, `ui.components.*`) 사용 중이므로 shim 제거 가능. 단 외부 코드 import 안전 확인 후 제거.
