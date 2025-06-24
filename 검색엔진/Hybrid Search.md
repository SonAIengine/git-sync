하이브리드 검색이란, 키워드 매칭(keyword search)과 의미 기반(semantic) 검색을 결합하여 검색 정확도를 높이는 기능.

Opensearch에서 하이브리드 검색을 사용하려면, 검색 파이프라인(search pipeline)을 구성하고, 검색 시점에파이프라인을 통해 얻은 결과 점수를 정규화 병합하여 최종 순위를 매기는 방식.

### 사전 준비
1. 텍스트 임베딩 모델 준비
2. 플러그인 활성화(knn_vector, text_embedding, flow_framework)

