 **목적**: 하이브리드 검색 시 매칭된 중첩 객체나 자식 문서를 검색하여 문서의 특정 부분이 쿼리와 어떻게 매칭되었는지 탐색

### 핵심 개념

#### Inner Hits 실행 과정

1. **서브쿼리 실행**: 각 서브쿼리가 inner hits의 관련성에 기반하여 부모 문서 선택
2. **문서 결합**: 모든 서브쿼리의 선택된 부모 문서들을 결합하고 점수를 정규화하여 하이브리드 점수 생성
3. **Inner Hits 검색**: 각 부모 문서에 대해 관련 inner_hits를 샤드에서 검색하여 최종 응답에 포함

#### 전통적 쿼리 vs 하이브리드 쿼리

- **전통적 쿼리**: 부모 문서의 최종 순위가 inner_hits 점수에 의해 직접 결정
- **하이브리드 쿼리**: 최종 순위는 하이브리드 점수로 결정되지만, 부모 문서는 여전히 inner_hits의 관련성에 기반하여 샤드에서 가져옴

#### 점수 표시

- **Inner_hits 섹션**: 정규화 이전의 원본(raw) 점수 표시
- **부모 문서**: 최종 하이브리드 점수 표시

### 구현 예제

#### 1단계: 인덱스 생성

중첩된 필드(`user`, `location`)를 가진 인덱스 생성:

```json
PUT /my-nlp-index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "user": {
        "type": "nested",
        "properties": {
          "name": { "type": "text" },
          "age": { "type": "integer" }
        }
      },
      "location": {
        "type": "nested",
        "properties": {
          "city": { "type": "text" },
          "state": { "type": "text" }
        }
      }
    }
  }
}
```

#### 2단계: 검색 파이프라인 생성

정규화와 결합 기법이 포함된 검색 파이프라인 설정:

```json
PUT /_search/pipeline/nlp-search-pipeline
{
  "description": "Post processor for hybrid search",
  "phase_results_processors": [
    {
      "normalization-processor": {
        "normalization": { "technique": "min_max" },
        "combination": { 
          "technique": "arithmetic_mean",
          "parameters": {}
        }
      }
    }
  ]
}
```

#### 3단계: 문서 삽입

```json
POST /my-nlp-index/_bulk
{"index": {"_index": "my-nlp-index"}}
{"user":[{"name":"John Alder","age":35},{"name":"Sammy","age":34}],"location":[{"city":"Amsterdam","state":"Netherlands"},{"city":"Udaipur","state":"Rajasthan"}]}
{"index": {"_index": "my-nlp-index"}}
{"user":[{"name":"John Wick","age":46},{"name":"John Snow","age":40}],"location":[{"city":"Tromso","state":"Norway"},{"city":"Los Angeles","state":"California"}]}
```

#### 4단계: Inner Hits를 포함한 하이브리드 검색

```json
GET /my-nlp-index/_search?search_pipeline=nlp-search-pipeline
{
  "query": {
    "hybrid": {
      "queries": [
        {
          "nested": {
            "path": "user",
            "query": { "match": { "user.name": "John" } },
            "score_mode": "sum",
            "inner_hits": {}
          }
        },
        {
          "nested": {
            "path": "location",
            "query": { "match": { "location.city": "Udaipur" } },
            "inner_hits": {}
          }
        }
      ]
    }
  }
}
```

### 고급 기능

#### Explain 파라미터 사용

점수 계산 과정을 자세히 분석하기 위해 `explain=true` 추가:

```json
GET /my-nlp-index/_search?search_pipeline=nlp-search-pipeline&explain=true
```

**주의사항**: explain은 리소스와 시간 면에서 비용이 많이 드는 작업이므로 프로덕션에서는 트러블슈팅 목적으로만 사용 권장

#### Inner Hits 정렬

나이별 내림차순 정렬 예제

```json
"inner_hits": {
  "sort": [
    {
      "user.age": { "order": "desc" }
    }
  ]
}
```

**결과**: 커스텀 정렬 적용 시 `_score` 필드는 `null`이 됨 (점수 계산하지 않음)

#### Inner Hits 페이지네이션

세 번째와 네 번째 중첩 객체만 가져오기

```json
"inner_hits": {
  "from": 2,
  "size": 2
}
```

#### 커스텀 이름 정의

Inner hits 필드에 커스텀 이름 지정

```json
"inner_hits": {
  "name": "coordinates"
}
```

### 응답 구조 분석

#### 기본 응답 형태

```json
{
  "hits": [
    {
      "_index": "my-nlp-index",
      "_id": "1",
      "_score": 1.0,
      "inner_hits": {
        "location": {
          "hits": {
            "max_score": 0.44583148,
            "hits": [
              {
                "_nested": { "field": "location", "offset": 1 },
                "_score": 0.44583148,
                "_source": { "city": "Udaipur", "state": "Rajasthan" }
              }
            ]
          }
        },
        "user": {
          "hits": {
            "max_score": 0.4394061,
            "hits": [
              {
                "_nested": { "field": "user", "offset": 0 },
                "_score": 0.4394061,
                "_source": { "name": "John Alder", "age": 35 }
              }
            ]
          }
        }
      }
    }
  ]
}
```

### 활용 사례

#### 장점

- 복합 검색에서 어떤 중첩 객체가 매칭되었는지 정확히 파악
- 부모-자식 관계에서 세밀한 검색 결과 분석
- 다중 필드 검색에서 각 필드별 기여도 확인

#### 실무 적용

- 사용자 프로필과 위치 정보를 함께 검색하는 시스템
- 제품 카탈로그에서 속성별 매칭 결과 분석
- 문서 내 특정 섹션과 메타데이터 동시 검색

### 성능 고려사항

- Inner hits는 추가적인 계산 비용 발생
- 대용량 데이터에서는 페이지네이션 활용 권장
- Explain 기능은 디버깅 목적으로만 제한적 사용