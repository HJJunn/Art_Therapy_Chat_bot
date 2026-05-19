# 🌐 HTP 도메인 Advanced RAG API 서버 구축

## 💡 프로젝트 개요

이 프로젝트는 HTP(집-나무-사람) 그림 검사 해석을 자동화하기 위한 **Advanced Conversational RAG Engine**을 구현하고, 이를 외부 서비스에서 쉽게 호출할 수 있도록 **API 서버 엔드포인트**를 정의합니다.

핵심은 **Multi-Query Rewriter (GPT-4o)**, **Cross-Encoder Reranker**, 그리고 **LoRA 파인튜닝된 Qwen2.5 LLM**을 결합하여, HTP 그림 캡션에 대한 전문적인 심리 해석을 제공하는 것입니다.

---

## 🏗️ 시스템 아키텍처 (3-Tiered RAG)



### 1. 임베딩 및 벡터 저장소 (`embeddings.py`)

| 구성 요소 | 모델/DB | 특징 |
| :--- | :--- | :--- |
| **임베딩 모델** | `HJUNN/bge-m3b-Art-Therapy-embedding-fine-tuning` | HTP 도메인에 특화된 BGE-M3 임베딩 모델 사용. |
| **Wrapper** | `MyEmbeddings` | Hugging Face `AutoModel`을 LangChain `Embeddings` 인터페이스에 맞게 래핑. |
| **벡터 DB** | `Chroma` | `htp_collection`에 HTP 문서들을 저장하며, 영구 저장(`./chroma_store`)을 지원. |
| **Reranker** | `BAAI/bge-reranker-v2-m3` | 초기 검색 결과의 순위를 재조정하는 데 사용. |

---

## 💻 RAG 엔진 상세 구현 (`rag_engine.py`)

### 1. 쿼리 재생성 및 분리 (`AdvancedQueryRewriter`)

* **목적:** 사용자의 대화 맥락과 현재 쿼리를 바탕으로 **복수의 독립적인 검색 쿼리**를 생성하여 검색 재현율(Recall)을 높입니다.
* **LLM:** `gpt-4o` (OpenAI)를 사용하여 고품질의 쿼리를 JSON 형식으로 생성하도록 유도합니다.
* **프롬프트:** 이전 대화 내용(`history_text`)을 명시적으로 포함하여 문맥 의존적인 질문도 정확히 재구성합니다.

### 2. Multi-Query Rerank Retriever (`MultiQueryRetriever`)

쿼리 재생성기와 Reranker를 통합하여 최종 검색 결과를 도출합니다.

* **검색 필터:** 검색 시 `filter={"category": category}`를 적용하여 **'집', '나무', '사람'** 등 요청된 HTP 유형 내에서만 검색을 수행합니다.
* **검색 및 Reranking 전략:**
    1. 각 재생성된 쿼리당 **k=5개**의 문서를 벡터 검색합니다.
    2. 검색된 문서들을 **Cross-Encoder**를 통해 재순위화합니다.
    3. 쿼리당 **Top-2 문서**만 최종 후보 목록에 추가하고, 문서 내용 중복을 제거합니다.
* **히스토리 관리:** 검색 품질 개선을 위해 이전 질의, 재생성된 쿼리, 검색된 문서, 최종 답변을 체계적으로 기록합니다.

### 3. 최종 응답 생성 (`AdvancedConversationalRAG`)

* **LLM:** LoRA 파인튜닝된 **`helena29/Qwen2.5_LoRA_for_HTP`** 모델을 사용하여 HTP 해석을 생성합니다.
* **프롬프트:** 시스템 역할을 **'HTP 검사 전문 심리학자'**로 지정하고, 검색된 문서를 컨텍스트로 제공하여 전문적이고 사실 기반의 **한국어** 답변을 유도합니다.
* **생성 설정:** `temperature=0.3` 등으로 설정하여 창의성보다는 정확성과 일관성을 우선시합니다.

---

## 🌐 API 엔드포인트 정의 (Web Integration)

구현된 RAG 엔진을 서비스화하기 위해 두 가지 주요 API 엔드포인트를 정의합니다 (예시 코드에 따라 FastAPI로 가정).

### 1. 이미지 캡션 생성 API (Pre-step)

| 경로 | 메서드 | 입력 (`CaptionRequest`) | 출력 | 역할 |
| :--- | :--- | :--- | :--- | :--- |
| `/caption` | POST | `image_base64` (Base64 인코딩된 이미지) | `{"caption": "생성된 캡션 문자열"}` | HTP 그림을 텍스트 캡션으로 변환 (외부 함수 `generate_caption` 필요). |

### 2. RAG 검색 및 해석 API (Core Service)

| 경로 | 메서드 | 입력 (`RagRequest`) | 출력 | 역할 |
| :--- | :--- | :--- | :--- | :--- |
| `/rag` | POST | `caption` (사용자 질의/캡션), `image_type` ("집"\|"나무"\|"사람") | `{ "result": "최종 답변", "rewritten_queries": [...], "source_documents": [...] }` | HTP 그림 캡션과 유형을 받아 최종적인 심리 해석을 제공. |
