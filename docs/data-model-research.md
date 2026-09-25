# 데이터 항목 조사: 영상 속 장소·장면 검색 서비스

> 상태: 초안 (PoC 전). 실제 Gemini 분석 결과를 보고 항목을 확정·조정한다.

## 1. 설계 원칙

1. **사실 기반**: 모든 추출값에는 **근거(evidence)** 를 붙인다. 근거가 없는 장소명은 저장하지 않고 "미식별 장소"로 남긴다.
2. **관찰과 추론 분리**: 영상에서 직접 보이거나 들린 것(`observed`)과 AI가 추정한 것(`inferred`)을 구분한다.
3. **사람 검증이 최종값**: AI 값과 관리자 검증값을 별도로 보관하고, 사용자에게는 검증된 값을 우선 노출한다.
4. **확장성**: 국가·언어·카테고리를 하드코딩하지 않는다. 일본은 `country_code = JP`인 데이터일 뿐이다.
5. **재분석 가능**: 어떤 모델·프롬프트 버전으로 뽑았는지 기록해, 모델 교체 시 비교·재처리할 수 있게 한다.

## 2. 데이터 출처

| 출처 | 얻는 것 | 비고 |
|---|---|---|
| YouTube Data API v3 | 제목, 설명, 채널, 게시일, 길이, 썸네일, 태그, 설명란 챕터 | 타인 영상의 자막 다운로드는 공식 API로 불가 |
| Gemini API | 장면 구간, 장면 설명, 화면 속 글자(간판 등), 음성 언급, 장소 후보 | 공개 YouTube URL을 직접 입력 가능 여부·길이 제한은 PoC에서 확인 |
| 지도/장소 API | 장소 ID, 정식 명칭(다국어), 주소, 좌표, 장소 유형 | Google Places, OpenStreetMap(Nominatim), 일본 국토지리원 등 후보 |
| 관리자 | 최종 장소명, 좌표, 병합/분리, 반려 사유 | 검증 이력 보관 |

## 3. 엔티티 구조

```
Channel 1─N Video 1─N Scene N─M Place
                     │        (ScenePlace: 연결 + 근거)
                     └─ AnalysisRun (분석 실행 기록)
Place 1─N PlaceName (다국어/별칭)
Place 1─N Review (검증 이력)
```

### 3.1 Video (영상)

| 필드 | 타입 | 출처 | 용도 |
|---|---|---|---|
| id | uuid | 내부 | |
| youtube_video_id | string, unique | URL 파싱 | 링크 생성 |
| title / description | text | YouTube API | 검색, 장소 힌트 |
| channel_id | fk | YouTube API | 채널 필터 |
| published_at | datetime | YouTube API | 최신순 정렬 |
| duration_sec | int | YouTube API | |
| thumbnail_url | string | YouTube API | 결과 카드 |
| language | string | YouTube API / Gemini | 다국어 처리 |
| chapters | json | 설명란 파싱 | 장면 분할 힌트 |
| category | string | 내부 (예: `travel_vlog`) | **확장용** |
| countries | string[] | 장면에서 집계 | 국가 필터 |
| status | enum | 내부 | `pending / analyzing / analyzed / failed / reviewed` |

### 3.2 Scene (장면: 영상의 시간 구간)

| 필드 | 타입 | 출처 | 용도 |
|---|---|---|---|
| id | uuid | 내부 | |
| video_id | fk | | |
| start_sec / end_sec | int | Gemini | `?t=` 딥링크, 구간 재생 |
| summary | text | Gemini | 검색·표시. **보이는 사실만** 서술 |
| summary_i18n | json | 번역 | 다국어 검색 (ko/ja/en) |
| activity_tags | string[] | Gemini | 예: `eating`, `walking`, `shopping`, `train_ride`, `hotel_room` |
| setting | enum | Gemini | `indoor / outdoor / vehicle / unknown` |
| time_of_day | enum | Gemini | `day / night / dusk / unknown` (화면 기준) |
| weather | enum | Gemini | `clear / rain / snow / cloudy / unknown` |
| visible_items | string[] | Gemini | 예: 라멘, 도리이, 벚꽃, 자판기 |
| on_screen_text | json[] | Gemini | 간판·메뉴·표지판 글자 원문 + 등장 초 |
| speech_mentions | json[] | Gemini | 음성·자막에서 언급된 고유명사 + 등장 초 |
| embedding | vector | 임베딩 모델 | 의미 기반 검색 |
| analysis_run_id | fk | 내부 | 어떤 분석에서 나왔는지 |

