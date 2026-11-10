# 그늘맵 설계 스펙 (개정)

- 작성일: 2026-09-28 (개정: 체감온도와 서버 도입)
- 로컬 전용 문서. GitHub에 올리지 않는다(.gitignore).
- 관련: 그늘맵 PRD (https://claude.ai/code/artifact/8571f184-07a3-4a9d-912f-b33e6687918f)
- 참고 구현: SHS0814/OSD (팀원 저장소). 참고만 하고 코드는 이 저장소에서 새로 작성한다.

## 1. 목표

충북대학교 캠퍼스의 그늘과 체감온도(UTCI)를 지도에 보여주고, 덜 더운 보행 경로를 안내하는 Android 앱.

1차 범위:

- 그림자 표시, 체감온도 히트맵
- 쾌적 경로 + 최단 경로 비교 (거리, 햇빛 비율, 평균 UTCI)
- GPS 출발지, 장소 입력(캠퍼스 목록 + 지도 탭 + 카카오 검색)
- 체감온도 모델: 1단계 → 2단계 → 3단계 순서로 고도화

제외: iOS, 시간 슬라이더(보류), 지점별 바람과 기온, 유체 시뮬레이션.

## 2. 전체 구조

```
Android 앱 (Kotlin, MapLibre)
  지도 · 그림자 · 체감온도 히트맵 · 경로 2개 + 비교 카드 · GPS · 장소 입력
        │ HTTP(S) / JSON
        ▼
Python 서버 (FastAPI)
  sun      시각 → 태양 고도, 방위각 (astral)
  shadow   건물 → 그림자 폴리곤 (swept polygon, shapely)
  weather  기상청 청주 ASOS 관측값
  climate  격자별 양지 비율 → MRT → UTCI (pythermalcomfort)   ← 단계별로 이 모듈만 교체
  route    보행 그래프 + 격자 → 쾌적 경로 / 최단 경로 (osmnx, networkx)
  places   캠퍼스 장소 목록 + 카카오 검색 중계
        │
data/  오프라인 스크립트(scripts/)로 한 번 만들어 커밋
```

- DB 없음. 서버 시작 시 `data/` 파일을 메모리에 올린다. 필요해지면 PostGIS 추가.
- 범위: 캠퍼스 경계(OSM relation `6705106`) + 바깥 100m.
- 계산 좌표계: EPSG:5179(미터). 응답은 EPSG:4326.
- 비밀 값(`KMA_APIHUB_KEY`, `KAKAO_REST_KEY`)은 서버 `.env`에만. 앱에는 키가 없다.
- 시연: PC에서 서버 실행 + `adb reverse tcp:8000 tcp:8000`. 필요하면 Cloudflare Tunnel.

## 3. 데이터 (`data/`, 커밋)

| 파일 | 내용 | 만드는 스크립트 |
|---|---|---|
| `campus_boundary.geojson` | OSM relation 6705106 경계 | `scripts/fetch_boundary.py` |
| `buildings.geojson` | 건물 외곽선 + `height_m`(없으면 null) + 높이 출처 | `scripts/build_buildings.py` |
| `walk_graph.graphml` | OSMnx 보행 그래프 (범위 안) | `scripts/build_graph.py` |
| `places.json` | `[{name, lat, lon}]` 캠퍼스 주요 장소 | 수작업 |
| `kma_asos_131_hourly.csv` | 청주 ASOS 시간자료 스냅샷 (API 키 없을 때 사용) | `scripts/fetch_kma.py` |
| (2단계) `trees.geojson` | 수관(숲 영역 + 가로수), 높이, 투과율 | `scripts/build_trees.py` |
| (3단계) `dtm.npz`, `landcover.geojson` | 지형 격자, 지면 종류 | `scripts/build_terrain.py`, `scripts/build_landcover.py` |

건물 높이:

- 외곽선은 OSM, 높이는 건축물대장 표제부 우선, OSM `height` 보조. 건물 높이 데이터 확보는 팀이 맡는다.
- 높이 없는 건물은 `height_m = null`. 그림자 계산에서 제외하고 개수를 응답에 포함한다. 층수 환산이나 기본값은 쓰지 않는다.
- 스크립트는 전체, 높이 있음, 높이 없음 건수를 출력한다.

## 4. 그림자 (`shadow`)

- `astral`로 태양 고도 `alt`, 방위각(북쪽 기준 시계방향)을 구한다.
- `L = min(height / tan(alt), 300m)`. 방향은 태양 방위각 + 180°.
- swept polygon: 원 건물 ∪ L만큼 이동한 건물 ∪ 각 변이 이동하며 만드는 사각형. `shapely.union_all`.
- `alt <= 0`이면 빈 결과.

## 5. 체감온도 (`climate`)

인터페이스 (단계가 바뀌어도 유지):

```
compute(dt) -> Climate
  grid: {origin_x, origin_y, cell=10, cols, rows}   # EPSG:5179, 북서쪽부터 행 우선
  sunlit: float[]        # 0~1, 건물 내부 셀은 null
  mrt:    float[] | None
  utci:   float[] | None  # 기상 관측 없으면 None
  weather: {observed_at, ta, rh, wind, ghi} | None
```

### 1단계

1. 격자: 범위를 10m 셀로 나눈다. 건물이 셀 중심을 덮으면 null.
2. 양지 비율: 셀마다 2m 간격 점 25개, `shapely.contains_xy`로 그림자 안인지 판정. `sunlit = 밖 / 25`.
3. 기상: 요청 시각의 ASOS 기온, 습도, 풍속, 일사량(MJ/m²/h → W/m²). 관측이 없으면 최근 3시간 안의 마지막 값, 그것도 없으면 `None`.
4. 일사 분리: Erbs 모델로 직달(DNI)과 산란(DHI).
5. MRT: SOLWEIG 방식 단순식. `Tmrt = ((Ta+273.15)^4 + ak·(fp·DNI·sunlit + 0.5·DHI + 0.5·albedo·GHI)/(εp·σ))^0.25 − 273.15`, `ak=0.7, εp=0.97, albedo=0.15, fp=0.308·cos(alt·(1−alt²/48402))`. 가정: 주변 장파 = 기온의 흑체 복사. (SolarCal `solar_gain`은 실내용 모델이라 쓰지 않음)
6. UTCI: `pythermalcomfort.utci(tdb=ta, tr=mrt, v=wind, rh=rh)` 벡터 계산.
7. 10분 단위 시각으로 캐시.

### 2단계 (+ 나무)

- 수관 폴리곤을 높이만큼 투영한 나무 그림자를 만든다. 나무 그림자 안의 점은 `투과율`만큼만 양지로 센다.
- 활엽수는 4/20~11/10에 잎이 있는 것으로 본다.

### 3단계 (+ 지형, 지면)

- 5m 지형 격자 위에 건물을 세운 표면 모델로 광선 추적(가슴 높이 1.1m에서 태양 방향).
- 지면 종류별 반사율과 지면 가열 계수로 지면 온도를 추정해 MRT에 반영.

## 6. 경로 (`route`)

- 서버 시작 시 `walk_graph.graphml` 로드, EPSG:5179로 투영.
- 구간 열 지수 `w`(0~1): 구간 위 5m 간격 점이 속한 셀 값의 평균.
  - `utci`가 있으면 `w = (구간 UTCI − 격자 최소) / (격자 최대 − 격자 최소)`
  - 없으면 `w = 구간 sunlit 평균`
- 비용: 쾌적 = `length × (1 + K × w)`, `K = 2` / 최단 = `length`.
- 출발지와 도착지는 `osmnx.distance.nearest_nodes`로 붙인다. 범위 밖이면 400.
- `networkx.shortest_path(weight=...)` 두 번.
- 응답: 경로마다 `{coords, distance_m, sun_ratio, mean_utci | null}`, 사용한 기준(`utci` | `sunlit`).

## 7. API

| 엔드포인트 | 응답 |
|---|---|
| `GET /health` | `{"status":"ok"}` |
| `GET /api/buildings` | 건물 FeatureCollection + 전체/높이 있음/없음 건수 |
| `GET /api/shadows?datetime=` | 태양 고도·방위각 + 그림자 FeatureCollection |
| `GET /api/climate?datetime=` | 5장의 `Climate` (격자는 위경도 모서리 4개 포함) |
| `GET /api/route?from=lat,lon&to=lat,lon&datetime=` | 쾌적/최단 경로 + 기준 |
| `GET /api/places?q=` | 캠퍼스 장소 + 카카오 검색 결과(범위 안만) |

- `datetime` 생략 시 현재 시각. 시간대 없는 값은 `Asia/Seoul`.
- 카카오 호출 실패 시 캠퍼스 장소만 반환하고 `kakao_error: true`.

## 8. 앱 (Kotlin)

- 화면 하나. 지도 전체 화면 + 상단 출발/도착 입력 + 레이어 선택(체감온도/그림자/끔) + 하단 비교 카드 + 범례.
- MapLibre Android SDK
  - 3D 건물 `FillExtrusionLayer` (높이 없는 건물은 평평한 회색)
  - 히트맵: 격자 → 셀당 1픽셀 비트맵 → `ImageSource` + `RasterLayer`. null은 투명.
    UTCI 색 구간: <26 / 26~32 / 32~38 / 38~46 / ≥46
  - 그림자 `FillLayer`, 경로 `LineLayer` 2개
- 장소 선택: Material `BottomSheetDialog`(검색창, 장소 목록, "지도에서 선택")
- 의존성: MapLibre, OkHttp, 내장 `org.json`, 내장 `LocationManager`, Material. Retrofit·Compose·Play 서비스 안 씀.
- 10분마다 격자와 그림자 갱신.
- 서버 주소는 Gradle 속성 `API_BASE_URL` (기본 `http://localhost:8000/`, adb reverse 전제).
- 하단 출처 표시: `© OpenStreetMap contributors · 기상청 · 국토교통부`.

## 9. 예외 처리

| 상황 | 처리 |
|---|---|
| 서버 연결 실패 | "서버에 연결할 수 없습니다" + 다시 시도 |
| 위치 권한 거부 | 출발지를 직접 고르도록 안내 |
| 범위 밖 위치/지점 | "충북대 주변만 지원합니다" |
| 기상 관측 없음 | 히트맵 숨김 + "기상 관측 없음, 그늘 기준 경로" 배너 |
| 해 진 뒤 | 그림자 레이어 숨김. 체감온도와 경로는 표시 |
| 두 경로가 같음 | "최단 경로가 이미 가장 쾌적한 길입니다" |

## 10. 테스트

서버 (pytest):

- 여름 정오 청주에서 그림자가 북쪽, 고도 45°에서 20m 건물 그림자 길이 20m
- ㄱ자 건물 swept 그림자 면적 < convex hull 면적
- 그림자에 덮인 셀 `sunlit == 0`
- 같은 기상에서 양지 셀 UTCI > 그늘 셀 UTCI
- Erbs 분리 후 직달 수평성분 + 산란 = 전일사
- 가상 그래프: 쾌적 경로는 그늘 우회로, 최단 경로는 직선
- 기상 없음 → `sunlit` 기준으로 경로가 나옴
- 성능: 실제 데이터로 `/api/climate`, `/api/route` 각각 1초 이내 (측정 후 기록)

앱 (JUnit): UTCI → 색 변환, 격자 → 비트맵 픽셀 위치.
UI는 실기기 체크리스트.

## 11. 미결 사항

- K 값 실제 경로로 검증
- 캠퍼스 주요 장소 목록
- 2단계 가로수 데이터 출처 (청주시 공공데이터 또는 수작업 입력)
- 3단계 지형 원본(수치지도 1:5,000) 확보 방법
