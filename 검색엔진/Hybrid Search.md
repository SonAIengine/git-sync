하이브리드 검색이란, 키워드 매칭(keyword search)과 의미 기반(semantic) 검색을 결합하여 검색 정확도를 높이는 기능.

Opensearch에서 하이브리드 검색을 사용하려면, 검색 파이프라인(search pipeline)을 구성하고, 검색 시점에파이프라인을 통해 얻은 결과 점수를 정규화 병합하여 최종 순위를 매기는 방식.

### 사전 준비
1. 텍스트 임베딩 모델 준비
2. 플러그인 활성화(knn_vector, text_embedding, flow_framework)


### Paginating hybrid query results
하이브리드 검색의 결과 페이지네이션
1. pagination_depth 의 역할
	- 각 샤드별로, 하이브리드 쿼리의 서브쿼리(벡터/불린 등)에서 최대 몇 개의 문서를 메모리에 유지할지 지정한다.
	- 예를 들어 pagination_depth: 50 으로 설정하면, 각 샤드에서 최대 50개의 후보 결과를 모아서 랭킹 후 병합하게 된다.
2. from과 size 파라미터
3. 페이지 간 일관성 유지
	- pagination_depth 를 변경하면, 실제 후보(pool)의 범위가 달라져 최종 결과 순서가 바뀔 수 있음.
	- 따라서 페이지를 넘길 때는 처음부터 끝까지 동일한 pagination_depth 값을 사용해야 일관된 페이징이 보장됩니다.
4. 깊은 페이지네이션(deep pagination)
	- 더 뒤쪽 페이지를 보고 싶다면 pagination_depth 를 충분히 크게 늘려야 한다.
	- 단, pagination_depth 를 높일수록 각 샤드에서 더 많은 후보를 메모리에 올려 처리하므로 성능 저하(메모리, CPU 증가)가 발생할 수 있음.

```
{
  "from": 0,
  "size": 5,
  "hybrid": {
    "query": { … },
    "vector_query": { … },
    "pagination_depth": 10
  }
}
```

### Hybrid search with search_after
1. search_after 개념
	- 라이브 커서(cursor) 역할
	- 이전 페이지에서 마지막으로 반환된 문서의 정렬 키 값을 다음 요청에 넘겨주면, 그 뒤부터 결과를 이어서 가져올 수 있음.
	- 큰 from 오프셋을 사용하지 않아도 돼서 성능 상 이점
2. 첫 번째 페이지 요청
    `GET /my-nlp-index/_search?search_pipeline=nlp-search-pipeline {   "query": { /* hybrid query */ },   "sort": [{ "doc_price": { "order": "desc" } }],   "size": 2 }`
    
	예시 결과의 `hits.sort`가 `[350]`, `[200]` 등으로 나옴
        
3. 두 번째 페이지 요청    
	`GET /my-nlp-index/_search?search_pipeline=nlp-search-pipeline {   "query": { /* 동일한 hybrid query */ },   "sort": [{ "doc_price": { "order": "desc" } }],   "search_after": [200],   "size": 2 }`
    
	doc_price가 200 이하인 문서부터 다시 2건을 가져옴
        

## 4. 예시 정리

- doc_price 기준 페이지네이션
    `"sort": [{ "doc_price": { "order": "desc" } }], "search_after": [200]`
    
     `doc_price`가 200보다 작은 순서로 뒤쪽 결과 반환
    
- **`_id` 기준 페이지네이션**
    
    json
    
    코드 복사
    
    `"sort": [{ "_id": { "order": "desc" } }], "search_after": ["7yaM4JABZkI1FQv8AwoN"]`
    
    → `_id`가 `"7yaM4JABZkI1FQv8AwoN"`보다 뒤에 오는 ID 문서 반환