### 3.3 Place (장소: 실제 세계의 한 지점)

| 필드 | 타입 | 출처 | 용도 |
|---|---|---|---|
| id | uuid | 내부 | |
| canonical_name | string | 관리자 확정 | 대표 표시명 |
| place_type | string | 지도 API / Gemini | 예: `restaurant`, `shrine`, `station`, `street`, `viewpoint` |
| parent_place_id | fk, nullable | 관리자 | 상위 장소 (雷門 → 浅草寺). 상위 장소로 검색해도 하위 장면이 나오게 |
| country_code | ISO 3166-1 | 지도 API | **확장 대비 필수** |
| admin_area_1 | string | 지도 API | 일본: 도도부현 (東京都) |
| admin_area_2 | string | 지도 API | 일본: 시구정촌 (渋谷区) |
| locality | string | 지도 API | 동네 (예: 道玄坂) |
| address | string | 지도 API | |
| lat / lng | decimal | 지도 API → **관리자 검증** | 지도 표시 |
| coord_precision | enum | 관리자 | `exact / building / area / city` (거리·동네 단위 장소 대응) |
| nearest_station | string | 지도 API / 관리자 | 일본 여행 검색에 유용 |
| external_ids | json | 지도 API | `{google_place_id, osm_id}` |
| verification_status | enum | 관리자 | `unverified / verified / rejected / merged` |

### 3.4 PlaceName (다국어 이름·별칭)

한 장소를 여러 이름으로 검색할 수 있게 한다.

| 필드 | 예시 |
|---|---|
| place_id | |
| name | 浅草寺 / 센소지 / Senso-ji / 아사쿠사 절 |
| language | ja / ko / en |
| kind | `official / reading(가나·로마자) / alias / typo` |

### 3.5 ScenePlace (장면 ↔ 장소 연결 + 근거) — **사실 판단의 핵심**

| 필드 | 타입 | 설명 |
|---|---|---|
| scene_id / place_id | fk | place_id는 미식별이면 null |
| raw_name | string | Gemini가 뽑은 이름 원문 |
| evidence_type | enum[] | `on_screen_text / speech / caption / landmark_visual / video_metadata` |
| evidence_detail | text | 근거 원문 (예: 간판 "一蘭 渋谷店", 음성 "여기가 도톤보리") |
| evidence_at_sec | int | 근거가 나오는 초 |
| certainty | enum | `observed`(직접 확인) / `inferred`(추정) |
| confidence | float 0–1 | 모델 자체 평가 (참고용) |
| review_status | enum | `pending / approved / corrected / rejected` |

### 3.6 AnalysisRun (분석 실행 기록)

| 필드 | 설명 |
|---|---|
| video_id | |
| model | 예: gemini 모델명 |
| prompt_version | 프롬프트 버전 |
| raw_response | 원본 JSON 보관 (재처리·디버깅용) |
| token_usage / cost | 비용 추적 |
| started_at / finished_at / error | |

### 3.7 Review (관리자 검증 이력)

| 필드 | 설명 |
|---|---|
| target_type / target_id | place 또는 scene_place |
| reviewer | 관리자 |
| action | `approve / edit / reject / merge` |
| before / after | 변경 전후 값 (json) |
| note | 사유 |

## 4. 사용자에게 보여줄 데이터 (검색 결과 카드)

- 장면 썸네일(영상 썸네일) + **해당 초로 이동하는 링크** (`youtube.com/watch?v=ID&t=123s`) 또는 임베드 플레이어의 `start` 파라미터
- 장면 설명 (사실 요약)
- 장소명 (검증됨 표시), 지역(도도부현·시), 가까운 역
- 지도 핀 (검증된 좌표만)
- 같은 장소가 나온 다른 영상/장면 목록

> 주의: 장면 순간의 프레임 이미지를 직접 추출하려면 영상 다운로드가 필요해 YouTube 약관에 저촉될 수 있다. 기본은 **영상 썸네일 + 타임스탬프 링크**로 간다.

## 5. "구글처럼" 검색을 위한 요구사항

