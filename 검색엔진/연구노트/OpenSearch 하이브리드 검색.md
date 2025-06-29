### 개요

하이브리드 검색(Hybrid Search)은 키워드 매칭(keyword search)과 의미 기반(semantic) 검색을 결합하여 검색 정확도를 높이는 기능입니다.

OpenSearch에서 하이브리드 검색을 사용하려면:

1. **검색 파이프라인(search pipeline)** 구성
2. **검색 시점**에 파이프라인을 통해 얻은 결과 점수를 정규화 병합
3. **최종 순위** 매기기

### 사전 준비사항

#### 필수 구성 요소

1. **텍스트 임베딩 모델** 준비
2. **플러그인 활성화**
    - `knn_vector` 플러그인
    - `text_embedding` 플러그인
    - `flow_framework` 플러그인

#### 기본 하이브리드 검색 설정

```json
GET /my-nlp-index/_search?search_pipeline=nlp-search-pipeline
{
  "query": {
    "hybrid": {
      "queries": [
        {
          "match": {
            "text": "검색어"
          }
        },
        {
          "neural": {
            "passage_embedding": {
              "query_text": "검색어",
              "model_id": "model_id",
              "k": 5
            }
          }
        }
      ]
    }
  }
}
```

### 페이지네이션 (Pagination)

#### 1. 기본 페이지네이션 (`from`/`size` 방식)

#### `pagination_depth`의 역할

- **각 샤드별로** 하이브리드 쿼리의 서브쿼리(벡터/불린 등)에서 **최대 몇 개의 문서**를 메모리에 유지할지 지정
- 예: `pagination_depth: 50` 설정 시, 각 샤드에서 최대 50개의 후보 결과를 모아서 랭킹 후 병합

```json
{
  "from": 0,
  "size": 5,
  "query": {
    "hybrid": {
      "queries": [
        { "match": { "text": "검색어" } },
        { "neural": { "passage_embedding": { "query_text": "검색어", "model_id": "model_id", "k": 5 } } }
      ],
      "pagination_depth": 10
    }
  }
}
```

#### 주요 파라미터

- **`from`**: 시작 위치 (0부터 시작)
- **`size`**: 한 페이지당 결과 수
- **`pagination_depth`**: 각 샤드별 후보 문서 수

#### 페이지 간 일관성 유지

- `pagination_depth`를 변경하면 실제 후보(pool)의 범위가 달라져 **최종 결과 순서가 바뀔 수 있음**
- **처음부터 끝까지 동일한 `pagination_depth` 값을 사용**해야 일관된 페이징 보장

#### 깊은 페이지네이션 (Deep Pagination) 고려사항

- **더 뒤쪽 페이지**를 보려면 `pagination_depth`를 충분히 크게 설정
- **단점**: `pagination_depth`가 높을수록 메모리/CPU 사용량 증가로 성능 저하 발생

### 2. `search_after` 방식 (권장)

#### 개념

- **라이브 커서(cursor)** 역할
- 이전 페이지 마지막 문서의 정렬 키 값을 다음 요청에 전달
- 큰 `from` 오프셋 없이도 효율적인 페이징 가능

#### 첫 번째 페이지 요청

```json
GET /my-nlp-index/_search?search_pipeline=nlp-search-pipeline
{
  "query": {
    "hybrid": {
      "queries": [
        { "match": { "text": "검색어" } },
        { "neural": { "passage_embedding": { "query_text": "검색어", "model_id": "model_id", "k": 5 } } }
      ]
    }
  },
  "sort": [
    { "doc_price": { "order": "desc" } }
  ],
  "size": 2
}
```

**예시 결과:**

```json
{
  "hits": {
    "hits": [
      {
        "_id": "1",
        "_score": 0.8,
        "_source": { "doc_price": 350 },
        "sort": [350]
      },
      {
        "_id": "2", 
        "_score": 0.7,
        "_source": { "doc_price": 200 },
        "sort": [200]
      }
    ]
  }
}
```

#### 두 번째 페이지 요청

