# 그늘맵 1단계 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 충북대 캠퍼스의 그림자와 체감온도(UTCI) 히트맵을 보여주고, 쾌적 경로와 최단 경로를 비교해 안내하는 Android 앱 + FastAPI 서버 (체감온도 모델 1단계).

**Architecture:** 계산은 전부 Python 서버가 한다. 데이터(경계, 건물, 보행 그래프, 기상 스냅샷, 장소)는 오프라인 스크립트로 `data/`에 만들어 커밋하고, 서버는 시작할 때 메모리에 올린다(DB 없음). Kotlin 앱은 REST로 받은 GeoJSON과 격자를 MapLibre에 그린다.

**Tech Stack:** Python 3.11+, FastAPI, shapely 2, pyproj, astral, numpy, pythermalcomfort, osmnx 2, networkx, httpx, pytest / Kotlin, Android SDK 35(minSdk 26), MapLibre Android SDK, OkHttp, Material

**Spec:** `docs/superpowers/specs/2026-09-28-shade-map-design.md` (로컬 전용)

**범위:** GitHub 이슈 #11~#23. 2단계(#24), 3단계(#25)는 1단계 검증 후 별도 계획.

## Global Constraints

- 저장소: iamLeeHyeon/shadow_map. 커밋 작성자는 iamLeeHyeon만. 커밋 메시지에 `Co-Authored-By` 줄 금지.
- `CLAUDE.md`, `.claude/`, `docs/superpowers/`는 커밋 금지(이미 `.gitignore`).
- 커밋 메시지 끝에 이슈 번호 `(#N)`. PR 본문에 `Closes #N`.
- 계산 좌표계 EPSG:5179(미터), 응답은 EPSG:4326(경도, 위도 순서의 GeoJSON).
- 시간대 없는 시각은 `Asia/Seoul`. `datetime` 생략 시 현재 시각.
- 범위: 캠퍼스 경계(OSM relation 6705106) + 바깥 100m.
- 격자 10m, 북서쪽부터 행 우선. 건물 내부·범위 밖 셀은 JSON `null`.
- 그림자 최대 길이 300m. 높이 없는 건물은 그림자 계산 제외, 층수 환산·기본값 금지.
- 경로 가중치 `K = 2`. 구간 샘플 간격 5m.
- 비밀 값(`KMA_APIHUB_KEY`, `KAKAO_REST_KEY`, `DATA_GO_KR_API_KEY`)은 `.env`에만. 앱에는 키 없음.
- 앱 의존성: MapLibre, OkHttp, Material, androidx core/appcompat만. Retrofit·Compose·Play 서비스 금지.
- 응답 JSON에 `NaN` 금지(항상 `null`로).

## Review Focus

1. 출발지와 도착지가 같은 노드로 붙음 → 거리 0 경로를 정상 응답해야 한다 (Task 9 테스트).
2. `MultiPolygon` 건물 → 그림자 계산이 죽지 않아야 한다 (Task 4 테스트).
3. 해가 낮을 때(고도 1°) 일사량이 있으면 Erbs 직달이 폭주 → 직달은 1100 W/m² 이하로 잘려야 한다 (Task 6 테스트).
4. 잘못된 `datetime` 문자열 → 500이 아니라 422 (Task 4 테스트).
5. UTCI가 유효 범위를 벗어나 `nan`이 되는 셀 → JSON에 `null`로 나가야 한다 (Task 6 테스트).

## File Structure

```
.env.example                      비밀 값 목록
README.md                         실행 방법 (Task 1, 10, 17)
backend/
  requirements.txt, requirements-dev.txt, pytest.ini
  app/config.py      경로, 상수, 환경변수, parse_dt
  app/geo.py         4326 ↔ 5179 변환
  app/store.py       data/ 파일 로드 → Store
  app/sun.py         태양 고도·방위각
  app/shadow.py      swept polygon 그림자
  app/weather.py     ASOS 스냅샷 + API허브 실시간
  app/climate.py     격자, 양지 비율, Erbs, MRT, UTCI, 캐시
  app/route.py       그래프 로드, 구간 샘플, 쾌적/최단 경로
  app/places.py      캠퍼스 장소 + 카카오 검색
  app/api.py         모든 엔드포인트
  app/main.py        FastAPI 앱, lifespan
  tests/…            모듈별 테스트
scripts/
  fetch_boundary.py, build_buildings.py, build_graph.py, fetch_kma.py, bench.py
data/
  campus_boundary.geojson, buildings.geojson, walk_graph.graphml,
  kma_asos_131_hourly.csv, places.json, raw/(커밋 제외)
android/ (Android Studio 템플릿 + 아래 파일)
  app/src/main/java/kr/ac/cbnu/shademap/
    MainActivity.kt   지도, 레이어, 화면 흐름
    Api.kt            OkHttp + org.json 래퍼
    Heatmap.kt        UTCI 색, 격자 → 픽셀 (순수 함수)
    PlacePicker.kt    장소 선택 BottomSheet
  app/src/main/res/layout/activity_main.xml
  app/src/test/java/kr/ac/cbnu/shademap/HeatmapTest.kt
```

서버 명령은 모두 `backend/`에서, 스크립트는 저장소 루트에서 `PYTHONPATH=backend python scripts/…`로 실행한다.

---

### Task 1: 서버 뼈대 (#11)

**Files:**
- Create: `backend/requirements.txt`, `backend/requirements-dev.txt`, `backend/pytest.ini`, `backend/app/__init__.py`, `backend/app/config.py`, `backend/app/api.py`, `backend/app/main.py`, `backend/tests/test_health.py`, `.env.example`
- Modify: `.gitignore`, `README.md`

**Interfaces:**
- Produces: `config.DATA_DIR: Path`, `config.CRS_M = "EPSG:5179"`, `config.TZ`, `config.CAMPUS_LAT/LON`, `config.BUFFER_M=100`, `config.CELL_M=10`, `config.MAX_SHADOW_M=300`, `config.K_COMFORT=2.0`, `config.SAMPLE_M=5`, `config.KMA_APIHUB_KEY`, `config.KAKAO_REST_KEY`, `config.parse_dt(s: str | None) -> datetime`, `api.router`, `main.app`

- [ ] **Step 1: 의존성 파일**

`backend/requirements.txt`:
```
fastapi>=0.115
uvicorn[standard]>=0.30
shapely>=2.0
pyproj>=3.6
astral>=3.2
numpy>=1.26
pythermalcomfort>=2.10
osmnx>=2.0
networkx>=3.3
httpx>=0.27
python-dotenv>=1.0
```

`backend/requirements-dev.txt`:
```
-r requirements.txt
pytest>=8
```

`backend/pytest.ini`:
```ini
[pytest]
pythonpath = .
testpaths = tests
```

`.env.example` (루트):
```
KMA_APIHUB_KEY=
KAKAO_REST_KEY=
DATA_GO_KR_API_KEY=
```

`.gitignore`에 추가:
```
.env
.venv/
__pycache__/
data/raw/
android/.gradle/
android/build/
android/app/build/
android/local.properties
android/.idea/
.DS_Store
```

- [ ] **Step 2: 실패하는 테스트**

`backend/tests/test_health.py`:
```python
from datetime import datetime

import pytest
from fastapi import HTTPException
from fastapi.testclient import TestClient

from app.config import TZ, parse_dt
from app.main import app


def test_health():
    assert TestClient(app).get("/health").json() == {"status": "ok"}


def test_parse_dt_naive_is_seoul():
    assert parse_dt("2026-07-01T12:00:00") == datetime(2026, 7, 1, 12, tzinfo=TZ)


def test_parse_dt_offset_converted_to_seoul():
    assert parse_dt("2026-07-01T03:00:00+00:00").hour == 12


def test_parse_dt_invalid_is_422():
    with pytest.raises(HTTPException) as e:
        parse_dt("not-a-date")
    assert e.value.status_code == 422
```

- [ ] **Step 3: 실패 확인**

```bash
cd backend && python3 -m venv ../.venv && source ../.venv/bin/activate && pip install -r requirements-dev.txt
pytest -q
```
Expected: FAIL (`ModuleNotFoundError: app.config`)

- [ ] **Step 4: 구현**

`backend/app/__init__.py`: 빈 파일.

`backend/app/config.py`:
```python
import os
from datetime import datetime
from pathlib import Path
from zoneinfo import ZoneInfo

from dotenv import load_dotenv
from fastapi import HTTPException

ROOT = Path(__file__).resolve().parents[2]
load_dotenv(ROOT / ".env")

DATA_DIR = ROOT / "data"
CRS_M = "EPSG:5179"
TZ = ZoneInfo("Asia/Seoul")
CAMPUS_LAT, CAMPUS_LON = 36.6268, 127.4583
CAMPUS_RELATION = 6705106
BUFFER_M = 100
CELL_M = 10
MAX_SHADOW_M = 300
K_COMFORT = 2.0
SAMPLE_M = 5

KMA_APIHUB_KEY = os.getenv("KMA_APIHUB_KEY", "")
KAKAO_REST_KEY = os.getenv("KAKAO_REST_KEY", "")


def parse_dt(s: str | None) -> datetime:
    if not s:
        return datetime.now(TZ)
    try:
        dt = datetime.fromisoformat(s)
    except ValueError:
        raise HTTPException(422, f"invalid datetime: {s}")
    return dt.replace(tzinfo=TZ) if dt.tzinfo is None else dt.astimezone(TZ)
```

`backend/app/api.py`:
```python
from fastapi import APIRouter

router = APIRouter()


@router.get("/health")
def health():
    return {"status": "ok"}
```

`backend/app/main.py`:
```python
from fastapi import FastAPI
from fastapi.middleware.gzip import GZipMiddleware

from app import api

app = FastAPI(title="shadow_map")
app.add_middleware(GZipMiddleware, minimum_size=1000)
app.include_router(api.router)
```

- [ ] **Step 5: 통과 확인**

Run: `cd backend && pytest -q`
Expected: 4 passed

- [ ] **Step 6: README**

`README.md` 전체를 다음으로:
````markdown
# shadow_map

충북대학교 캠퍼스의 그늘과 체감온도를 보여주고, 덜 더운 보행 경로를 안내하는 Android 앱과 서버.

## 서버 실행

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r backend/requirements-dev.txt
cp .env.example .env   # 키는 선택. 없으면 커밋된 기상 스냅샷만 사용
cd backend && uvicorn app.main:app --reload
```

테스트: `cd backend && pytest -q`
````

- [ ] **Step 7: 커밋**

```bash
git add .gitignore .env.example README.md backend
git commit -m "feat: add FastAPI server skeleton (#11)"
```

---

### Task 2: 캠퍼스 경계와 건물 데이터 (#13 데이터·서버)

**Files:**
- Create: `backend/app/geo.py`, `backend/app/store.py`, `scripts/fetch_boundary.py`, `scripts/build_buildings.py`, `backend/tests/test_store.py`, `data/campus_boundary.geojson`, `data/buildings.geojson`
- Modify: `backend/app/api.py`, `backend/app/main.py`

**Interfaces:**
- Consumes: `config.*` (Task 1)
- Produces:
  - `geo.to_m(geom) -> geom`, `geo.to_ll(geom) -> geom`, `geo.lonlat_to_m(lon, lat) -> (x, y)`, `geo.m_to_lonlat(x, y) -> (lon, lat)`
  - `store.Building(name: str, height_m: float | None, geom_m: Polygon | MultiPolygon, feature: dict)`
  - `store.Store(area_m: Polygon, buildings: list[Building], graph=None, edge_idx=None, places: list[dict]=[])` (graph, edge_idx, places는 Task 8, 9, 10에서 채움)
  - `store.load(data_dir: Path) -> Store`
  - `GET /api/buildings` → `{"type":"FeatureCollection","features":[…],"counts":{"total":n,"with_height":n,"without_height":n}}`

- [ ] **Step 1: 경계 수집 스크립트**

`scripts/fetch_boundary.py`:
```python
"""OSM relation 6705106(충북대) 경계 → data/campus_boundary.geojson"""
import json

import osmnx as ox

from app.config import CAMPUS_RELATION, DATA_DIR

gdf = ox.geocode_to_gdf(f"R{CAMPUS_RELATION}", by_osmid=True)
geom = gdf.geometry.iloc[0]
out = DATA_DIR / "campus_boundary.geojson"
out.write_text(json.dumps({"type": "FeatureCollection", "features": [
    {"type": "Feature", "properties": {"osm_relation": CAMPUS_RELATION}, "geometry": geom.__geo_interface__}
]}))
print(f"{out}: area {gdf.to_crs('EPSG:5179').area.iloc[0] / 1e4:.1f} ha")
```

Run: `PYTHONPATH=backend python scripts/fetch_boundary.py`
Expected: `…campus_boundary.geojson: area 8x.x ha` (참고 조사값 87.9 ha 근처)

- [ ] **Step 2: 건물 스크립트**

높이 입력은 팀이 만드는 `data/raw/heights.csv`(열: `osm_id,height_m,source`, source는 `ledger` 또는 `osm`)와 OSM `height` 태그. CSV가 우선.

`scripts/build_buildings.py`:
```python
"""경계+100m 안 OSM 건물에 높이를 붙여 data/buildings.geojson 생성.

높이: data/raw/heights.csv(건축물대장 등, 팀이 작성) > OSM height 태그. 없으면 null.
층수 환산이나 기본값은 쓰지 않는다.
"""
import csv
import json

import osmnx as ox
from shapely.geometry import shape

from app.config import BUFFER_M, DATA_DIR
from app.geo import to_ll, to_m


def parse_height(v) -> float | None:
    try:
        h = float(str(v).lower().replace("m", "").strip())
    except ValueError:
        return None
    return h if h > 0 else None


