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


```json
{
  "name": "YJ Kim",
  "email": "x2bee@plateer.com",
  "skills": ["Java", "Kotlin", "SQL"]
}

