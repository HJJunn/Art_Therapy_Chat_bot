# 🚀 HTP (집-나무-사람) 도메인 특화 Advanced RAG 시스템

## 💡 프로젝트 개요

본 프로젝트는 **HTP(House-Tree-Person) 그림 검사 해석 문서**를 활용하여, 대화 문맥 이해 기반 쿼리 재생성, Multi-modal 데이터 추출, Reranker 파인튜닝, 그리고 최종적으로 **전문적인 심리 해석을 제공하는 RAG 챗봇 API 서버**를 구축하는 것을 목표로 합니다.

## ✨ 시스템 아키텍처 및 파이프라인 개요

<img width="2206" height="674" alt="image" src="https://github.com/user-attachments/assets/8604cab2-f936-412e-a865-cdb98e1d1bde" />


프로젝트는 크게 세 단계로 나뉩니다.

1.  **데이터 엔지니어링:** 복잡한 PDF에서 텍스트를 추출하고, RAG에 적합하도록 청킹 및 카테고리 태깅을 수행합니다.
2.  **검색 모델 최적화:** 임베딩 모델과 Cross-Encoder Reranker를 HTP 데이터로 파인튜닝하여 검색 정확도를 극대화합니다.
3.  **서비스화:** 최적화된 컴포넌트들을 통합하여 대화형 RAG API 서버를 구축합니다.

---

## 1. 데이터 엔지니어링 및 코퍼스 구축

### 1.1. 텍스트 청킹 및 정제

* **목표:** RAG 검색에 적합한 독립적 의미 단위(청크)를 생성합니다.
* **전략:** 번호(`\d+\.`) 및 기호(`■`, `*`) 기반의 계층적 청킹을 수행합니다.
* **메타데이터:** 모든 청크에 '집', '나무', '사람' 카테고리를 할당하고, 수동 보정을 통해 데이터 정확도를 높입니다.

### 1.2. PDF 멀티모달 텍스트 추출

* **문제:** HTP 해석본 PDF의 복잡한 테이블 구조에서 텍스트 누락 및 구조 손실 방지.
* **전략:**
    * `fitz`를 사용하여 페이지별 **PNG 이미지** (시각적 레이아웃)와 **XML 텍스트** (디지털 텍스트)를 추출합니다.
    * **GPT-4.1-mini** LLM에 이미지와 XML을 동시 입력하여, 테이블 구조를 해석하고 누락 없이 텍스트를 구조화하도록 지시합니다.

---
## 2. RAG 임베딩 모델 파인튜닝 

**미술 치료 캡션 검색 최적화**를 위해 범용 BGE-M3 모델을 도메인 특화 모델로 변환합니다. 

### 2.1. 데이터 증강 및 학습 데이터 준비

* **GPT-4o 캡션 생성:** 원본 문서를 기반으로 각 문서당 **다양하고 객관적인 그림 캡션(Query)** 쌍을 생성합니다.
* **데이터셋 변환:** 생성된 `(Query, Document)` 쌍을 `SentenceTransformer`의 `InputExample` 형태로 변환합니다.

### 2.2. Sentence Transformer 학습

* **기반 모델:** `BAAI/bge-m3`
* **손실 함수:** **Multiple Negatives Ranking Loss (MNRL)** 기법을 사용하여, 배치 내에서 Negative 샘플과의 거리는 멀리하고 Positive 쌍의 거리는 가깝게 만듭니다.
* **학습 설정:**
    * **Epochs:** 2
    * **Batch Size:** 32
    * **Warmup Steps:** 10%
* **결과 모델:** `HJUNN/bge-m3b-Art-Therapy-embedding-fine-tuning`

### 2.3. 성능 평가

* **지표:** Information Retrieval Evaluator를 사용하여 **NDCG, MAP, Recall** 등의 지표를 통해 원본 모델 대비 검색 성능 향상을 검증합니다.

----
## 3. Reranker 모델 파인튜닝 실험

검색 결과의 순위 정확도(Precision)를 높이기 위해 Cross-Encoder를 파인튜닝합니다.

### 3.1. 네거티브 샘플링 전략 (Negative Sampling)

| 전략 | 설명 | 활용 손실 함수 |
| :--- | :--- | :--- |
| **하드 네거티브 샘플링 (HNS)** | 쿼리 임베딩과 **유사도는 높으나 정답은 아닌** 문서를 선택합니다. 코사인 유사도 0.30~0.60 사이의 문서를 필터링하여 선정했습니다. | MarginRankingLoss, BCE Loss |
| **이지 네거티브 샘플링 (ENS)** | **랜덤하게** 문서를 선택하거나, 관련성이 매우 낮은 문서를 선택합니다. (코드에는 명시되지 않았지만 대비되는 개념으로 사용됨) | Pairwise Loss, MSE Loss |


### 3.2. 손실 함수 비교 실험 결과

| 실험 모델 | 손실 함수 | 학습 방식 | 주요 특징 및 결과 |
| :--- | :--- | :--- | :--- |
| `bge-reranker-base` | **MarginRankingLoss** | Pairwise Ranking | Pos/Neg 스코어 차이(Margin)를 유지하여 스코어 발산을 방지하고 안정적인 점수 분포를 유도. |
| `bge-reranker-v2-m3` | **CrossEntropyLoss** | Pairwise Ranking | Pos 로짓이 Neg 로짓보다 높도록 분류 문제로 접근하여 학습 (가장 안정적). |
| `bge-reranker-base` | **BCEWithLogitsLoss** | Pairwise Classification | Label Smoothing(0.9/0.1) 적용을 통해 일반화 성능 향상 시도. |
| `klue/bert-base` | **MSE Loss** | Pairwise Regression | 경량 모델(`klue/bert-base`)을 활용한 랭킹 학습 실험. |

---

## 4. Advanced Conversational RAG Engine 구현

최적화된 컴포넌트를 통합하여 API 서비스가 가능한 대화형 RAG 엔진을 구축합니다.

### 4.1. 임베딩 및 벡터 DB 설정

* **임베딩 모델:** `HJUNN/bge-m3b-Art-Therapy-embedding-fine-tuning`
* **DB:** `Chroma` (./chroma\_store)
* **Wrapper:** `MyEmbeddings` 클래스를 사용하여 `AutoModel` 기반 임베딩을 LangChain 인터페이스에 맞게 통합.

### 4.2. 핵심 검색 체인 (Retriever)

| 모듈 | 기술 / 모델 | 역할 |
| :--- | :--- | :--- |
| **쿼리 재생성** | `AdvancedQueryRewriter` (`GPT-4o`) | 대화 기록을 기반으로 모호한 쿼리를 **Multi-Query JSON** 형식으로 재구성. |
| **검색** | `MultiQueryRetriever` | 재생성된 쿼리별로 검색 후 결과를 통합 및 중복 제거. **카테고리 필터링**(`filter={"category": category}`) 적용. |
| **재순위화** | `CrossEncoder` ('HJUNN/bge_BCE_cross_encoder') | 초기 검색 결과(k=5)에 대해 Reranking을 수행하여 쿼리당 **Top-3** 문서를 최종 선정. |

### 4.3. 최종 응답 생성 (LLM)

* **모델:** `helena29/Qwen2.5_LoRA_for_HTP` (HTP 파인튜닝 LLM)
* **프롬프트:** 검색된 컨텍스트와 함께 전문 심리학자 역할을 부여하여, **사실 기반의 전문적인 한국어 해석** 답변을 생성하도록 유도합니다.
* **대화 관리:** `history` 리스트에 이전 대화 기록을 저장하고, 다음 쿼리 재생성에 활용하여 대화 문맥을 유지합니다.

---
