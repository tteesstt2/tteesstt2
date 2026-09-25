# 검색과 화면에서 쓰는 데이터

데이터 예시는 [`data-model-example.json`](./data-model-example.json) 참고.

## 1. 검색

### 1.1 검색 단위
**장면(Scene) 1개 = 검색 문서 1개.** DB의 Video·Scene·Place·PlaceName을 합쳐 평탄화한 문서를 검색엔진에 넣는다 (`search_document_example`).

### 1.2 검색 대상 필드와 가중치

| 순위 | 필드 | 이유 | 예: "아사쿠사" 검색 |
|---|---|---|---|
| 1 | `place_names` (다국어·읽기·별칭) | 장소명 검색이 가장 많음 | 浅草 / Asakusa 모두 매칭 |
| 2 | `region`, `nearest_station` | "시부야 라멘"처럼 지역+무엇 | 浅草 지역 장면 전부 |
| 3 | `on_screen_text`, `speech_text` | 장소 확정 전에도 검색되게 | 간판 글자 매칭 |
| 4 | `summary` (ko/ja/en) | 장면 내용 검색 | |
| 5 | `tags` | "야경", "비", "먹방" 같은 분위기 | |
| 6 | `video_title`, `channel_name` | 특정 유튜버 찾기 | |

### 1.3 검색 방식 (두 가지를 섞음)

| 방식 | 쓰는 데이터 | 잘하는 것 |
|---|---|---|
| 키워드 검색 | 위 텍스트 필드 | 정확한 이름 (雷門, 이치란), 오타 허용 |
| 의미 검색 | `embedding` | "비 오는 밤 골목", "조용한 신사" 같은 문장 |

두 점수를 합친 뒤 아래 값으로 순위를 보정한다.
- `verified = true` → 가산점 (검증된 장소 우선)
- `confidence` 높을수록 가산점
- `published_at` 최신일수록 약간 가산점

### 1.4 필터 (검색 결과 옆 선택 항목)

| 필터 | 필드 |
|---|---|
| 국가 / 지역 | `region` (JP → 東京都 → 台東区) |
| 장소 유형 | `place_type` (음식점, 신사, 역, 전망대…) |
| 활동 | `tags` 중 activity (먹기, 걷기, 쇼핑…) |
| 시간대 / 날씨 | `tags` 중 day/night, rain/snow |
| 지도 범위 | `location` (지도에서 보이는 영역 안) |
| 검증된 것만 | `verified` |

### 1.5 검색어 처리 예시

| 입력 | 처리 |
|---|---|
| `센소지` | PlaceName 별칭 매칭 → 浅草寺 및 하위 장소(雷門 등, `parent_place_id`) 장면 |
| `시부야 라멘` | 지역(渋谷) + 장소유형/품목(ramen) |
| `비 오는 밤 골목` | 의미 검색 + 태그 rain·night |
| `이치란` (장소 미검증 영상) | `speech_text`·`on_screen_text`로 매칭 |

## 2. 화면별 사용 데이터

### 2.1 검색 결과 목록 (장면 카드)

```
┌───────────────────────────────────────┐
│ [영상 썸네일]  ▶ 10:10                 │
│ 雷門 · 카미나리몬  ✔검증                │
│ 도쿄 台東区 浅草 · 浅草駅               │
│ 큰 붉은 등이 걸린 문을 지나 상점가...    │
│ #걷기 #낮 · 여행하는 민지 · 2026.08     │
└───────────────────────────────────────┘
```

| 표시 | 데이터 |
|---|---|
| 썸네일 | `video.thumbnail_url` |
| 재생 시점 | `scene.start_sec` → `mm:ss` |
| 장소명 | `place.canonical_name` + 사용자 언어의 `names` |
| 검증 배지 | `place.verification_status` |
| 지역·역 | `admin_area_1/2`, `locality`, `nearest_station` |
| 설명 | `scene.summary` (사용자 언어) |
| 태그 | `activity_tags`, `time_of_day`, `weather` |
| 영상 정보 | `channel.name`, `video.published_at` |

클릭 → `youtube.com/watch?v={youtube_video_id}&t={start_sec}s` 또는 장면 상세.

### 2.2 장면 상세

| 영역 | 데이터 |
|---|---|
| 플레이어 | YouTube 임베드, `start=start_sec`, `end=end_sec` |
| 장면 설명 | `summary`, `visible_items` |
| 장소 정보 | Place 전체 + 작은 지도(`location`) |
| **근거 표시** | `evidence_detail`, `evidence_at_sec` (예: "10:15 간판 '雷門'") — 사실 기반임을 보여줌 |
| 같은 영상의 다른 장면 | 같은 `video_id`의 Scene 목록 (타임라인) |

### 2.3 장소 상세

| 영역 | 데이터 |
|---|---|
| 이름 | `canonical_name`, `names` |
| 주소·역·지도 | `address`, `nearest_station`, `location`, `coord_precision` |
| 이 장소가 나온 장면들 | ScenePlace로 연결된 Scene (최신순/인기순) |
| 통계 | `scene_count`, 등장 영상 수 |
| 주변 장소 | `location` 반경 검색 |

### 2.4 지도 보기

| 표시 | 데이터 |
|---|---|
| 핀 | `verified = true`인 Place의 `location` |
| 핀 크기/색 | `scene_count`, `place_type` |
| 핀 클릭 | 장소 요약 + 대표 장면 카드 |

`coord_precision = area/city`(골목·동네 단위)는 핀 대신 영역 원으로 표시.

### 2.5 관리자 검증 화면

| 영역 | 데이터 |
|---|---|
| 대기 목록 | `review_status = pending`인 ScenePlace (낮은 `confidence` 순) |
| 근거 확인 | 플레이어(`evidence_at_sec`부터) + `raw_name`, `evidence_detail`, `on_screen_text`, `speech_mentions` |
| 장소 후보 | 지도 API 검색 결과 (이름, 주소, 좌표) |
| 편집 | 이름·좌표(지도에서 핀 이동)·`coord_precision`·`place_type` |
| 동작 | 승인 / 수정 / 반려 / 기존 장소와 병합 → Review 기록 |
| 이력 | 해당 대상의 Review 목록 |

## 3. 사용자에게 노출하지 않는 데이터

| 데이터 | 용도 |
|---|---|
| `analysis_runs` (모델, 프롬프트, 원본 응답, 비용) | 재분석, 품질 비교, 비용 관리 |
| `confidence` 수치 | 순위 계산, 관리자 우선순위 |
| `reviews` | 감사 기록 |
| `embedding` | 의미 검색 |