```json
GET /my-nlp-index/_search?search_pipeline=nlp-search-pipeline
{
  "query": {
    "hybrid": {
      "queries": [
        { "match": { "text": "검색어" } },
        { "neural": { "passage_embedding": { "query_text": "검색어", "model_id": "model_id", "k": 5 } } }
      ]
    }
  },
  "sort": [
    { "doc_price": { "order": "desc" } }
  ],
  "search_after": [200],
  "size": 2
}
```

- `doc_price`가 200보다 작은 문서부터 다음 2건을 반환

### 페이지네이션 방식별 비교

|방식|장점|단점|권장 사용|
|---|---|---|---|
|`from`/`size`|구현 단순, 임의 페이지 접근 가능|깊은 페이지에서 성능 저하|얕은 페이지네이션|
|`search_after`|성능 우수, 대용량 데이터 처리|임의 페이지 접근 불가|깊은 페이지네이션, 실시간 스크롤|

## 정렬 기준별 `search_after` 예시

### 1. 가격 기준 페이지네이션

```json
{
  "sort": [
    { "doc_price": { "order": "desc" } }
  ],
  "search_after": [200]
}
```

- `doc_price`가 200보다 작은 순서로 결과 반환

### 2. 문서 ID 기준 페이지네이션

```json
{
  "sort": [
    { "_id": { "order": "desc" } }
  ],
  "search_after": ["7yaM4JABZkI1FQv8AwoN"]
}
```

- `_id`가 `"7yaM4JABZkI1FQv8AwoN"`보다 뒤에 오는 문서 반환

### 3. 복합 정렬 기준

```json
{
  "sort": [
    { "_score": { "order": "desc" } },
    { "timestamp": { "order": "desc" } },
    { "_id": { "order": "asc" } }
  ],
  "search_after": [0.85, "2024-06-24T10:00:00Z", "doc_123"]
}
```

- 관련도 점수 → 시간 → ID 순으로 정렬
- `search_after` 배열도 정렬 순서와 동일하게 구성

## 성능 최적화 팁

#### 1. `pagination_depth` 최적화

```json
{
  "pagination_depth": 100  // 필요한 만큼만 설정
}
```

#### 2. `search_after` 사용 권장

- 10페이지 이상의 깊은 페이지네이션에서는 `search_after` 사용

#### 3. 적절한 `k` 값 설정

```json
{
  "neural": {
    "passage_embedding": {
      "k": 50  // 너무 크지 않게 설정
    }
  }
}
```

#### 4. 필요한 필드만 반환

```json
{
  "_source": {
    "includes": ["title", "content"],
    "excludes": ["passage_embedding"]
  }
}
```

### 실제 사용 예시

#### 전자상거래 상품 검색

```json
GET /products/_search?search_pipeline=product-search-pipeline
{
  "query": {
    "hybrid": {
      "queries": [
        {
          "bool": {
            "must": [
              { "match": { "title": "무선 이어폰" } },
              { "range": { "price": { "lte": 100000 } } }
            ]
          }
        },
        {
          "neural": {
            "description_embedding": {
              "query_text": "블루투스 무선 이어폰 고음질",
              "model_id": "product_embedding_model",
              "k": 20
            }
          }
        }
      ],
      "pagination_depth": 50
    }
  },
  "sort": [
    { "_score": { "order": "desc" } },
    { "sales_count": { "order": "desc" } },
    { "price": { "order": "asc" } }
  ],
  "size": 20
}
```

#### 뉴스/문서 검색

```json
GET /articles/_search?search_pipeline=news-search-pipeline
{
  "query": {
    "hybrid": {
      "queries": [
        {
          "bool": {
            "must": [
              { "match": { "title": "인공지능" } },
              { "match": { "content": "기술 발전" } }
            ],
            "filter": [
              { "range": { "published_date": { "gte": "2024-01-01" } } }
            ]
          }
        },
        {
          "neural": {
            "content_embedding": {
              "query_text": "AI 기술의 최신 동향과 발전 방향",
              "model_id": "news_embedding_model", 
              "k": 30
            }
          }
        }
      ]
    }
  },
  "sort": [
    { "_score": { "order": "desc" } },
    { "published_date": { "order": "desc" } }
  ],
  "search_after": [0.75, "2024-06-20T15:30:00Z"],
  "size": 10
}
```