boundary = shape(json.loads((DATA_DIR / "campus_boundary.geojson").read_text())["features"][0]["geometry"])
area = to_ll(to_m(boundary).buffer(BUFFER_M))
gdf = ox.features_from_polygon(area, tags={"building": True})
gdf = gdf[gdf.geometry.geom_type.isin(["Polygon", "MultiPolygon"])]

manual = {}
csv_path = DATA_DIR / "raw" / "heights.csv"
if csv_path.exists():
    with csv_path.open() as f:
        manual = {int(r["osm_id"]): (float(r["height_m"]), r["source"]) for r in csv.DictReader(f)}

features, with_h = [], 0
for (kind, osm_id), row in gdf.iterrows():
    h, src = manual.get(osm_id, (None, None))
    if h is None and (h := parse_height(row.get("height"))) is not None:
        src = "osm"
    with_h += h is not None
    features.append({"type": "Feature", "geometry": row.geometry.__geo_interface__, "properties": {
        "osm_id": int(osm_id),
        "name": next((n for n in (row.get("name:ko"), row.get("name")) if isinstance(n, str)), None),  # NaN 제외
        "height_m": h, "height_source": src}})

(DATA_DIR / "buildings.geojson").write_text(json.dumps(
    {"type": "FeatureCollection", "features": features}, ensure_ascii=False))
print(f"total {len(features)}, with_height {with_h}, without_height {len(features) - with_h}")
```

- [ ] **Step 3: geo, store 실패 테스트**

`backend/tests/test_store.py`:
```python
import json

from shapely.geometry import box

from app import store
from app.geo import m_to_lonlat, to_ll, to_m


def write_fixture(tmp_path):
    area = to_ll(box(958000, 1946000, 958400, 1946400))
    (tmp_path / "campus_boundary.geojson").write_text(json.dumps({"type": "FeatureCollection", "features": [
        {"type": "Feature", "properties": {}, "geometry": area.__geo_interface__}]}))
    b1 = to_ll(box(958100, 1946100, 958120, 1946120))
    b2 = to_ll(box(958200, 1946200, 958220, 1946220))
    (tmp_path / "buildings.geojson").write_text(json.dumps({"type": "FeatureCollection", "features": [
        {"type": "Feature", "geometry": b1.__geo_interface__, "properties": {"name": "A", "height_m": 20.0}},
        {"type": "Feature", "geometry": b2.__geo_interface__, "properties": {"name": "B", "height_m": None}},
    ]}))


def test_roundtrip():
    lon, lat = m_to_lonlat(958000, 1946000)
    assert abs(to_m(to_ll(box(958000, 1946000, 958001, 1946001))).bounds[0] - 958000) < 0.01
    assert 127 < lon < 128 and 36 < lat < 37


def test_load(tmp_path):
    write_fixture(tmp_path)
    s = store.load(tmp_path)
    assert len(s.buildings) == 2
    assert s.buildings[1].height_m is None
    assert s.area_m.area > 400 * 400  # 100m 버퍼 포함
    assert abs(s.buildings[0].geom_m.area - 400) < 1
```

EPSG:5179에서 x≈958000, y≈1946000은 청주 부근이다.

- [ ] **Step 4: 실패 확인**

Run: `cd backend && pytest tests/test_store.py -q`
Expected: FAIL (`ModuleNotFoundError: app.geo`)

- [ ] **Step 5: 구현**

`backend/app/geo.py`:
```python
from pyproj import Transformer
from shapely.ops import transform

from app.config import CRS_M

_TO_M = Transformer.from_crs("EPSG:4326", CRS_M, always_xy=True)
_TO_LL = Transformer.from_crs(CRS_M, "EPSG:4326", always_xy=True)


def to_m(geom):
    return transform(_TO_M.transform, geom)


def to_ll(geom):
    return transform(_TO_LL.transform, geom)


def lonlat_to_m(lon: float, lat: float) -> tuple[float, float]:
    return _TO_M.transform(lon, lat)


def m_to_lonlat(x: float, y: float) -> tuple[float, float]:
    return _TO_LL.transform(x, y)
```

`backend/app/store.py`:
```python
import json
from dataclasses import dataclass, field
from pathlib import Path

from shapely.geometry import MultiPolygon, Polygon, shape

from app.config import BUFFER_M
from app.geo import to_m


@dataclass
class Building:
    name: str | None
    height_m: float | None
    geom_m: Polygon | MultiPolygon
    feature: dict


@dataclass
class Store:
    area_m: Polygon
    buildings: list[Building]
    graph: object = None
    edge_idx: dict | None = None
    places: list[dict] = field(default_factory=list)


def _features(path: Path) -> list[dict]:
    return json.loads(path.read_text())["features"]


def load(data_dir: Path) -> Store:
    boundary = shape(_features(data_dir / "campus_boundary.geojson")[0]["geometry"])
    buildings = [
        Building(f["properties"].get("name"), f["properties"].get("height_m"), to_m(shape(f["geometry"])), f)
        for f in _features(data_dir / "buildings.geojson")
    ]
    return Store(area_m=to_m(boundary).buffer(BUFFER_M), buildings=buildings)
```

`backend/app/api.py`에 추가:
```python
from fastapi import Request


@router.get("/api/buildings")
def buildings(request: Request):
    bs = request.app.state.store.buildings
    with_h = sum(b.height_m is not None for b in bs)
    return {"type": "FeatureCollection", "features": [b.feature for b in bs],
            "counts": {"total": len(bs), "with_height": with_h, "without_height": len(bs) - with_h}}
```

`backend/app/main.py`를 다음으로:
```python
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.gzip import GZipMiddleware

from app import api, store
from app.config import DATA_DIR


@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.store = store.load(DATA_DIR)
    yield


app = FastAPI(title="shadow_map", lifespan=lifespan)
app.add_middleware(GZipMiddleware, minimum_size=1000)
app.include_router(api.router)
```

- [ ] **Step 6: API 테스트 추가 후 통과 확인**

`backend/tests/test_store.py`에 추가:
```python
from fastapi.testclient import TestClient

from app.main import app


def test_buildings_api(tmp_path):
    write_fixture(tmp_path)
    app.state.store = store.load(tmp_path)
    body = TestClient(app).get("/api/buildings").json()
    assert body["counts"] == {"total": 2, "with_height": 1, "without_height": 1}
```

`TestClient(app)`를 `with` 없이 쓰면 lifespan이 돌지 않으므로 `app.state.store`를 직접 넣은 값이 쓰인다.

Run: `cd backend && pytest -q`
Expected: 모두 PASS

- [ ] **Step 7: 실제 데이터 생성과 검증 테스트**

```bash
PYTHONPATH=backend python scripts/build_buildings.py
```
Expected: `total N, with_height M, without_height K` 출력. `data/raw/heights.csv`가 없으면 OSM 태그만 반영된다.

`backend/tests/test_data_snapshot.py`:
```python
import json

from shapely.geometry import shape

from app.config import DATA_DIR
from app.store import load


def test_snapshot_valid():
    s = load(DATA_DIR)
    assert s.buildings
    for b in s.buildings:
        assert b.geom_m.is_valid and b.geom_m.geom_type in ("Polygon", "MultiPolygon")
        assert s.area_m.buffer(1).contains(b.geom_m.representative_point())


def test_no_floor_based_heights():
    for f in json.loads((DATA_DIR / "buildings.geojson").read_text())["features"]:
        p = f["properties"]
        assert (p["height_m"] is None) == (p["height_source"] is None)
```

Run: `cd backend && pytest -q` → PASS

- [ ] **Step 8: 커밋**

```bash
git add backend scripts/fetch_boundary.py scripts/build_buildings.py data/campus_boundary.geojson data/buildings.geojson
git commit -m "feat: add campus boundary and building height data (#13)"
```

---

### Task 3: 보행 그래프 (#14)

**Files:**
- Create: `scripts/build_graph.py`, `data/walk_graph.graphml`

**Interfaces:**
- Produces: `data/walk_graph.graphml` (EPSG:4326, osmnx 형식, 가장 큰 연결 요소만)

- [ ] **Step 1: 스크립트**

`scripts/build_graph.py`:
```python
"""경계+100m 안 보행 그래프 → data/walk_graph.graphml"""
import json

import networkx as nx
import osmnx as ox
from shapely.geometry import shape

from app.config import BUFFER_M, DATA_DIR
from app.geo import to_ll, to_m

boundary = shape(json.loads((DATA_DIR / "campus_boundary.geojson").read_text())["features"][0]["geometry"])
area = to_ll(to_m(boundary).buffer(BUFFER_M))
G = ox.graph_from_polygon(area, network_type="walk", retain_all=True)
print(f"raw: nodes {len(G)}, edges {G.number_of_edges()}, components {nx.number_weakly_connected_components(G)}")
G = ox.truncate.largest_component(G, strongly=False)
ox.save_graphml(G, DATA_DIR / "walk_graph.graphml")
print(f"saved: nodes {len(G)}, edges {G.number_of_edges()}")
```

- [ ] **Step 2: 실행**

Run: `PYTHONPATH=backend python scripts/build_graph.py`
Expected: `saved: nodes N, edges M` (N은 수백~수천)

- [ ] **Step 3: 연결 확인**

```bash
PYTHONPATH=backend python -c "
import osmnx as ox; from app.config import DATA_DIR
G = ox.load_graphml(DATA_DIR/'walk_graph.graphml')
import networkx as nx; print(nx.is_weakly_connected(G))"
```
Expected: `True`

- [ ] **Step 4: 커밋**

```bash
git add scripts/build_graph.py data/walk_graph.graphml
git commit -m "feat: add walk graph for campus routing (#14)"
```

---

### Task 4: 태양과 그림자 (#16 서버)

**Files:**
- Create: `backend/app/sun.py`, `backend/app/shadow.py`, `backend/tests/test_shadow.py`
- Modify: `backend/app/api.py`

**Interfaces:**
- Consumes: `store.Building`, `geo.to_ll`, `config.parse_dt`
- Produces:
  - `sun.position(dt: datetime) -> tuple[float, float]` (고도°, 북쪽 기준 시계방향 방위각°)
  - `shadow.shadow_of(geom_m, height: float, alt: float, az: float) -> BaseGeometry`
  - `shadow.shadows(buildings: list[Building], alt: float, az: float) -> BaseGeometry` (해가 지면 빈 GeometryCollection)
  - `GET /api/shadows?datetime=` → `{"datetime","altitude","azimuth","type":"FeatureCollection","features":[…]}`

- [ ] **Step 1: 실패하는 테스트**

`backend/tests/test_shadow.py`:
```python
from datetime import datetime

from fastapi.testclient import TestClient
from shapely.affinity import translate
from shapely.geometry import MultiPolygon, Polygon, box
from shapely.ops import unary_union

from app.config import TZ
from app.shadow import shadow_of, shadows
from app.store import Building
from app.sun import position

SQ = box(0, 0, 10, 10)


def test_summer_noon_sun_is_south_and_high():
    alt, az = position(datetime(2026, 6, 21, 12, 30, tzinfo=TZ))
    assert alt > 70 and 160 < az < 200


def test_shadow_points_north_when_sun_south():
    s = shadow_of(SQ, 10, 60, 180)
    assert s.bounds[3] > 10 and s.bounds[1] == 0


def test_shadow_length_at_45_degrees():
    s = shadow_of(SQ, 20, 45, 180)
    assert abs(s.bounds[3] - 30) < 1e-6


def test_shadow_capped_at_300m():
    s = shadow_of(SQ, 50, 1, 180)
    assert abs(s.bounds[3] - 310) < 1e-6


def test_l_shape_swept_smaller_than_hull():
    L = Polygon([(0, 0), (30, 0), (30, 10), (10, 10), (10, 30), (0, 30)])
    s = shadow_of(L, 10, 45, 225)  # 남서쪽 해 → 북동쪽 그림자
    hull = unary_union([L, translate(L, 10 * 0.7071, 10 * 0.7071)]).convex_hull
    assert s.area < hull.area


def test_multipolygon_building():
    mp = MultiPolygon([box(0, 0, 10, 10), box(20, 0, 30, 10)])
    assert shadow_of(mp, 10, 45, 180).area > mp.area


def test_night_is_empty_and_skips_unknown_height():
    bs = [Building("a", 10.0, SQ, {}), Building("b", None, box(50, 50, 60, 60), {})]
    assert shadows(bs, -5, 180).is_empty
    assert shadows(bs, 45, 180).bounds[2] <= 10  # b는 제외


def test_shadows_api_and_invalid_datetime(tmp_path):
    from app.main import app
    from app.store import Store
    app.state.store = Store(area_m=box(0, 0, 1, 1), buildings=[Building("a", 10.0, box(958100, 1946100, 958110, 1946110), {})])
    c = TestClient(app)
    body = c.get("/api/shadows", params={"datetime": "2026-07-01T12:00:00"}).json()
    assert body["altitude"] > 60 and len(body["features"]) == 1
    assert c.get("/api/shadows", params={"datetime": "bad"}).status_code == 422
```

- [ ] **Step 2: 실패 확인**

Run: `cd backend && pytest tests/test_shadow.py -q`
Expected: FAIL (`ModuleNotFoundError: app.shadow`)

- [ ] **Step 3: 구현**

`backend/app/sun.py`:
```python
from datetime import datetime

from astral import Observer
from astral.sun import azimuth, elevation

from app.config import CAMPUS_LAT, CAMPUS_LON

_OBS = Observer(latitude=CAMPUS_LAT, longitude=CAMPUS_LON)


