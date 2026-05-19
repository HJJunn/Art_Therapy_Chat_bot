# 🤖 HTP 도메인 특화 Advanced RAG 챗봇 파이프라인

## 💡 프로젝트 개요

이 프로젝트는 HTP(집-나무-사람) 해석 문서 데이터를 활용하여, **대화 이력 기반 쿼리 재생성** 및 **Reranking** 기능을 포함하는 **고급 대화형 RAG 챗봇 시스템**을 구축합니다.

목표는 사용자의 HTP 그림 특성에 대한 질문에 대해, 도메인 지식 기반의 정확하고 전문적인 심리 해석 답변을 제공하는 것입니다.

---

## 🛠️ 기술 스택 및 구성 요소

| 구성 요소 | 모델 / 라이브러리 | 역할 |
| :--- | :--- | :--- |
| **임베딩 모델** | `HJUNN/bge-m3b-Art-Therapy-embedding-fine-tuning` | HTP 도메인 특화 벡터 임베딩 생성 |
| **벡터 데이터베이스** | `Chroma` | `langchain_docs`를 저장하고 벡터 검색 수행 |
| **Reranker** | `BAAI/bge-reranker-v2-m3` | 초기 검색 결과(k=20)를 재순위화(top_k=10)하여 정확도 향상 |
| **쿼리 재생성기** | `OpenAI (GPT-4o)` | 이전 대화(History)를 기반으로 Multi-Query를 생성하여 검색 품질 개선 |
| **응답 생성 LLM** | `helena29/Qwen2.5_LoRA_for_HTP` | HTP 데이터로 LoRA 파인튜닝된 LLM으로 최종 답변 생성 |

---

## 💻 핵심 파이프라인 상세

### 1. 벡터 DB 구축 및 임베딩 (`MyEmbeddings` Class)

* **모델 로드:** `HJUNN/bge-m3b-Art-Therapy-embedding-fine-tuning` 모델을 로드하고, `get_embedding` 함수를 정의합니다.
* **LangChain Wrapper:** `MyEmbeddings` 클래스를 정의하여 `AutoModel` 기반의 임베딩 모델을 LangChain `Embeddings` 인터페이스에 맞게 래핑합니다.
* **DB 생성:** `Chroma.from_documents`를 사용하여 `langchain_docs` (사전 청킹된 HTP 문서)를 벡터 저장소에 저장하고 영구화합니다.

### 2. Reranker 통합 리트리버 (`Retriever_with_cross_encoder`)

초기 검색 결과의 순위를 개선하기 위해 Cross-Encoder를 통합합니다.

* **초기 검색:** `vectorstore.similarity_search`를 사용하여 쿼리당 **k=20개**의 문서를 검색합니다.
* **Reranking:** 검색된 문서와 쿼리 쌍을 `BAAI/bge-reranker-v2-m3` 모델에 입력하여 점수를 계산합니다.
* **최종 결과:** 점수가 높은 순서대로 **rerank_top_k=10개**의 문서를 최종적으로 반환합니다.

### 3. 대화형 쿼리 재생성 (`AdvancedQueryRewriter`)

대화의 문맥을 이해하고 모호한 쿼리를 구체적인 검색 쿼리로 확장합니다.

* **LLM 사용:** `ChatOpenAI(gpt-4o)`를 사용하여 쿼리 재생성 품질을 높입니다.
* **프롬프트:** 이전 대화 기록(`history_text`)과 현재 질문(`current_query`)을 명시적으로 제공하고, **Multi-Query JSON 형식**으로 출력하도록 강제합니다.

### 4. 멀티 쿼리 검색 (`MultiQueryRetriever`)

생성된 여러 개의 쿼리를 모두 사용하여 문서를 검색하고 결과를 통합합니다.

* **검색:** `AdvancedQueryRewriter`가 생성한 각 쿼리에 대해 `vectorstore.similarity_search`를 실행합니다.
* **통합 및 정제:** 모든 검색 결과를 병합하고, 문서 내용이 중복되는 경우 제거하여 고유한 문서만 유지합니다.

### 5. 최종 응답 생성 (`AdvancedConversationalRAG`)

검색된 문서를 기반으로 최종 답변을 생성하고 대화 이력을 관리합니다.

* **LLM 로드:** HTP 데이터로 LoRA 파인튜닝된 `helena29/Qwen2.5_LoRA_for_HTP` 모델을 로드합니다.
* **프롬프트:** 전문적인 심리학자 역할(`role: system`)을 부여하고, 검색된 컨텍스트를 포함하여 Qwen Chat 형식으로 포맷팅합니다.
* **응답:** LLM의 `generate` 함수를 사용하여 HTP 해석 가이드라인에 따른 최종 답변을 한국어로 생성합니다.
* **히스토리 관리:** 모든 쿼리, 재생성된 쿼리, 검색된 문서, 최종 답변을 `history`에 저장하여 다음 대화의 문맥으로 활용합니다.

---

## 🚀 실행 예시 분석

| 예시 쿼리 | 재생성 쿼리 | 검색 결과 | 해석 특징 |
| :--- | :--- | :--- | :--- |
| **"손이 크고, 눈썹이 진한 사람 그림"** | 2개로 분리: '손이 큰 사람 그림', '눈썹이 진한 사람 그림' | 2개 고유 문서 | **손 (impulsivity, aggression)** + **눈썹 (guarded personality)**의 복합 해석 제공. |
| **"손가락을 자세히 나타냈어"** | 2개로 분리: '손가락을 자세히 나타낸 그림', '손가락의 세부 묘사' | 1개 고유 문서 | **대인 관계 인식** 및 **사회적 단서에 대한 민감성** 측면에서 해석 제공. |
| **"창문이 있고, 굴뚝이 있는 집"** | 2개로 분리: '창문이 있는 집 그림', '굴뚝이 있는 집 그림' | 2개 고유 문서 | **창문 (2차적 인간관계, 외부환경 인식)**과 **굴뚝 (성적 충동, 가족적 따뜻함)**에 대한 독립된 해석을 통합하여 제시. |