| 요구 | 구현 방향 | 필요한 데이터 |
|---|---|---|
| 다국어 (한/일/영) | 모든 이름·설명을 다국어로 색인 | PlaceName, summary_i18n |
| 읽기·표기 차이 (浅草 / あさくさ / Asakusa / 아사쿠사) | 가나·로마자 읽기 별칭 저장 | PlaceName.kind = reading |
| 오타 허용 | 퍼지 매칭 | 검색엔진 기능 |
| 뜻으로 검색 ("비 오는 밤 골목") | 의미 기반(벡터) 검색 | Scene.embedding, 태그 |
| 필터 | 패싯 | country, admin_area, place_type, activity_tags, time_of_day, weather |
| 지도 범위 검색 | 좌표 기반 쿼리 | lat/lng |
| 랭킹 | 키워드 점수 + 의미 점수 + 검증 여부 가산점 | verification_status, confidence |

검색엔진 후보: PostgreSQL(전문 검색 + pgvector), Meilisearch, Elasticsearch/OpenSearch. 초기에는 **PostgreSQL + pgvector** 하나로 시작하고, 규모가 커지면 분리하는 것을 추천한다.

## 6. Gemini 추출 가능성 예상 (PoC에서 검증할 항목)

| 항목 | 예상 | 검증 포인트 |
|---|---|---|
| 장면 구간 분할 + 타임스탬프 | 높음 | 초 단위 오차 범위 |
| 장면 사실 요약 | 높음 | 추측성 표현 섞이는지 |
| 화면 속 일본어 간판 읽기 | 중~높음 | 작은 글자·흐린 화면 정확도 |
| 음성 언급 고유명사 | 중 | 한국어/일본어 음성, BGM 섞인 경우 |
| 유명 랜드마크 인식 | 중~높음 | 근거 없이 단정하는지 |
| 일반 가게/골목 특정 | 낮음 | → 관리자 검증 필수 |
| 좌표 | 낮음 (AI가 직접 뽑지 않음) | 지도 API로 후보를 찾고 관리자가 확정 |

## 7. Gemini 출력 형식 초안 (PoC용)

```json
{
  "scenes": [
    {
      "start_sec": 125,
      "end_sec": 190,
      "summary": "좁은 골목의 라멘 가게 카운터석에서 라멘을 먹는다.",
      "activity_tags": ["eating"],
      "setting": "indoor",
      "time_of_day": "night",
      "weather": "unknown",
      "visible_items": ["ramen", "counter_seat", "ticket_machine"],
      "on_screen_text": [{ "text": "一蘭 渋谷店", "at_sec": 128 }],
      "speech_mentions": [{ "text": "이치란", "at_sec": 130 }],
      "places": [
        {
          "raw_name": "一蘭 渋谷店",
          "place_type": "restaurant",
          "evidence_type": ["on_screen_text", "speech"],
          "evidence_detail": "간판에 '一蘭 渋谷店' 표기, 화자가 '이치란'이라고 언급",
          "evidence_at_sec": 128,
          "certainty": "observed",
          "confidence": 0.9
        }
      ]
    }
  ]
}
```

프롬프트 규칙 초안:
- 화면이나 음성으로 확인되지 않은 장소명은 `places`에 넣지 말 것
- 추정인 경우 `certainty: "inferred"`로 표시하고 근거를 적을 것
- 등장 인물의 신원(일반인 얼굴·이름)은 추출하지 말 것

## 8. 확인이 필요한 사항 (정책·약관)

- **YouTube API 데이터 보관 정책**: API로 받은 메타데이터의 저장 기간·갱신 의무 확인 필요
- **Google Places 데이터 저장 제한**: place_id 외 좌표 등은 캐시 기간 제한이 있을 수 있음. 관리자가 검증한 자체 좌표를 쓸지, OSM 등 개방 데이터를 쓸지 결정 필요
- **Gemini의 YouTube URL 입력**: 지원 범위(공개 영상 여부, 길이, 일일 한도)와 비용
- **개인정보**: 브이로거 외 일반인 정보는 저장하지 않음

## 9. 다음 단계

1. 일본 여행 브이로그 3~5개 선정 (도심 / 음식 / 자연 / 야간 등 다양하게)
2. 7번 형식으로 Gemini 분석 스크립트 작성·실행
3. 결과를 사람이 채점: 타임스탬프 정확도, 장소명 정확도, 근거 없는 단정 비율
4. 결과에 따라 필드 추가/삭제 후 DB 스키마 확정