def position(dt: datetime) -> tuple[float, float]:
    return elevation(_OBS, dt), azimuth(_OBS, dt)
```

`backend/app/shadow.py`:
```python
import math

import shapely
from shapely.affinity import translate
from shapely.geometry import GeometryCollection, Polygon

from app.config import MAX_SHADOW_M


def _polygons(geom):
    return list(geom.geoms) if hasattr(geom, "geoms") else [geom]


def shadow_of(geom_m, height: float, alt: float, az: float):
    length = min(height / math.tan(math.radians(alt)), MAX_SHADOW_M)
    dx = -length * math.sin(math.radians(az))
    dy = -length * math.cos(math.radians(az))
    parts = [geom_m, translate(geom_m, dx, dy)]
    for poly in _polygons(geom_m):
        cs = list(poly.exterior.coords)
        for (x1, y1), (x2, y2) in zip(cs, cs[1:]):
            q = Polygon([(x1, y1), (x2, y2), (x2 + dx, y2 + dy), (x1 + dx, y1 + dy)])
            if q.area > 0:
                parts.append(q)
    return shapely.union_all(parts)


def shadows(buildings, alt: float, az: float):
    if alt <= 0:
        return GeometryCollection()
    return shapely.union_all([shadow_of(b.geom_m, b.height_m, alt, az) for b in buildings if b.height_m])
```

`backend/app/api.py`에 추가:
```python
from fastapi import Query

from app import shadow, sun
from app.config import parse_dt
from app.geo import to_ll


@router.get("/api/shadows")
def shadows(request: Request, datetime: str | None = Query(None)):
    dt = parse_dt(datetime)
    alt, az = sun.position(dt)
    geom = shadow.shadows(request.app.state.store.buildings, alt, az)
    feats = [] if geom.is_empty else [{"type": "Feature", "properties": {}, "geometry": to_ll(geom).__geo_interface__}]
    return {"datetime": dt.isoformat(), "altitude": alt, "azimuth": az, "type": "FeatureCollection", "features": feats}
```

- [ ] **Step 4: 통과 확인**

Run: `cd backend && pytest -q`
Expected: 모두 PASS

- [ ] **Step 5: 커밋**

```bash
git add backend
git commit -m "feat: compute swept-polygon building shadows (#16)"
```

---

### Task 5: 기상 스냅샷과 조회 (#15 1부)

**Files:**
- Create: `scripts/fetch_kma.py`, `backend/app/weather.py`, `backend/tests/test_weather.py`, `data/kma_asos_131_hourly.csv`
- Modify: `backend/app/main.py`

**Interfaces:**
- Produces:
  - `weather.Obs(observed_at: datetime, ta: float, rh: float, wind: float, ghi: float)` (ghi는 W/m²)
  - `weather.load_csv(path: Path) -> dict[datetime, Obs]`
  - `weather.Weather(obs: dict[datetime, Obs])`, `.at(dt) -> Obs | None`, `.add(items: list[Obs])`
  - `app.state.weather: Weather`

- [ ] **Step 1: 스냅샷 수집 스크립트**

공공데이터포털 "기상청_지상(종관, ASOS) 시간자료 조회서비스" 활용신청 후 `.env`의 `DATA_GO_KR_API_KEY`에 Decoding 키를 넣는다.

`scripts/fetch_kma.py`:
```python
"""청주 ASOS(131) 시간자료 → data/kma_asos_131_hourly.csv. 사용: python scripts/fetch_kma.py 20250601 20260927"""
import csv
import os
import sys
from datetime import date, timedelta

import httpx

from app.config import DATA_DIR

URL = "http://apis.data.go.kr/1360000/AsosHourlyInfoService/getWthrDataList"
KEY = os.environ["DATA_GO_KR_API_KEY"]
start, end = (date.fromisoformat(f"{a[:4]}-{a[4:6]}-{a[6:]}") for a in sys.argv[1:3])

rows = []
d = start
while d <= end:
    e = min(d + timedelta(days=30), end)
    r = httpx.get(URL, timeout=30, params={
        "serviceKey": KEY, "pageNo": 1, "numOfRows": 999, "dataType": "JSON", "dataCd": "ASOS", "dateCd": "HR",
        "startDt": d.strftime("%Y%m%d"), "startHh": "00", "endDt": e.strftime("%Y%m%d"), "endHh": "23", "stnIds": "131"})
    r.raise_for_status()
    items = r.json()["response"]["body"]["items"]["item"]
    rows += [(i["tm"], i["ta"], i["hm"], i["ws"], i["icsr"]) for i in items]
    d = e + timedelta(days=1)