### 주의사항

1. **일관된 파라미터 사용**: 같은 검색 세션에서는 동일한 `pagination_depth` 값 유지
2. **메모리 관리**: `pagination_depth`와 `k` 값을 적절히 설정하여 메모리 사용량 최적화
3. **정렬 기준 선택**: 하이브리드 검색에서는 `_score`를 첫 번째 정렬 기준으로 사용 권장
4. **실시간성**: `search_after`는 실시간 데이터 변경에 민감하므로 주의 필요

## OpenSearch 후처리 필터링(Post-filtering)

후처리 필터링(Post-filtering)은 검색 결과가 모두 검색된 후에 추가적인 필터를 적용하는 기능입니다. 이는 일반적인 필터와는 완전히 다른 동작 방식을 가지며, 특별한 사용 사례에서 매우 유용합니다.
#### 후처리 필터의 특징

- **적용 시점**: 검색 결과가 완전히 계산된 **후에** 적용
- **점수 영향 없음**: 문서 관련도 점수 계산에 영향을 주지 않음
- **집계 결과 분리**: 집계는 전체 결과 기준, 표시는 필터링된 결과
- **성능 특성**: 전체 검색 후 필터링하므로 일반 필터보다 느림

#### 일반 필터 vs 후처리 필터

| 구분         | 일반 필터 (`query.bool.filter`) | 후처리 필터 (`post_filter`) |
| ---------- | --------------------------- | ---------------------- |
| **적용 시점**  | 검색 실행 시                     | 검색 결과 후                |
| **검색 범위**  | 필터링된 범위만 검색                 | 전체 범위 검색 후 필터링         |
| **점수 계산**  | 필터링된 문서만으로 점수 계산            | 전체 문서로 점수 계산 후 필터링     |
| **집계 영향**  | 집계 결과에 영향                   | 집계 결과에 영향 없음           |
| **성능**     | 빠름 (검색 범위 축소)               | 상대적으로 느림               |
| **메모리 사용** | 적음                          | 많음                     |
| **사용 목적**  | 검색 범위 제한                    | 표시 결과 선별               |

#### 일반적인 후처리 필터링

```json
GET /my-index/_search
{
  "query": {
    "match": {
      "title": "OpenSearch"
    }
  },
  "post_filter": {
    "term": {
      "status": "published"
    }
  }
}
```

#### 하이브리드 검색에서의 후처리 필터링

```json
GET /my-nlp-index/_search?search_pipeline=nlp-search-pipeline
{
  "query": {
    "hybrid": {
      "queries": [
        {
          "match": {
            "passage_text": "hello"
          }
        },
        {
          "term": {
            "passage_text": {
              "value": "planet"
            }
          }
        }
      ]
    }
  },
  "post_filter": {
    "match": { 
      "passage_text": "world" 
    }
  }
}
```

**결과 예시**

```json
{
  "took": 18,
  "timed_out": false,
  "_shards": {
    "total": 2,
    "successful": 2,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 0.3,
    "hits": [
      {
        "_index": "my-nlp-index",
        "_id": "1",
        "_score": 0.3,
        "_source": {
          "id": "s1",
          "passage_text": "Hello world"
        }
      }
    ]
  }
}
```

### 결론

후처리 필터링은 표시 결과와 집계 결과를 분리하여 관리할 수 있는 강력한 기능입니다. 특히 다음과 같은 상황에서 매우 유용합니다.

#### 적합한 사용 사례

- 권한 기반 콘텐츠 필터링
- 개인화된 결과 표시
- A/B 테스트 및 실험
- 전체 통계는 유지하되 표시만 제한

#### 피해야 할 사용 사례

- 단순한 범위 제한 (일반 필터가 더 효율적)
- 성능이 중요한 대용량 검색
- 집계 결과도 필터링이 필요한 경우