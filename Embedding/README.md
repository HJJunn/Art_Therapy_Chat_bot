# 🎨 RAG 임베딩 모델 파인튜닝: 미술 치료 캡션 검색 최적화

## 💡 프로젝트 개요

이 프로젝트는 **미술 치료 문서**에서 추출한 텍스트 데이터를 기반으로 임베딩 모델의 검색 성능을 향상시키는 것을 목표로 합니다. Retrieval-Augmented Generation (RAG) 시스템의 정확도를 높이기 위해, 범용 모델인 **BAAI/bge-m3**를 미술 치료 분야에 특화된 캡션-문서 쌍으로 파인튜닝합니다.

핵심 과정은 다음과 같습니다:

1.  **데이터 증강 (Data Augmentation):** GPT-4o를 사용하여 원본 문서에서 **다양하고 객관적인 그림 캡션(Query)**을 생성합니다.
2.  **모델 학습 (Fine-tuning):** 생성된 캡션(Query)과 원본 문서(Document) 쌍을 **Multiple Negatives Ranking Loss (MNRL)** 기법으로 학습시켜, 도메인 특화된 임베딩 공간을 형성합니다.
3.  **성능 검증:** 파인튜닝된 모델의 **Information Retrieval (IR)** 성능을 평가하고, 결과를 Hugging Face Hub에 공유합니다.

---

## 🛠️ 기술 스택 (Tech Stack)

| 구분 | 기술 / 라이브러리 | 용도 |
| :--- | :--- | :--- |
| **기반 모델** | `BAAI/bge-m3` | 초기 임베딩 모델 |
| **학습 프레임워크** | `SentenceTransformer` | 임베딩 모델 파인튜닝 및 평가 |
| **데이터 증강** | `OpenAI (GPT-4o)` | 문서 기반 Query (캡션) 생성 |
| **데이터 처리** | `LangChain`, `tqdm`, `pandas` | 문서 로드 및 전처리, 학습 데이터셋 구성 |
| **데이터 공유** | `HuggingFace Hub` | 데이터셋 및 파인튜닝된 모델 저장 |
| **손실 함수** | `MultipleNegativesRankingLoss` | 검색 성능 최적화 학습 |

---
## 💻 주요 실행 단계
### 1. 데이터 증강 및 학습 데이터 준비

* **GPT-4o 캡션 생성:** 원본 문서를 기반으로 각 문서당 3개의 다양한 캡션(Query) 쌍을 생성합니다.
* **데이터셋 변환:** `(query, doc)` 쌍을 `InputExample` 형태로 변환하여 학습에 사용합니다.

### 2. Sentence Transformer 파인튜닝

BGE-M3 모델에 생성된 학습 데이터를 적용하여 MNRL 손실 함수로 학습합니다.

| 설정 | 값 |
| :--- | :--- |
| **모델** | `BAAI/bge-m3` |
| **Epochs** | `2` |
| **Batch Size** | `32` |
| **Warmup Steps** | `10%` |
| **저장 경로** | `exp_finetune` |

### 3. 성능 평가 (Information Retrieval)

파인튜닝된 모델과 원본 모델의 검색 성능을 비교합니다.

* `집-나무-사람 논문 수기본.txt`를 전처리하여 검증 코퍼스를 구성합니다.
* GPT-4o를 이용해 검증용 쿼리(`val_queries`)를 생성합니다.
* `InformationRetrievalEvaluator`를 사용하여 NDCG, MAP, Recall 등의 지표를 비교합니다.

### 4. 결과 공유 (Hugging Face Hub)

최종 결과물은 다음 저장소에 업로드됩니다.

* **학습 데이터셋:** `HJUNN/Art_Therapy_caption_train_dataset`
* **파인튜닝 모델:** `HJUNN/bge-m3b-Art-Therapy-embedding-fine-tuning`