with (DATA_DIR / "kma_asos_131_hourly.csv").open("w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["tm", "ta", "hm", "ws", "icsr"])
    w.writerows(rows)
print(f"{len(rows)} hours")
```

Run: `set -a; source .env; set +a; PYTHONPATH=backend python scripts/fetch_kma.py 20250601 <어제 날짜>`
Expected: `N hours` (하루 24시간 × 일수 근처)

- [ ] **Step 2: 실패하는 테스트**

`backend/tests/test_weather.py`:
```python
from datetime import datetime

from app.config import TZ
from app.weather import Weather, load_csv


def write_csv(tmp_path):
    p = tmp_path / "w.csv"
    p.write_text("tm,ta,hm,ws,icsr\n"
                 "2026-07-01 12:00,30.1,60,2.0,3.2\n"
                 "2026-07-01 13:00,,55,2.0,3.0\n"
                 "2026-07-01 20:00,26.0,70,1.0,\n")
    return p


def test_load_and_units(tmp_path):
    obs = load_csv(write_csv(tmp_path))
    o = obs[datetime(2026, 7, 1, 12, tzinfo=TZ)]
    assert o.ta == 30.1 and abs(o.ghi - 3.2e6 / 3600) < 1e-6
    assert datetime(2026, 7, 1, 13, tzinfo=TZ) not in obs  # 기온 결측 행은 버림
    assert obs[datetime(2026, 7, 1, 20, tzinfo=TZ)].ghi == 0  # 밤 일사 빈칸은 0


def test_at_uses_hour_and_falls_back_3h(tmp_path):
    w = Weather(load_csv(write_csv(tmp_path)))
    assert w.at(datetime(2026, 7, 1, 12, 40, tzinfo=TZ)).ta == 30.1
    assert w.at(datetime(2026, 7, 1, 14, 10, tzinfo=TZ)).ta == 30.1  # 13시 결측 → 12시
    assert w.at(datetime(2026, 7, 1, 16, 0, tzinfo=TZ)) is None  # 4시간 전은 너무 오래됨
    assert w.at(datetime(2025, 1, 1, tzinfo=TZ)) is None
```

- [ ] **Step 3: 실패 확인**

Run: `cd backend && pytest tests/test_weather.py -q`
Expected: FAIL (`ModuleNotFoundError: app.weather`)

- [ ] **Step 4: 구현**

`backend/app/weather.py`:
```python
import csv
from dataclasses import dataclass
from datetime import datetime, timedelta
from pathlib import Path

from app.config import TZ

MJ_H_TO_W = 1e6 / 3600


@dataclass(frozen=True)
class Obs:
    observed_at: datetime
    ta: float
    rh: float
    wind: float
    ghi: float


def load_csv(path: Path) -> dict[datetime, Obs]:
    out = {}
    with path.open() as f:
        for r in csv.DictReader(f):
            if not (r["ta"] and r["hm"] and r["ws"]):
                continue
            t = datetime.strptime(r["tm"], "%Y-%m-%d %H:%M").replace(tzinfo=TZ)
            out[t] = Obs(t, float(r["ta"]), float(r["hm"]), float(r["ws"]), float(r["icsr"] or 0) * MJ_H_TO_W)
    return out


class Weather:
    def __init__(self, obs: dict[datetime, Obs]):
        self.obs = obs

    def at(self, dt: datetime) -> Obs | None:
        hour = dt.astimezone(TZ).replace(minute=0, second=0, microsecond=0)
        for back in range(4):
            if (o := self.obs.get(hour - timedelta(hours=back))) is not None:
                return o
        return None

    def add(self, items: list[Obs]) -> None:
        for o in items:
            self.obs[o.observed_at] = o
```

`backend/app/main.py`의 lifespan에 한 줄 추가(import 포함):
```python
from app import weather
...
    app.state.weather = weather.Weather(weather.load_csv(DATA_DIR / "kma_asos_131_hourly.csv"))
```

- [ ] **Step 5: 통과 확인**

Run: `cd backend && pytest -q`
Expected: 모두 PASS

- [ ] **Step 6: 커밋**

```bash
git add backend scripts/fetch_kma.py data/kma_asos_131_hourly.csv
git commit -m "feat: add Cheongju ASOS weather snapshot and lookup (#15)"
```

---

### Task 6: 체감온도 1단계 (#17)

**Files:**
- Create: `backend/app/climate.py`, `backend/tests/test_climate.py`, `backend/tests/conftest.py`
- Modify: `backend/app/api.py`

**Interfaces:**
- Consumes: `shadow.shadows`, `sun.position`, `weather.Weather/Obs`, `store.Store`, `geo.m_to_lonlat`
- Produces:
  - `climate.Grid(x0: float, y0: float, cell: float, cols: int, rows: int)` (x0, y0 = 북서쪽 모서리, EPSG:5179)
  - `climate.make_grid(area_m) -> Grid`
  - `climate.cell_of(grid, xs: np.ndarray, ys: np.ndarray) -> np.ndarray` (평탄 인덱스, 격자 밖은 -1)
  - `climate.erbs(ghi: float, alt: float) -> tuple[float, float]` (DNI, DHI W/m²)
  - `climate.mrt(ta, sunlit: np.ndarray, dni, dhi, ghi, alt) -> np.ndarray`
  - `climate.Climate(grid, sunlit: np.ndarray, mrt: np.ndarray | None, utci: np.ndarray | None, weather: Obs | None, alt: float, az: float)` (배열 길이 rows*cols, 건물·범위 밖은 nan)
  - `climate.compute(store, weather, dt) -> Climate` (10분 캐시)
  - `climate.to_json(c: Climate) -> dict`
  - `GET /api/climate?datetime=`

MRT는 스펙의 SolarCal(`solar_gain`, 실내용 모델) 대신 SOLWEIG 방식의 단순식을 쓴다: 사람이 받는 단파 흡수량을 장파 평형에 더한다. 1단계 가정은 주변 장파 = 기온의 흑체 복사.
`Tmrt = ((Ta+273.15)^4 + ak·(fp·DNI·sunlit + 0.5·DHI + 0.5·albedo·GHI) / (εp·σ))^0.25 − 273.15`, `ak=0.7, εp=0.97, albedo=0.15, fp = 0.308·cos(alt·(1 − alt²/48402))` (alt는 도 단위, SOLWEIG 서 있는 사람).

- [ ] **Step 1: 실패하는 테스트**

`backend/tests/test_climate.py`:
```python
import json
import math
from datetime import datetime

import numpy as np
from fastapi.testclient import TestClient
from shapely.geometry import box

from app import climate
from app.config import TZ
from app.store import Building, Store
from app.weather import Obs, Weather

AREA = box(958000, 1946000, 958100, 1946100)  # 100m × 100m → 10×10 셀


def test_grid_shape_and_cell_index():
    g = climate.make_grid(AREA)
    assert (g.cols, g.rows) == (10, 10)
    idx = climate.cell_of(g, np.array([958005.0, 958095.0, 957000.0]), np.array([1946095.0, 1946005.0, 0.0]))
    assert idx.tolist() == [0, 99, -1]  # 북서쪽 셀 0, 남동쪽 셀 99


def test_erbs_conserves_ghi():
    dni, dhi = climate.erbs(800, 60)
    assert abs(dni * math.sin(math.radians(60)) + dhi - 800) < 1e-6


def test_erbs_low_sun_is_capped():
    dni, dhi = climate.erbs(100, 1)
    assert dni <= 1100


def test_erbs_night_is_zero():
    assert climate.erbs(0, -5) == (0.0, 0.0)


def test_sun_hotter_than_shade():
    m = climate.mrt(30, np.array([1.0, 0.0]), 700, 100, 700, 60)
    assert m[0] > m[1] > 30


HOUR = datetime(2026, 7, 1, 12, 0, tzinfo=TZ)
NOON = datetime(2026, 7, 1, 12, 30, tzinfo=TZ)  # 청주 남중 ≈ 12:34, 해가 거의 정남


def store_with_building():
    # 100m 건물 → 한여름 남중 고도(~76°)에서 그림자 약 25m가 북쪽으로
    b = Building("a", 100.0, box(958040, 1946040, 958060, 1946060), {})
    return Store(area_m=AREA, buildings=[b])


def weather():
    return Weather({HOUR: Obs(HOUR, 30.0, 50.0, 1.5, 800.0)})


def test_compute_shade_building_and_utci():
    c = climate.compute(store_with_building(), weather(), NOON)
    g = c.grid
    center = 5 * g.cols + 5
    assert math.isnan(c.sunlit[center])  # 건물 안
    north = 2 * g.cols + 5  # 건물 북쪽 = 그림자
    south = 8 * g.cols + 5
    assert c.sunlit[north] == 0 and c.sunlit[south] == 1
    assert c.utci[south] > c.utci[north]


def test_compute_without_weather_has_no_utci():
    t = datetime(2027, 7, 1, 12, 0, tzinfo=TZ)
    c = climate.compute(store_with_building(), Weather({}), t)
    assert c.utci is None and c.mrt is None and c.weather is None


def test_climate_json_has_null_not_nan():
    c = climate.compute(store_with_building(), weather(), NOON)
    c.utci[0] = np.nan
    text = json.dumps(climate.to_json(c), allow_nan=False)  # NaN이 있으면 ValueError
    assert '"utci": [null' in text


def test_climate_api():
    from app.main import app
    app.state.store = store_with_building()
    app.state.weather = weather()
    body = TestClient(app).get("/api/climate", params={"datetime": "2026-07-01T12:30:00"}).json()
    assert body["grid"]["cols"] == 10 and len(body["utci"]) == 100 and len(body["grid"]["corners"]) == 4
```

`compute`는 `(id(store), 10분 단위 시각)`으로 캐시하는데, 테스트마다 새로 만든 Store가 이전 객체의 id를 재사용할 수 있다. 캐시를 테스트마다 비운다.

`backend/tests/conftest.py`:
```python
import pytest


@pytest.fixture(autouse=True)
def clear_climate_cache():
    from app import climate
    climate._cache.clear()
```

- [ ] **Step 2: 실패 확인**

Run: `cd backend && pytest tests/test_climate.py -q`
Expected: FAIL (`ModuleNotFoundError: app.climate`)

- [ ] **Step 3: 구현**

`backend/app/climate.py`:
```python
import math
from dataclasses import dataclass
from datetime import datetime

import numpy as np
import shapely
from pythermalcomfort.models import utci as utci_model

from app import shadow, sun
from app.config import CELL_M
from app.geo import m_to_lonlat
from app.weather import Obs

SIGMA, AK, EP, ALBEDO = 5.67e-8, 0.7, 0.97, 0.15
OFFSETS = np.array([-4.0, -2.0, 0.0, 2.0, 4.0])  # 10m 셀 안 2m 간격 5×5 점
_cache: dict = {}


@dataclass
class Grid:
    x0: float
    y0: float
    cell: float
    cols: int
    rows: int


@dataclass
class Climate:
    grid: Grid
    sunlit: np.ndarray
    mrt: np.ndarray | None
    utci: np.ndarray | None
    weather: Obs | None
    alt: float
    az: float


def make_grid(area_m) -> Grid:
    minx, miny, maxx, maxy = area_m.bounds
    return Grid(minx, maxy, CELL_M, math.ceil((maxx - minx) / CELL_M), math.ceil((maxy - miny) / CELL_M))


def centers(g: Grid) -> tuple[np.ndarray, np.ndarray]:
    r, c = np.divmod(np.arange(g.rows * g.cols), g.cols)
    return g.x0 + (c + 0.5) * g.cell, g.y0 - (r + 0.5) * g.cell


def cell_of(g: Grid, xs: np.ndarray, ys: np.ndarray) -> np.ndarray:
    c = np.floor((xs - g.x0) / g.cell).astype(int)
    r = np.floor((g.y0 - ys) / g.cell).astype(int)
    ok = (c >= 0) & (c < g.cols) & (r >= 0) & (r < g.rows)
    return np.where(ok, r * g.cols + c, -1)


def erbs(ghi: float, alt: float) -> tuple[float, float]:
    if alt <= 0 or ghi <= 0:
        return 0.0, 0.0
    sin_a = math.sin(math.radians(alt))
    kt = min(ghi / (1367 * sin_a), 1.0)
    if kt <= 0.22:
        kd = 1 - 0.09 * kt
    elif kt <= 0.80:
        kd = 0.9511 - 0.1604 * kt + 4.388 * kt**2 - 16.638 * kt**3 + 12.336 * kt**4
    else:
        kd = 0.165
    dhi = kd * ghi
    return min((ghi - dhi) / sin_a, 1100.0), dhi  # ponytail: 낮은 해에서 DNI 폭주 방지용 상한


def mrt(ta, sunlit, dni, dhi, ghi, alt):
    fp = 0.308 * math.cos(math.radians(alt * (1 - alt**2 / 48402))) if alt > 0 else 0.0
    k_abs = AK * (fp * dni * sunlit + 0.5 * dhi + 0.5 * ALBEDO * ghi)
    return ((ta + 273.15) ** 4 + k_abs / (EP * SIGMA)) ** 0.25 - 273.15


def _utci(ta, tr, wind, rh):
    res = utci_model(tdb=np.full_like(tr, ta), tr=tr, v=np.full_like(tr, min(max(wind, 0.5), 17)),
                     rh=np.full_like(tr, rh), limit_inputs=True)
    return np.asarray(getattr(res, "utci", res), dtype=float)


def _sunlit(store, g: Grid, alt: float, az: float) -> np.ndarray:
    cx, cy = centers(g)
    blocked = ~shapely.contains_xy(store.area_m, cx, cy)
    bunion = shapely.union_all([b.geom_m for b in store.buildings])
    blocked |= shapely.contains_xy(bunion, cx, cy)
    if alt <= 0:
        out = np.zeros(cx.size)
    else:
        shade = shadow.shadows(store.buildings, alt, az)
        shapely.prepare(shade)
        px = (cx[:, None, None] + OFFSETS[None, :, None]).repeat(5, 2).reshape(cx.size, 25)
        py = (cy[:, None, None] + OFFSETS[None, None, :]).repeat(5, 1).reshape(cy.size, 25)
        out = 1 - shapely.contains_xy(shade, px, py).mean(axis=1)
    out[blocked] = np.nan
    return out


def compute(store, weather, dt: datetime) -> Climate:
    key = (id(store), dt.replace(minute=dt.minute // 10 * 10, second=0, microsecond=0))
    if key in _cache:
        return _cache[key]
    g = make_grid(store.area_m)
    alt, az = sun.position(dt)
    sunlit = _sunlit(store, g, alt, az)
    obs = weather.at(dt)
    m = u = None
    if obs is not None:
        dni, dhi = erbs(obs.ghi, alt)
        m = mrt(obs.ta, sunlit, dni, dhi, obs.ghi, alt)
        u = _utci(obs.ta, m, obs.wind, obs.rh)
    _cache[key] = c = Climate(g, sunlit, m, u, obs, alt, az)
    return c


def _nulls(a):
    return None if a is None else [None if math.isnan(v) else round(float(v), 2) for v in a]


def to_json(c: Climate) -> dict:
    g = c.grid
    x1, y1 = g.x0 + g.cols * g.cell, g.y0 - g.rows * g.cell
    corners = [m_to_lonlat(g.x0, g.y0), m_to_lonlat(x1, g.y0), m_to_lonlat(x1, y1), m_to_lonlat(g.x0, y1)]
    w = c.weather
    return {
        "grid": {"cols": g.cols, "rows": g.rows, "cell_m": g.cell, "corners": [[lon, lat] for lon, lat in corners]},
        "altitude": c.alt, "azimuth": c.az,
        "weather": None if w is None else {"observed_at": w.observed_at.isoformat(), "ta": w.ta, "rh": w.rh,
                                           "wind": w.wind, "ghi": w.ghi},
        "sunlit": _nulls(c.sunlit), "mrt": _nulls(c.mrt), "utci": _nulls(c.utci),
    }
```

`corners` 순서는 북서, 북동, 남동, 남서(MapLibre `LatLngQuad`의 topLeft, topRight, bottomRight, bottomLeft)다.

`backend/app/api.py`에 추가:
```python
from app import climate


@router.get("/api/climate")
def get_climate(request: Request, datetime: str | None = Query(None)):
    s = request.app.state
    return climate.to_json(climate.compute(s.store, s.weather, parse_dt(datetime)))
```

- [ ] **Step 4: 통과 확인**

Run: `cd backend && pytest -q`
Expected: 모두 PASS. `_utci`에서 pythermalcomfort 버전 차이로 인자 이름 오류가 나면 `python -c "from pythermalcomfort.models import utci; help(utci)"`로 시그니처를 확인하고 인자 이름만 맞춘다.

- [ ] **Step 5: 커밋**

```bash
git add backend
git commit -m "feat: compute stage-1 UTCI grid from shade and weather (#17)"
```

---

### Task 7: 기상청 API허브 실시간 (#15 2부)

**Files:**
- Modify: `backend/app/weather.py`, `backend/app/main.py`, `backend/tests/test_weather.py`

**Interfaces:**
- Produces: `weather.parse_apihub(text: str) -> list[Obs]`, `weather.refresh_loop(w: Weather, key: str)` (async, 5분 주기)

API허브 지상관측 시간자료(`kma_sfctm3.php`) 응답은 `#`으로 시작하는 주석 줄 중 `YYMMDDHHMI`로 시작하는 줄에 열 이름이 있고, 이후 공백 구분 값 줄이 온다. 결측은 음수(-9, -99 등). 열은 이름으로 찾는다(`TA` 기온, `HM` 습도, `WS` 풍속, `SI` 일사 MJ/m²).

- [ ] **Step 1: 실패하는 테스트**

`backend/tests/test_weather.py`에 추가:
```python
from app.weather import parse_apihub

SAMPLE = """#START7777
#--------------------------------
#  YYMMDDHHMI STN  WD   WS    TA    HM    SI
#        KST  ID  16  m/s     C     %  MJ/m2
202607011200 131  20  2.1  30.4  58.0  3.10
202607011300 131  20  2.3  -99.0  55.0  3.00
202607012000 131  20  1.0  26.0  70.0  -9.0
#7777END
"""


def test_parse_apihub():
    obs = {o.observed_at.hour: o for o in parse_apihub(SAMPLE)}
    assert obs[12].ta == 30.4 and obs[12].rh == 58.0 and abs(obs[12].ghi - 3.1e6 / 3600) < 1e-6
    assert 13 not in obs  # 기온 결측
    assert obs[20].ghi == 0  # 일사 결측(밤) → 0
```

- [ ] **Step 2: 실패 확인**

Run: `cd backend && pytest tests/test_weather.py -q`
Expected: FAIL (`ImportError: parse_apihub`)

- [ ] **Step 3: 구현**

`backend/app/weather.py`에 추가:
```python
import asyncio
import logging

import httpx

APIHUB_URL = "https://apihub.kma.go.kr/api/typ01/url/kma_sfctm3.php"
log = logging.getLogger(__name__)


def parse_apihub(text: str) -> list[Obs]:
    cols, out = None, []
    for line in text.splitlines():
        if line.startswith("#"):
            parts = line.lstrip("#").split()
            if parts and parts[0] == "YYMMDDHHMI":
                cols = {name: i for i, name in reversed(list(enumerate(parts)))}  # 중복 이름은 첫 열
            continue
        v = line.split()
        if not cols or len(v) < len(cols):
            continue
        ta, rh, ws, si = (float(v[cols[k]]) for k in ("TA", "HM", "WS", "SI"))
        if ta <= -50 or rh < 0 or ws < 0:
            continue
        t = datetime.strptime(v[0], "%Y%m%d%H%M").replace(tzinfo=TZ)
        out.append(Obs(t, ta, rh, ws, max(si, 0) * MJ_H_TO_W))
    return out


async def refresh_loop(w: Weather, key: str, every_s: int = 300) -> None:
    async with httpx.AsyncClient(timeout=10) as client:
        while True:
            now = datetime.now(TZ)
            params = {"tm1": (now - timedelta(hours=3)).strftime("%Y%m%d%H00"), "tm2": now.strftime("%Y%m%d%H%M"),
                      "stn": "131", "help": "1", "authKey": key}
            try:
                r = await client.get(APIHUB_URL, params=params)
                r.raise_for_status()
                w.add(parse_apihub(r.text))
            except httpx.HTTPError as e:
                log.warning("apihub refresh failed: %s", e)
            await asyncio.sleep(every_s)
```

`help=1`은 열 이름 주석 줄을 포함시키는 옵션이다.

`backend/app/main.py`의 lifespan을 다음으로:
```python
import asyncio

from app.config import DATA_DIR, KMA_APIHUB_KEY


@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.store = store.load(DATA_DIR)
    app.state.weather = weather.Weather(weather.load_csv(DATA_DIR / "kma_asos_131_hourly.csv"))
    task = asyncio.create_task(weather.refresh_loop(app.state.weather, KMA_APIHUB_KEY)) if KMA_APIHUB_KEY else None
    yield
    if task:
        task.cancel()
```

- [ ] **Step 4: 통과 확인**

Run: `cd backend && pytest -q`
Expected: 모두 PASS

- [ ] **Step 5: 실제 응답으로 확인**

```bash
set -a; source .env; set +a
curl -s "https://apihub.kma.go.kr/api/typ01/url/kma_sfctm3.php?tm1=$(date +%Y%m%d)0900&tm2=$(date +%Y%m%d)1200&stn=131&help=1&authKey=$KMA_APIHUB_KEY" | head -20
```
Expected: `# YYMMDDHHMI …` 열 이름 줄에 `TA`, `HM`, `WS`, `SI`가 있다. 이름이 다르면(예: 첫 열 이름) `parse_apihub`의 열 이름만 실제 응답에 맞추고 `SAMPLE`도 같이 고친다.

- [ ] **Step 6: 커밋**

```bash
git add backend
git commit -m "feat: refresh live weather from KMA API hub (#15)"
```

---

### Task 8: 경로 탐색 (#19)

**Files:**
- Create: `backend/app/route.py`, `backend/tests/test_route.py`
- Modify: `backend/app/store.py`, `backend/app/api.py`

**Interfaces:**
- Consumes: `climate.Climate`, `climate.Grid`, `climate.cell_of`, `climate.compute`, `geo.lonlat_to_m`, `geo.m_to_lonlat`
- Produces:
  - `route.load_graph(path: Path) -> nx.MultiDiGraph` (EPSG:5179로 투영)
  - `route.index_edges(G, grid: Grid) -> dict[tuple, np.ndarray]` (구간별 5m 샘플의 셀 인덱스)
  - `route.OutOfArea(Exception)`
  - `route.plan(G, edge_idx, c: Climate, area_m, start: tuple[float,float], end: tuple[float,float], k=K_COMFORT) -> dict` (start/end는 (lat, lon))
  - 응답: `{"basis": "utci"|"sunlit", "comfort": R, "shortest": R}`, `R = {"coords": [[lon,lat],…], "distance_m", "sun_ratio", "mean_utci"}`
  - `GET /api/route?from=lat,lon&to=lat,lon&datetime=`

- [ ] **Step 1: 실패하는 테스트**

`backend/tests/test_route.py`:
```python
import networkx as nx
import numpy as np
import pytest
from shapely.geometry import LineString, box

from app.climate import Climate, Grid
from app.geo import m_to_lonlat
from app.route import OutOfArea, index_edges, plan

X0, Y0 = 958000.0, 1946000.0


def toy_graph():
    # A(0,0) - B(100,0) 직선 100m (양지), A - C(0,30) - D(100,30) - B 우회 160m (그늘)
    pts = {"A": (0, 0), "B": (100, 0), "C": (0, 30), "D": (100, 30)}
    G = nx.MultiDiGraph(crs="EPSG:5179")
    for n, (x, y) in pts.items():
        G.add_node(n, x=X0 + x, y=Y0 + y)
    for u, v in [("A", "B"), ("A", "C"), ("C", "D"), ("D", "B")]:
        (x1, y1), (x2, y2) = pts[u], pts[v]
        line = LineString([(X0 + x1, Y0 + y1), (X0 + x2, Y0 + y2)])
        G.add_edge(u, v, 0, length=line.length, geometry=line)
        G.add_edge(v, u, 0, length=line.length, geometry=LineString(line.coords[::-1]))
    return G


def toy_climate(with_utci):
    g = Grid(X0 - 10, Y0 + 50, 10, 13, 7)  # x: -10~120, y: 50~-20
    r, _ = np.divmod(np.arange(g.rows * g.cols), g.cols)
    ys = g.y0 - (r + 0.5) * g.cell
    sunlit = np.where(ys < Y0 + 15, 1.0, 0.0)
    utci = np.where(ys < Y0 + 15, 38.0, 30.0) if with_utci else None
    return Climate(g, sunlit, None, utci, None, 60, 180)


def ll(x, y):
    lon, lat = m_to_lonlat(X0 + x, Y0 + y)
    return lat, lon


AREA = box(X0 - 10, Y0 - 20, X0 + 120, Y0 + 50)


@pytest.mark.parametrize("with_utci", [True, False])
def test_comfort_takes_shaded_detour(with_utci):
    G, c = toy_graph(), toy_climate(with_utci)
    r = plan(G, index_edges(G, c.grid), c, AREA, ll(0, 0), ll(100, 0))
    assert r["basis"] == ("utci" if with_utci else "sunlit")
    assert abs(r["shortest"]["distance_m"] - 100) < 1e-6
    assert abs(r["comfort"]["distance_m"] - 160) < 1e-6
    assert r["comfort"]["sun_ratio"] < r["shortest"]["sun_ratio"]
    assert (r["comfort"]["mean_utci"] is None) == (not with_utci)


def test_same_point_zero_length():
    G, c = toy_graph(), toy_climate(False)
    r = plan(G, index_edges(G, c.grid), c, AREA, ll(0, 0), ll(1, 1))
    assert r["shortest"]["distance_m"] == 0 and len(r["comfort"]["coords"]) == 1


def test_out_of_area():
    G, c = toy_graph(), toy_climate(False)
    with pytest.raises(OutOfArea):
        plan(G, index_edges(G, c.grid), c, AREA, ll(0, 0), ll(5000, 0))
```

- [ ] **Step 2: 실패 확인**

Run: `cd backend && pytest tests/test_route.py -q`
Expected: FAIL (`ModuleNotFoundError: app.route`)

- [ ] **Step 3: 구현**

`backend/app/route.py`:
```python
from pathlib import Path

import networkx as nx
import numpy as np
import osmnx as ox
from shapely.geometry import LineString, Point

from app.climate import Climate, Grid, cell_of
from app.config import CRS_M, K_COMFORT, SAMPLE_M
from app.geo import lonlat_to_m, m_to_lonlat


class OutOfArea(Exception):
    pass


def load_graph(path: Path) -> nx.MultiDiGraph:
    return ox.project_graph(ox.load_graphml(path), to_crs=CRS_M)


def _line(G, u, v, d) -> LineString:
    if "geometry" in d:
        return d["geometry"]
    return LineString([(G.nodes[u]["x"], G.nodes[u]["y"]), (G.nodes[v]["x"], G.nodes[v]["y"])])


def index_edges(G, grid: Grid) -> dict[tuple, np.ndarray]:
    out = {}
    for u, v, k, d in G.edges(keys=True, data=True):
        line = _line(G, u, v, d)
        ds = np.arange(0, line.length + 1e-9, SAMPLE_M) if line.length > 0 else np.array([0.0])
        pts = [line.interpolate(x) for x in ds]
        out[(u, v, k)] = cell_of(grid, np.array([p.x for p in pts]), np.array([p.y for p in pts]))
    return out


def _mean(values: np.ndarray, idx: np.ndarray) -> float:
    vals = values[idx[idx >= 0]]
    vals = vals[~np.isnan(vals)]
    return float(vals.mean()) if vals.size else float("nan")


def _nearest(G, x: float, y: float):
    # ponytail: 노드 수천 개라 선형 탐색. 느려지면 scipy KDTree
    ids = list(G.nodes)
    xs = np.array([G.nodes[n]["x"] for n in ids])
    ys = np.array([G.nodes[n]["y"] for n in ids])
    return ids[int(np.argmin((xs - x) ** 2 + (ys - y) ** 2))]


def plan(G, edge_idx, c: Climate, area_m, start, end, k: float = K_COMFORT) -> dict:
    pts = [lonlat_to_m(lon, lat) for lat, lon in (start, end)]
    if not all(area_m.contains(Point(p)) for p in pts):
        raise OutOfArea
    basis = "utci" if c.utci is not None else "sunlit"
    heat = c.utci if c.utci is not None else c.sunlit
    lo, hi = np.nanmin(heat), np.nanmax(heat)
    sun_e = {e: _mean(c.sunlit, i) for e, i in edge_idx.items()}
    heat_e = {e: _mean(heat, i) for e, i in edge_idx.items()}

    def w(e):
        h = heat_e[e]
        return 0.0 if np.isnan(h) or hi <= lo else (h - lo) / (hi - lo)

    comfort = {e: G.edges[e]["length"] * (1 + k * w(e)) for e in edge_idx}
    s, t = (_nearest(G, *p) for p in pts)

    def summarize(path, cost):
        coords = [m_to_lonlat(G.nodes[path[0]]["x"], G.nodes[path[0]]["y"])]
        dist = sun_len = utci_len = 0.0
        for u, v in zip(path, path[1:]):
            key = min(G[u][v], key=lambda kk: cost[(u, v, kk)])
            e, d = (u, v, key), G[u][v][key]
            coords += [m_to_lonlat(x, y) for x, y in list(_line(G, u, v, d).coords)[1:]]
            dist += d["length"]
            sun_len += d["length"] * np.nan_to_num(sun_e[e])
            if c.utci is not None:
                utci_len += d["length"] * np.nan_to_num(heat_e[e])
        return {"coords": [[lon, lat] for lon, lat in coords], "distance_m": dist,
                "sun_ratio": 100 * sun_len / dist if dist else 0.0,
                "mean_utci": (utci_len / dist if dist else None) if c.utci is not None else None}

    length = {e: G.edges[e]["length"] for e in edge_idx}
    by = lambda cost: lambda u, v, d: min(cost[(u, v, kk)] for kk in d)  # noqa: E731
    return {"basis": basis,
            "comfort": summarize(nx.shortest_path(G, s, t, weight=by(comfort)), comfort),
            "shortest": summarize(nx.shortest_path(G, s, t, weight=by(length)), length)}
```

`test_same_point_zero_length`에서 `mean_utci`는 utci가 없어 None이고, 거리 0이면 `sun_ratio` 0이다.

`backend/app/store.py`의 `load` 끝부분을 다음으로 바꾼다(그래프는 파일이 있을 때만 — 테스트 픽스처에는 없다):
```python
    s = Store(area_m=to_m(boundary).buffer(BUFFER_M), buildings=buildings)
    graph_path = data_dir / "walk_graph.graphml"
    if graph_path.exists():
        from app.climate import make_grid
        from app.route import index_edges, load_graph
        s.graph = load_graph(graph_path)
        s.edge_idx = index_edges(s.graph, make_grid(s.area_m))
    return s
```

`backend/app/api.py`에 추가:
```python
from fastapi import HTTPException

from app import route


def _latlon(s: str) -> tuple[float, float]:
    try:
        lat, lon = (float(v) for v in s.split(","))
    except ValueError:
        raise HTTPException(422, f"expected lat,lon: {s}")
    return lat, lon


@router.get("/api/route")
def get_route(request: Request, from_: str = Query(alias="from"), to: str = Query(),
              datetime: str | None = Query(None)):
    s = request.app.state
    c = climate.compute(s.store, s.weather, parse_dt(datetime))
    try:
        return route.plan(s.store.graph, s.store.edge_idx, c, s.store.area_m, _latlon(from_), _latlon(to))
    except route.OutOfArea:
        raise HTTPException(400, "outside service area")
```

- [ ] **Step 4: 통과 확인**

Run: `cd backend && pytest -q`
Expected: 모두 PASS

- [ ] **Step 5: 성능 측정 스크립트**

`scripts/bench.py`:
```python
"""실제 데이터로 /api/climate, /api/route 계산 시간 측정"""
import time
from datetime import datetime

from app import climate, route, store
from app.config import DATA_DIR, TZ
from app.geo import m_to_lonlat
from app.weather import Weather, load_csv

s = store.load(DATA_DIR)
w = Weather(load_csv(DATA_DIR / "kma_asos_131_hourly.csv"))
dt = max(w.obs).replace(tzinfo=TZ) if w.obs else datetime.now(TZ)
t0 = time.perf_counter()
c = climate.compute(s, w, dt)
t1 = time.perf_counter()
nodes = list(s.graph.nodes)
a, b = (m_to_lonlat(s.graph.nodes[n]["x"], s.graph.nodes[n]["y"]) for n in (nodes[0], nodes[len(nodes) // 2]))
route.plan(s.graph, s.edge_idx, c, s.area_m, (a[1], a[0]), (b[1], b[0]))
t2 = time.perf_counter()
print(f"climate {t1 - t0:.2f}s, route {t2 - t1:.2f}s, cells {c.grid.rows * c.grid.cols}, edges {len(s.edge_idx)}")
```

Run: `PYTHONPATH=backend python scripts/bench.py`
Expected: `climate X.XXs, route Y.YYs …`. 각각 1초를 넘으면 결과를 이슈 #17, #19에 댓글로 기록하고, `_sunlit`의 점 수(25 → 9)나 `summarize`의 샘플 간격을 먼저 줄여본다.

- [ ] **Step 6: 커밋**

```bash
git add backend scripts/bench.py
git commit -m "feat: add comfort and shortest route API (#19)"
```

---

### Task 9: 장소 검색 (#22 서버)

**Files:**
- Create: `backend/app/places.py`, `backend/tests/test_places.py`, `data/places.json`
- Modify: `backend/app/store.py`, `backend/app/api.py`

**Interfaces:**
- Produces:
  - `places.search(q: str, places: list[dict], area_m, key: str, client: httpx.Client) -> dict` → `{"places": [{"name","lat","lon"}…], "kakao_error": bool}`
  - `GET /api/places?q=`

- [ ] **Step 1: 장소 목록**

`data/places.json` (좌표는 팀이 지도에서 확인해 고친다. 아래는 형식 예시이며 실제 좌표로 교체 필수):
```json
[
  {"name": "중앙도서관", "lat": 36.6284, "lon": 127.4575},
  {"name": "정문", "lat": 36.6320, "lon": 127.4553}
]
```

- [ ] **Step 2: 실패하는 테스트**

`backend/tests/test_places.py`:
```python
import httpx
from shapely.geometry import box

from app.geo import lonlat_to_m
from app.places import search

PLACES = [{"name": "중앙도서관", "lat": 36.6284, "lon": 127.4575}, {"name": "정문", "lat": 36.632, "lon": 127.4553}]
x, y = lonlat_to_m(127.4575, 36.6284)
AREA = box(x - 1000, y - 1000, x + 1000, y + 1000)


def kakao(handler):
    return httpx.Client(transport=httpx.MockTransport(handler))


def test_local_only_without_key():
    r = search("도서관", PLACES, AREA, "", kakao(lambda req: httpx.Response(500)))
    assert [p["name"] for p in r["places"]] == ["중앙도서관"] and r["kakao_error"] is False


def test_kakao_results_filtered_to_area():
    def handler(req):
        assert req.headers["Authorization"] == "KakaoAK k"
        return httpx.Response(200, json={"documents": [
            {"place_name": "충북대 학생회관", "x": "127.4570", "y": "36.6290"},
            {"place_name": "서울역", "x": "126.9707", "y": "37.5547"}]})
    r = search("학생회관", PLACES, AREA, "k", kakao(handler))
    assert [p["name"] for p in r["places"]] == ["충북대 학생회관"]


def test_kakao_failure_flagged():
    r = search("도서관", PLACES, AREA, "k", kakao(lambda req: httpx.Response(500)))
    assert r["kakao_error"] is True and r["places"][0]["name"] == "중앙도서관"


def test_empty_query_returns_all_local():
    assert len(search("", PLACES, AREA, "k", kakao(lambda req: httpx.Response(500)))["places"]) == 2
```

- [ ] **Step 3: 실패 확인**

Run: `cd backend && pytest tests/test_places.py -q`
Expected: FAIL (`ModuleNotFoundError: app.places`)

- [ ] **Step 4: 구현**

`backend/app/places.py`:
```python
import httpx
from shapely.geometry import Point

from app.config import CAMPUS_LAT, CAMPUS_LON
from app.geo import lonlat_to_m

KAKAO_URL = "https://dapi.kakao.com/v2/local/search/keyword.json"


def search(q: str, places: list[dict], area_m, key: str, client: httpx.Client) -> dict:
    out = [p for p in places if q in p["name"]]
    error = False
    if q and key:
        try:
            r = client.get(KAKAO_URL, headers={"Authorization": f"KakaoAK {key}"}, timeout=3,
                           params={"query": q, "x": CAMPUS_LON, "y": CAMPUS_LAT, "radius": 2000})
            r.raise_for_status()
            for d in r.json()["documents"]:
                lon, lat = float(d["x"]), float(d["y"])
                if area_m.contains(Point(lonlat_to_m(lon, lat))):
                    out.append({"name": d["place_name"], "lat": lat, "lon": lon})
        except httpx.HTTPError:
            error = True
    return {"places": out, "kakao_error": error}
```

`backend/app/store.py`의 `load`에서 `return s` 바로 앞에 추가:
```python
    places_path = data_dir / "places.json"
    if places_path.exists():
        s.places = json.loads(places_path.read_text())
```

`backend/app/api.py`에 추가:
```python
import httpx

from app import places
from app.config import KAKAO_REST_KEY

_http = httpx.Client()


@router.get("/api/places")
def get_places(request: Request, q: str = ""):
    s = request.app.state.store
    return places.search(q.strip(), s.places, s.area_m, KAKAO_REST_KEY, _http)
```

- [ ] **Step 5: 통과 확인**

Run: `cd backend && pytest -q`
Expected: 모두 PASS

- [ ] **Step 6: 커밋**

```bash
git add backend data/places.json
git commit -m "feat: add campus places and Kakao search proxy (#22)"
```

---

### Task 10: 앱 뼈대와 서버 연결 (#12)

**Files:**
- Create: Android Studio 프로젝트 `android/`, `android/app/src/main/java/kr/ac/cbnu/shademap/Api.kt`, `android/app/src/debug/res/xml/network_security_config.xml`
- Modify: `android/app/build.gradle.kts`, `android/app/src/main/AndroidManifest.xml`, `android/app/src/main/java/kr/ac/cbnu/shademap/MainActivity.kt`, `android/app/src/main/res/layout/activity_main.xml`, `README.md`

**Interfaces:**
- Produces:
  - `Api(baseUrl: String)` with `fun get(path: String, params: Map<String, String> = emptyMap(), onOk: (JSONObject) -> Unit, onErr: (String) -> Unit)` (콜백은 메인 스레드)
  - `MainActivity.map: MapLibreMap`, `MainActivity.style: Style`, `MainActivity.api: Api`

- [ ] **Step 1: 프로젝트 생성 (사람이 하는 단계)**

Android Studio → New Project → **Empty Views Activity**
- Name `ShadeMap`, Package `kr.ac.cbnu.shademap`, Save location `<저장소>/android`, Language Kotlin, Minimum SDK API 26, Build configuration language Kotlin DSL.

- [ ] **Step 2: Gradle 설정**

`android/app/build.gradle.kts`의 `android {}`와 `dependencies {}`를 다음으로 맞춘다:
```kotlin
android {
    namespace = "kr.ac.cbnu.shademap"
    compileSdk = 35
    defaultConfig {
        applicationId = "kr.ac.cbnu.shademap"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "0.1.0"
        val apiBaseUrl = providers.gradleProperty("API_BASE_URL").orElse("http://localhost:8000/").get()
        buildConfigField("String", "API_BASE_URL", "\"$apiBaseUrl\"")
    }
    buildFeatures {
        buildConfig = true
        viewBinding = true
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
}

dependencies {
    implementation("androidx.core:core-ktx:1.16.0")
    implementation("androidx.appcompat:appcompat:1.7.1")
    implementation("com.google.android.material:material:1.12.0")
    implementation("org.maplibre.gl:android-sdk:11.8.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    testImplementation("junit:junit:4.13.2")
}
```

MapLibre 버전은 Maven Central의 `org.maplibre.gl:android-sdk` 최신 11.x(또는 참고 앱이 쓰는 `android-sdk-opengl:13.4.1`)로 바꿔도 된다. 패키지는 `org.maplibre.android.*`다.

- [ ] **Step 3: 매니페스트와 네트워크 정책**

`AndroidManifest.xml`의 `<manifest>` 안, `<application>` 앞:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```
`<application>`에 속성 추가: `android:networkSecurityConfig="@xml/network_security_config"`

`android/app/src/main/res/xml/network_security_config.xml` (release, localhost만 평문 허용):
```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">localhost</domain>
        <domain includeSubdomains="false">10.0.2.2</domain>
    </domain-config>
</network-security-config>
```

- [ ] **Step 4: Api.kt**

```kotlin
package kr.ac.cbnu.shademap

import android.os.Handler
import android.os.Looper
import okhttp3.Call
import okhttp3.Callback
import okhttp3.HttpUrl.Companion.toHttpUrl
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.Response
import org.json.JSONObject
import java.io.IOException

class Api(private val baseUrl: String) {
    private val client = OkHttpClient()
    private val main = Handler(Looper.getMainLooper())

    fun get(path: String, params: Map<String, String> = emptyMap(), onOk: (JSONObject) -> Unit, onErr: (String) -> Unit) {
        val url = baseUrl.toHttpUrl().newBuilder().addPathSegments(path).apply {
            params.forEach { (k, v) -> addQueryParameter(k, v) }
        }.build()
        client.newCall(Request.Builder().url(url).build()).enqueue(object : Callback {
            override fun onFailure(call: Call, e: IOException) {
                main.post { onErr("서버에 연결할 수 없습니다") }
            }

            override fun onResponse(call: Call, response: Response) {
                val body = response.use { it.body?.string().orEmpty() }
                main.post {
                    if (response.isSuccessful) onOk(JSONObject(body))
                    else onErr(if (response.code == 400) "충북대 주변만 지원합니다" else "서버 오류 (${response.code})")
                }
            }
        })
    }
}
```

- [ ] **Step 5: 레이아웃**

`activity_main.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent" android:layout_height="match_parent">

    <org.maplibre.android.maps.MapView
        android:id="@+id/mapView"
        android:layout_width="match_parent" android:layout_height="match_parent" />

    <TextView
        android:id="@+id/status"
        android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:layout_gravity="top|center_horizontal" android:layout_marginTop="48dp"
        android:background="#CC000000" android:padding="8dp" android:textColor="#FFFFFF"
        android:visibility="gone" />

    <TextView
        android:id="@+id/attribution"
        android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:layout_gravity="bottom|start" android:padding="4dp" android:textSize="10sp"
        android:text="© OpenStreetMap contributors · 기상청 · 국토교통부" />
</FrameLayout>
```

- [ ] **Step 6: MainActivity**

```kotlin
package kr.ac.cbnu.shademap

import android.os.Bundle
import android.view.View
import androidx.appcompat.app.AppCompatActivity
import kr.ac.cbnu.shademap.databinding.ActivityMainBinding
import org.maplibre.android.MapLibre
import org.maplibre.android.camera.CameraPosition
import org.maplibre.android.geometry.LatLng
import org.maplibre.android.maps.MapLibreMap
import org.maplibre.android.maps.Style

class MainActivity : AppCompatActivity() {
    lateinit var binding: ActivityMainBinding
    lateinit var map: MapLibreMap
    lateinit var style: Style
    val api = Api(BuildConfig.API_BASE_URL)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        MapLibre.getInstance(this)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        binding.mapView.onCreate(savedInstanceState)
        binding.mapView.getMapAsync { m ->
            map = m
            map.cameraPosition = CameraPosition.Builder().target(LatLng(36.6268, 127.4583)).zoom(15.5).tilt(45.0).build()
            map.setStyle(Style.Builder().fromUri(STYLE_URL)) { s ->
                style = s
                onStyleReady()
            }
        }
    }

    private fun onStyleReady() {
        api.get("health", onOk = { hideStatus() }, onErr = ::showStatus)
    }

    fun showStatus(msg: String) {
        binding.status.text = msg
        binding.status.visibility = View.VISIBLE
    }

    fun hideStatus() {
        binding.status.visibility = View.GONE
    }

    override fun onStart() { super.onStart(); binding.mapView.onStart() }
    override fun onResume() { super.onResume(); binding.mapView.onResume() }
    override fun onPause() { binding.mapView.onPause(); super.onPause() }
    override fun onStop() { binding.mapView.onStop(); super.onStop() }
    override fun onLowMemory() { super.onLowMemory(); binding.mapView.onLowMemory() }
    override fun onDestroy() { binding.mapView.onDestroy(); super.onDestroy() }
    override fun onSaveInstanceState(outState: Bundle) { super.onSaveInstanceState(outState); binding.mapView.onSaveInstanceState(outState) }

    companion object {
        const val STYLE_URL = "https://tiles.openfreemap.org/styles/liberty"
    }
}
```

- [ ] **Step 7: README 실기기 연결**

`README.md`에 추가:
````markdown
## 앱 실행 (Android)

1. Android Studio에서 `android/`를 열고 JDK 17로 Gradle Sync.
2. 서버를 LAN에 바인딩: `cd backend && uvicorn app.main:app --host 0.0.0.0 --port 8000`
3. 폰을 USB로 연결하고 `adb reverse tcp:8000 tcp:8000` (폰을 다시 꽂을 때마다 다시 실행)
4. Run. 서버 주소를 바꾸려면 `~/.gradle/gradle.properties`에 `API_BASE_URL=https://…/`
````

- [ ] **Step 8: 실기기 확인**

서버를 켜고 앱 실행 → 충북대 지도가 보이고 상단 오류 문구가 없다. 서버를 끄고 다시 실행 → "서버에 연결할 수 없습니다"가 보인다.

- [ ] **Step 9: 커밋**

```bash
git add android README.md
git commit -m "feat: add Android app skeleton with MapLibre map (#12)"
```

---

### Task 11: 3D 건물과 그림자 레이어 (#13 앱, #16 앱)

**Files:**
- Modify: `android/app/src/main/java/kr/ac/cbnu/shademap/MainActivity.kt`

**Interfaces:**
- Consumes: `GET /api/buildings`, `GET /api/shadows` (Task 2, 4)
- Produces: 레이어 id 상수 `SHADOW_LAYER = "shadows"`, `BUILDING_LAYER = "buildings"`, 함수 `addBelowLabels(layer)`, `refreshTimeLayers()` (Task 12가 확장)

- [ ] **Step 1: 레이어 추가**

`MainActivity.kt`에 import 추가:
```kotlin
import android.graphics.Color
import org.maplibre.android.style.expressions.Expression.coalesce
import org.maplibre.android.style.expressions.Expression.color
import org.maplibre.android.style.expressions.Expression.get
import org.maplibre.android.style.expressions.Expression.literal
import org.maplibre.android.style.expressions.Expression.switchCase
import org.maplibre.android.style.expressions.Expression.eq
import org.maplibre.android.style.expressions.Expression.typeOf
import org.maplibre.android.style.layers.FillExtrusionLayer
import org.maplibre.android.style.layers.FillLayer
import org.maplibre.android.style.layers.Layer
import org.maplibre.android.style.layers.Property
import org.maplibre.android.style.layers.PropertyFactory.*
import org.maplibre.android.style.layers.SymbolLayer
import org.maplibre.android.style.sources.GeoJsonSource
```

`onStyleReady()`를 다음으로:
```kotlin
    private fun onStyleReady() {
        style.addSource(GeoJsonSource(SHADOW_LAYER, EMPTY_FC))
        addBelowLabels(FillLayer(SHADOW_LAYER, SHADOW_LAYER).withProperties(
            fillColor(Color.rgb(51, 65, 85)), fillOpacity(0.55f)))
        style.addSource(GeoJsonSource(BUILDING_LAYER, EMPTY_FC))
        val hasHeight = eq(typeOf(get("height_m")), literal("number"))
        addBelowLabels(FillExtrusionLayer(BUILDING_LAYER, BUILDING_LAYER).withProperties(
            fillExtrusionHeight(coalesce(get("height_m"), literal(0f))),
            fillExtrusionColor(switchCase(hasHeight, color(Color.rgb(214, 137, 60)), color(Color.rgb(176, 176, 176)))),
            fillExtrusionOpacity(0.9f)))
        api.get("api/buildings", onOk = {
            style.getSourceAs<GeoJsonSource>(BUILDING_LAYER)?.setGeoJson(it.toString())
            val n = it.getJSONObject("counts").getInt("without_height")
            if (n > 0) showStatus("높이 정보 없는 건물 ${n}동은 그림자에서 제외")
        }, onErr = ::showStatus)
        refreshTimeLayers()
    }

    fun addBelowLabels(layer: Layer) {
        val firstLabel = style.layers.firstOrNull { it is SymbolLayer }
        if (firstLabel == null) style.addLayer(layer) else style.addLayerBelow(layer, firstLabel.id)
    }

    fun refreshTimeLayers() {
        // 해가 지면 서버가 빈 FeatureCollection을 주므로 그림자는 저절로 사라진다
        api.get("api/shadows", onOk = {
            style.getSourceAs<GeoJsonSource>(SHADOW_LAYER)?.setGeoJson(it.toString())
        }, onErr = ::showStatus)
    }
```

`companion object`에 추가:
```kotlin
        const val SHADOW_LAYER = "shadows"
        const val BUILDING_LAYER = "buildings"
        const val EMPTY_FC = """{"type":"FeatureCollection","features":[]}"""
```

- [ ] **Step 2: 실기기 확인**

서버 실행 + 앱 실행 → 캠퍼스 건물이 3D로 서고(높이 없는 건물은 회색 평면), 건물 그림자가 반투명하게 깔린다. 그림자 방향이 태양 반대쪽인지(오후라면 북동쪽) 눈으로 확인한다.

- [ ] **Step 3: 커밋**

```bash
git add android
git commit -m "feat: show 3D buildings and shadow layer in app (#13, #16)"
```

---

### Task 12: 체감온도 히트맵, 범례, 레이어 전환 (#18)

**Files:**
- Create: `android/app/src/main/java/kr/ac/cbnu/shademap/Heatmap.kt`, `android/app/src/test/java/kr/ac/cbnu/shademap/HeatmapTest.kt`
- Modify: `MainActivity.kt`, `activity_main.xml`

**Interfaces:**
- Consumes: `GET /api/climate` (Task 6): `grid.cols`, `grid.rows`, `grid.corners` [[lon,lat]×4, 북서·북동·남동·남서], `utci` (null 포함), `weather.observed_at`
- Produces: `Heatmap.utciColor(v: Double?): Int`, `Heatmap.pixels(values: List<Double?>): IntArray`, `Heatmap.BANDS`

- [ ] **Step 1: 실패하는 테스트**

`HeatmapTest.kt`:
```kotlin
package kr.ac.cbnu.shademap

import org.junit.Assert.assertEquals
import org.junit.Test

class HeatmapTest {
    @Test fun nullIsTransparent() = assertEquals(0, Heatmap.utciColor(null))

    @Test fun bandBoundaries() {
        assertEquals(Heatmap.BANDS[0].second, Heatmap.utciColor(25.9))
        assertEquals(Heatmap.BANDS[1].second, Heatmap.utciColor(26.0))
        assertEquals(Heatmap.BANDS[3].second, Heatmap.utciColor(38.0))
        assertEquals(Heatmap.BANDS[4].second, Heatmap.utciColor(46.0))
    }

    @Test fun pixelOrderMatchesGridNorthWestFirst() {
        val px = Heatmap.pixels(listOf(40.0, null, 20.0, 30.0))
        assertEquals(Heatmap.utciColor(40.0), px[0])  // 북서 = 비트맵 (0,0)
        assertEquals(0, px[1])
        assertEquals(Heatmap.utciColor(30.0), px[3])
    }
}
```

- [ ] **Step 2: 실패 확인**

Run: `cd android && ./gradlew testDebugUnitTest`
Expected: FAIL (`Unresolved reference: Heatmap`)

- [ ] **Step 3: 구현**

`Heatmap.kt`:
```kotlin
package kr.ac.cbnu.shademap

import android.graphics.Bitmap

object Heatmap {
    private fun argb(a: Int, r: Int, g: Int, b: Int) = (a shl 24) or (r shl 16) or (g shl 8) or b

    /** 하한(℃) 이상이면 해당 색. UTCI 열 스트레스 구간 */
    val BANDS: List<Pair<Double, Int>> = listOf(
        Double.NEGATIVE_INFINITY to argb(255, 102, 189, 99),  // 26 미만: 없음
        26.0 to argb(255, 254, 224, 139),                     // 보통
        32.0 to argb(255, 253, 174, 97),                      // 강함
        38.0 to argb(255, 244, 109, 67),                      // 매우 강함
        46.0 to argb(255, 165, 0, 38),                        // 극심
    )

    fun utciColor(v: Double?): Int = if (v == null) 0 else BANDS.last { v >= it.first }.second

    fun pixels(values: List<Double?>): IntArray = IntArray(values.size) { utciColor(values[it]) }

    fun bitmap(values: List<Double?>, cols: Int, rows: Int): Bitmap =
        Bitmap.createBitmap(pixels(values), cols, rows, Bitmap.Config.ARGB_8888)
}
```

- [ ] **Step 4: 통과 확인**

Run: `cd android && ./gradlew testDebugUnitTest`
Expected: BUILD SUCCESSFUL, 3 tests passed

- [ ] **Step 5: 레이아웃에 레이어 버튼, 범례, 시각 표시 추가**

`activity_main.xml`의 `status` TextView 뒤에 추가:
```xml
    <LinearLayout
        android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:layout_gravity="top|end" android:layout_marginTop="8dp" android:layout_marginEnd="8dp"
        android:orientation="vertical" android:gravity="end">
        <TextView
            android:id="@+id/timeLabel"
            android:layout_width="wrap_content" android:layout_height="wrap_content"
            android:background="#CCFFFFFF" android:padding="4dp" android:textSize="12sp" />
        <com.google.android.material.button.MaterialButton
            android:id="@+id/layerButton"
            android:layout_width="wrap_content" android:layout_height="wrap_content"
            android:text="체감온도" />
    </LinearLayout>

    <TextView
        android:id="@+id/legend"
        android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:layout_gravity="bottom|start" android:layout_marginBottom="20dp" android:layout_marginStart="4dp"
        android:background="#CCFFFFFF" android:padding="4dp" android:textSize="11sp"
        android:text="체감온도 ■&lt;26 ■26–32 ■32–38 ■38–46 ■46+" />
```

범례의 ■ 색은 Step 6의 `buildLegend()`에서 `SpannableString`으로 칠한다.

- [ ] **Step 6: MainActivity 확장**

import 추가:
```kotlin
import android.os.Handler
import android.os.Looper
import android.text.SpannableString
import android.text.Spannable
import android.text.style.ForegroundColorSpan
import org.maplibre.android.geometry.LatLngQuad
import org.maplibre.android.style.layers.RasterLayer
import org.maplibre.android.style.sources.ImageSource
import org.json.JSONArray
import java.time.OffsetDateTime
import java.time.format.DateTimeFormatter
```

필드와 함수 추가:
```kotlin
    private enum class Mode(val label: String) { HEAT("체감온도"), SHADOW("그림자"), OFF("끔") }
    private var mode = Mode.HEAT
    private var hasHeat = false
    private val ticker = Handler(Looper.getMainLooper())
    private val refresh = object : Runnable {
        override fun run() { refreshTimeLayers(); ticker.postDelayed(this, 10 * 60 * 1000L) }
    }

    private fun applyMode() {
        binding.layerButton.text = mode.label
        fun show(id: String, on: Boolean) = style.getLayer(id)?.setProperties(visibility(if (on) Property.VISIBLE else Property.NONE))
        show(HEAT_LAYER, mode == Mode.HEAT && hasHeat)
        show(SHADOW_LAYER, mode == Mode.SHADOW || (mode == Mode.HEAT && !hasHeat))
        binding.legend.visibility = if (mode == Mode.HEAT && hasHeat) View.VISIBLE else View.GONE
    }

    private fun buildLegend() {
        val text = binding.legend.text.toString()
        val span = SpannableString(text)
        var from = 0
        Heatmap.BANDS.forEach { (_, c) ->
            val i = text.indexOf('■', from)
            span.setSpan(ForegroundColorSpan(c), i, i + 1, Spannable.SPAN_EXCLUSIVE_EXCLUSIVE)
            from = i + 1
        }
        binding.legend.text = span
    }

    private fun showHeat(root: org.json.JSONObject) {
        val grid = root.getJSONObject("grid")
        val cols = grid.getInt("cols")
        val rows = grid.getInt("rows")
        val c = grid.getJSONArray("corners")
        fun ll(i: Int) = LatLng(c.getJSONArray(i).getDouble(1), c.getJSONArray(i).getDouble(0))
        val quad = LatLngQuad(ll(0), ll(1), ll(2), ll(3))
        val utci = root.optJSONArray("utci")
        hasHeat = utci != null
        if (utci != null) {
            val values = List(utci.length()) { if (utci.isNull(it)) null else utci.getDouble(it) }
            val bmp = Heatmap.bitmap(values, cols, rows)
            val src = style.getSourceAs<ImageSource>(HEAT_LAYER)
            if (src == null) {
                style.addSource(ImageSource(HEAT_LAYER, quad, bmp))
                style.addLayerBelow(RasterLayer(HEAT_LAYER, HEAT_LAYER).withProperties(
                    rasterOpacity(0.75f), rasterResampling(Property.RASTER_RESAMPLING_NEAREST)), SHADOW_LAYER)
            } else {
                src.setCoordinates(quad)
                src.setImage(bmp)
            }
        }
        val now = OffsetDateTime.now().format(DateTimeFormatter.ofPattern("HH:mm"))
        val obs = root.optJSONObject("weather")?.getString("observed_at")?.substring(11, 16)
        binding.timeLabel.text = if (obs != null) "$now 기준 · 관측 $obs" else "$now 기준"
        if (!hasHeat) showStatus("기상 관측 없음, 그늘 기준 경로") else if (binding.status.text.startsWith("기상")) hideStatus()
        applyMode()
    }
```

`refreshTimeLayers()`의 shadows 요청 아래에 climate 요청을 추가한다:
```kotlin
        api.get("api/climate", onOk = ::showHeat, onErr = ::showStatus)
```

`onStyleReady()` 끝의 `refreshTimeLayers()`를 `ticker.post(refresh)`로 바꾸고, 앞에 추가:
```kotlin
        buildLegend()
        binding.layerButton.setOnClickListener {
            mode = Mode.entries[(mode.ordinal + 1) % Mode.entries.size]
            applyMode()
        }
```

`onDestroy()`의 첫 줄에 `ticker.removeCallbacks(refresh)` 추가.

`companion object`에 `const val HEAT_LAYER = "utci"` 추가.

- [ ] **Step 7: 실기기 확인**

서버 실행 → 히트맵이 건물 외곽과 어긋나지 않게 깔린다(격자가 뒤집히면 건물 그림자 쪽이 남쪽에 나타남). 레이어 버튼으로 체감온도 → 그림자 → 끔이 바뀐다. 스냅샷 기간 밖 시각이라 기상이 없으면 배너가 뜨고 그림자가 대신 보인다.

- [ ] **Step 8: 커밋**

```bash
git add android
git commit -m "feat: show UTCI heatmap with legend and layer toggle (#18)"
```

---

### Task 13: 경로 표시와 비교 카드 (#20)

**Files:**
- Modify: `MainActivity.kt`, `activity_main.xml`

**Interfaces:**
- Consumes: `GET /api/route?from=lat,lon&to=lat,lon` (Task 8)
- Produces: `MainActivity.origin: LatLng?`(null = 현재 위치), `MainActivity.destination: LatLng?`, `fun requestRoute()`, `fun setPoint(isOrigin: Boolean, name: String, p: LatLng)`, `var currentLocation: LatLng?`

- [ ] **Step 1: 레이아웃**

`activity_main.xml`의 `mapView` 바로 뒤에 추가:
```xml
    <LinearLayout
        android:layout_width="match_parent" android:layout_height="wrap_content"
        android:layout_gravity="top" android:layout_margin="8dp" android:layout_marginEnd="120dp"
        android:background="#F2FFFFFF" android:orientation="horizontal" android:padding="4dp">
        <LinearLayout
            android:layout_width="0dp" android:layout_weight="1" android:layout_height="wrap_content"
            android:orientation="vertical">
            <TextView android:id="@+id/originField" android:layout_width="match_parent"
                android:layout_height="40dp" android:gravity="center_vertical" android:text="● 출발: 현재 위치" />
            <TextView android:id="@+id/destField" android:layout_width="match_parent"
                android:layout_height="40dp" android:gravity="center_vertical" android:text="● 도착: 선택" />
        </LinearLayout>
        <com.google.android.material.button.MaterialButton
            android:id="@+id/swapButton" style="@style/Widget.Material3.Button.TextButton"
            android:layout_width="wrap_content" android:layout_height="wrap_content" android:text="⇅" />
    </LinearLayout>

    <LinearLayout
        android:id="@+id/routeCard"
        android:layout_width="match_parent" android:layout_height="wrap_content"
        android:layout_gravity="bottom" android:layout_margin="8dp" android:layout_marginBottom="24dp"
        android:background="#F2FFFFFF" android:orientation="vertical" android:padding="8dp"
        android:visibility="gone">
        <TextView android:id="@+id/comfortRow" android:layout_width="match_parent"
            android:layout_height="40dp" android:gravity="center_vertical" />
        <TextView android:id="@+id/shortestRow" android:layout_width="match_parent"
            android:layout_height="40dp" android:gravity="center_vertical" />
        <TextView android:id="@+id/diffRow" android:layout_width="match_parent"
            android:layout_height="wrap_content" android:textSize="12sp" />
    </LinearLayout>
```

- [ ] **Step 2: 경로 레이어와 요청**

import 추가:
```kotlin
import org.maplibre.android.style.layers.LineLayer
import kotlin.math.roundToInt
```

필드와 함수:
```kotlin
    var origin: LatLng? = null
    var destination: LatLng? = null
    var currentLocation: LatLng? = null
    private var originName = "현재 위치"
    private var destName = "선택"

    fun setPoint(isOrigin: Boolean, name: String, p: LatLng) {
        if (isOrigin) { origin = p; originName = name } else { destination = p; destName = name }
        binding.originField.text = "● 출발: $originName"
        binding.destField.text = "● 도착: $destName"
        requestRoute()
    }

    private fun addRouteLayers() {
        for ((id, color, dash) in listOf(
            Triple(SHORTEST_LAYER, Color.rgb(120, 120, 120), true),
            Triple(COMFORT_LAYER, Color.rgb(22, 163, 74), false))) {
            style.addSource(GeoJsonSource(id, EMPTY_FC))
            val layer = LineLayer(id, id).withProperties(lineColor(color), lineWidth(5f))
            if (dash) layer.setProperties(lineDasharray(arrayOf(1.5f, 1.5f)))
            style.addLayer(layer)
        }
    }

    private fun lineFc(coords: JSONArray) =
        """{"type":"FeatureCollection","features":[{"type":"Feature","properties":{},"geometry":{"type":"LineString","coordinates":$coords}}]}"""

    fun requestRoute() {
        val from = origin ?: currentLocation
        val to = destination ?: return
        if (from == null) { showStatus("출발지를 선택하세요"); return }
        api.get("api/route", mapOf("from" to "${from.latitude},${from.longitude}", "to" to "${to.latitude},${to.longitude}"),
            onOk = ::showRoute, onErr = ::showStatus)
    }

    private fun summary(icon: String, name: String, r: org.json.JSONObject): String {
        val utci = if (r.isNull("mean_utci")) "" else " · 체감 ${r.getDouble("mean_utci").roundToInt()}℃"
        return "$icon $name  ${r.getDouble("distance_m").roundToInt()}m · 햇빛 ${r.getDouble("sun_ratio").roundToInt()}%$utci"
    }

    private fun showRoute(root: org.json.JSONObject) {
        val c = root.getJSONObject("comfort")
        val s = root.getJSONObject("shortest")
        style.getSourceAs<GeoJsonSource>(COMFORT_LAYER)?.setGeoJson(lineFc(c.getJSONArray("coords")))
        style.getSourceAs<GeoJsonSource>(SHORTEST_LAYER)?.setGeoJson(lineFc(s.getJSONArray("coords")))
        binding.comfortRow.text = summary("🌳", "쾌적 경로", c)
        binding.shortestRow.text = summary("☀", "최단 경로", s)
        val extra = (c.getDouble("distance_m") - s.getDouble("distance_m")).roundToInt()
        val less = (s.getDouble("sun_ratio") - c.getDouble("sun_ratio")).roundToInt()
        binding.diffRow.text = if (c.getJSONArray("coords").toString() == s.getJSONArray("coords").toString())
            "최단 경로가 이미 가장 쾌적한 길입니다" else "${extra}m 더 걷고 햇빛 ${less}%p 줄어듭니다"
        binding.routeCard.visibility = View.VISIBLE
        highlight(COMFORT_LAYER)
    }

    private fun highlight(id: String) {
        for (l in listOf(COMFORT_LAYER, SHORTEST_LAYER))
            style.getLayer(l)?.setProperties(lineWidth(if (l == id) 7f else 4f), lineOpacity(if (l == id) 1f else 0.6f))
    }
```

`onStyleReady()`에서 `ticker.post(refresh)` 앞에 추가:
```kotlin
        addRouteLayers()
        binding.comfortRow.setOnClickListener { highlight(COMFORT_LAYER) }
        binding.shortestRow.setOnClickListener { highlight(SHORTEST_LAYER) }
        // 출발이 "현재 위치"(null)면 도착으로 옮길 수 없으므로, 두 지점을 모두 직접 고른 경우에만 교체
        binding.swapButton.setOnClickListener {
            val o = origin ?: return@setOnClickListener showStatus("출발지를 직접 고른 뒤 바꿀 수 있습니다")
            val d = destination ?: return@setOnClickListener
            val on = originName
            setPoint(true, destName, d)
            setPoint(false, on, o)
        }
        // 임시 확인용: #22 전까지 지도를 길게 누르면 도착지
        map.addOnMapLongClickListener { setPoint(false, "지도 지점", it); true }
```

`companion object`에 추가:
```kotlin
        const val COMFORT_LAYER = "route-comfort"
        const val SHORTEST_LAYER = "route-shortest"
```

- [ ] **Step 3: 실기기 확인**

출발지는 아직 GPS가 없으므로 `origin`을 임시로 `LatLng(36.6268, 127.4583)`으로 두고 확인한 뒤 되돌린다. 지도를 길게 눌러 도착지 → 초록 실선과 회색 점선 경로, 카드의 거리·햇빛·체감이 보인다. 카드 행을 누르면 해당 경로가 굵어진다. 서버를 끄면 "서버에 연결할 수 없습니다". 캠퍼스에서 먼 곳을 누르면 "충북대 주변만 지원합니다".

- [ ] **Step 4: 커밋**

```bash
git add android
git commit -m "feat: show comfort and shortest routes with comparison card (#20)"
```

---

### Task 14: GPS 출발지 (#21)

**Files:**
- Modify: `MainActivity.kt`

**Interfaces:**
- Consumes: `currentLocation`, `requestRoute()`, `showStatus()` (Task 13)

- [ ] **Step 1: 위치 요청**

import 추가:
```kotlin
import android.Manifest
import android.content.pm.PackageManager
import android.location.LocationManager
import androidx.core.content.ContextCompat
import androidx.core.location.LocationManagerCompat
import androidx.activity.result.contract.ActivityResultContracts
```

필드와 함수:
```kotlin
    private val permission = registerForActivityResult(ActivityResultContracts.RequestPermission()) { ok ->
        if (ok) locate() else showStatus("위치 권한이 없어 출발지를 직접 골라야 합니다")
    }

    private fun locate() {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) != PackageManager.PERMISSION_GRANTED) {
            permission.launch(Manifest.permission.ACCESS_FINE_LOCATION)
            return
        }
        val lm = getSystemService(LOCATION_SERVICE) as LocationManager
        val provider = if (lm.isProviderEnabled(LocationManager.GPS_PROVIDER)) LocationManager.GPS_PROVIDER else LocationManager.NETWORK_PROVIDER
        LocationManagerCompat.getCurrentLocation(lm, provider, null, ContextCompat.getMainExecutor(this)) { loc ->
            if (loc == null) { showStatus("현재 위치를 찾을 수 없습니다. 출발지를 직접 고르세요"); return@getCurrentLocation }
            currentLocation = LatLng(loc.latitude, loc.longitude)
            if (origin == null) requestRoute()
        }
    }
```

`onStyleReady()`의 끝에 `locate()` 추가. 범위 밖 위치는 서버가 400을 돌려주고 `Api`가 "충북대 주변만 지원합니다"로 보여준다.

- [ ] **Step 2: 실기기 확인**

1. 권한 허용 + 캠퍼스 안 → 도착지를 누르면 현재 위치에서 경로가 나온다.
2. 앱 설정에서 권한 해제 후 재실행 → 거부하면 "위치 권한이 없어…" 문구, 앱은 계속 동작.
3. (가능하면) 캠퍼스 밖에서 → 경로 요청 시 "충북대 주변만 지원합니다".

- [ ] **Step 3: 커밋**

```bash
git add android
git commit -m "feat: use current location as default origin (#21)"
```

---

### Task 15: 장소 선택 (#22 앱)

**Files:**
- Create: `android/app/src/main/java/kr/ac/cbnu/shademap/PlacePicker.kt`
- Modify: `MainActivity.kt`

**Interfaces:**
- Consumes: `GET /api/places?q=` (Task 9), `MainActivity.setPoint(isOrigin, name, p)`, `MainActivity.api`
- Produces: `PlacePicker.show(activity: MainActivity, isOrigin: Boolean)`

- [ ] **Step 1: PlacePicker**

```kotlin
package kr.ac.cbnu.shademap

import android.os.Handler
import android.os.Looper
import android.widget.ArrayAdapter
import android.widget.EditText
import android.widget.LinearLayout
import android.widget.ListView
import android.widget.Toast
import androidx.core.widget.doAfterTextChanged
import com.google.android.material.bottomsheet.BottomSheetDialog
import com.google.android.material.button.MaterialButton
import org.maplibre.android.geometry.LatLng

object PlacePicker {
    fun show(a: MainActivity, isOrigin: Boolean) {
        val dialog = BottomSheetDialog(a)
        val search = EditText(a).apply { hint = "장소 검색" }
        val pickOnMap = MaterialButton(a).apply { text = "지도에서 선택" }
        val list = ListView(a)
        val adapter = ArrayAdapter<String>(a, android.R.layout.simple_list_item_1)
        list.adapter = adapter
        var results = listOf<Pair<String, LatLng>>()
        val handler = Handler(Looper.getMainLooper())

        fun load(q: String) = a.api.get("api/places", mapOf("q" to q), onOk = { root ->
            val arr = root.getJSONArray("places")
            results = List(arr.length()) {
                val p = arr.getJSONObject(it)
                p.getString("name") to LatLng(p.getDouble("lat"), p.getDouble("lon"))
            }
            adapter.clear()
            adapter.addAll(results.map { it.first })
            if (root.optBoolean("kakao_error")) Toast.makeText(a, "검색 서버 오류, 캠퍼스 장소만 표시", Toast.LENGTH_SHORT).show()
        }, onErr = { Toast.makeText(a, it, Toast.LENGTH_SHORT).show() })

        search.doAfterTextChanged { text ->
            handler.removeCallbacksAndMessages(null)
            handler.postDelayed({ load(text.toString()) }, 300)
        }
        list.setOnItemClickListener { _, _, i, _ ->
            val (name, p) = results[i]
            a.setPoint(isOrigin, name, p)
            dialog.dismiss()
        }
        pickOnMap.setOnClickListener {
            dialog.dismiss()
            Toast.makeText(a, "지도를 눌러 지점을 고르세요", Toast.LENGTH_SHORT).show()
            a.map.addOnMapClickListener(object : org.maplibre.android.maps.MapLibreMap.OnMapClickListener {
                override fun onMapClick(point: LatLng): Boolean {
                    a.map.removeOnMapClickListener(this)
                    a.setPoint(isOrigin, "지도 지점", point)
                    return true
                }
            })
        }
        dialog.setContentView(LinearLayout(a).apply {
            orientation = LinearLayout.VERTICAL
            setPadding(32, 32, 32, 32)
            addView(search)
            addView(pickOnMap)
            addView(list)
        })
        dialog.show()
        load("")
    }
}
```

- [ ] **Step 2: MainActivity 연결**

`onStyleReady()`에 추가:
```kotlin
        binding.originField.setOnClickListener { PlacePicker.show(this, true) }
        binding.destField.setOnClickListener { PlacePicker.show(this, false) }
```
Task 13의 임시 `addOnMapLongClickListener` 줄은 삭제한다.

범위 밖 지점을 고르면 서버 400 → "충북대 주변만 지원합니다" 문구가 뜬다.

- [ ] **Step 3: 실기기 확인**

1. 도착 누르기 → 캠퍼스 장소 목록이 보이고, 고르면 경로가 나온다.
2. "중앙도서관" 입력 → 결과가 보인다(서버 `.env`에 `KAKAO_REST_KEY`가 있으면 카카오 결과도 섞임).
3. "지도에서 선택" → 지도를 누르면 그 지점으로 경로.
4. 비행기 모드 → 오류 문구만 뜨고 앱이 멈추지 않는다.
5. `grep -r "KakaoAK\|KAKAO" android/` 결과가 없다(키가 앱에 없음).

- [ ] **Step 4: 커밋**

```bash
git add android
git commit -m "feat: add place picker with campus list, map pick and search (#22)"
```

---

### Task 16: 1단계 실기기 검증 (#23)

**Files:**
- Modify: `README.md`

- [ ] **Step 1: 전체 테스트**

```bash
cd backend && pytest -q
cd ../android && ./gradlew testDebugUnitTest
```
Expected: 둘 다 모두 PASS

- [ ] **Step 2: 성능 측정**

Run: `PYTHONPATH=backend python scripts/bench.py`
Expected: climate, route 각각 1초 이내. 결과를 #23에 댓글로 남긴다.

- [ ] **Step 3: 이슈 #23 체크리스트를 실기기에서 하나씩 확인**

- 3D 건물, 히트맵, 그림자(레이어 전환)
- 두 경로 + 비교 카드
- 위치 권한 거부 / 범위 밖 / 해 진 뒤 / 기상 없음 / 서버 꺼짐
- K 값: 캠퍼스 경로 3개 이상으로 "쾌적 경로가 너무 멀리 돌거나(K 과대) 최단과 늘 같으면(K 과소)" `config.K_COMFORT` 조정

- [ ] **Step 4: README에 한계 기록**

`README.md`에 추가:
```markdown
## 1단계 한계

- 나무 그늘, 지형, 지면 종류는 반영하지 않는다(2, 3단계).
- 기온, 습도, 바람은 캠퍼스 전체에 기상청 청주 관측값 하나를 쓴다.
- 높이 정보가 없는 건물은 그림자를 만들지 않는다.
- 평균복사온도는 주변 장파 복사를 기온과 같다고 가정한 단순식이다.
```

- [ ] **Step 5: 커밋**

```bash
git add README.md
git commit -m "docs: record stage-1 verification and limits (#23)"
```

---

## 이슈 ↔ 태스크

| 이슈 | 태스크 |
|---|---|
| #11 | 1 |
| #12 | 10 |
| #13 | 2, 11 |
| #14 | 3 |
| #15 | 5, 7 |
| #16 | 4, 11 |
| #17 | 6 |
| #18 | 12 |
| #19 | 8 |
| #20 | 13 |
| #21 | 14 |
| #22 | 9, 15 |
| #23 | 16 |

병렬 가능: 서버(1→2→4→5→6→8→9, 7은 5 뒤 언제든)와 앱(10은 1 뒤, 11은 2·4 뒤)은 팀원이 나눠 동시에 진행할 수 있다. 3은 2 뒤 독립.
