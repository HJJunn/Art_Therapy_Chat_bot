# 🔥 Reranker 파인튜닝 실험: HTP 도메인 검색 최적화

## 💡 프로젝트 개요

이 프로젝트는 기존에 파인튜닝된 임베딩 모델(BGE-M3)의 검색 결과(Recall)를 개선하기 위해, **Cross-Encoder Reranker**를 HTP(집-나무-사람) 도메인 데이터로 추가 파인튜닝하는 과정을 담고 있습니다.

다양한 **손실 함수(Loss Function)**와 **네거티브 샘플링 전략**을 실험하여, 검색 순위(Precision)를 극대화하는 최적의 Reranker 모델을 찾습니다.

---

## 🛠️ 기술 스택 및 데이터셋

* **Reranker 모델:** `BAAI/bge-reranker-base` (베이스라인)
* **데이터셋:** `HJUNN/Art_Therapy_caption_train_dataset` (Query-Document Positive Pairs)
* **임베딩 모델 (HNS용):** `HJUNN/bge-m3b-Art-Therapy-embedding-fine-tuning` (Hard Negative Sampling에 사용)
* **평가 지표:** Reranking 후 순위 기반 점수 (`calculate_similarity` 함수를 통한 Softmax 기반 순위 확인)

---

## 🔬 주요 실험 내용

Reranker 파인튜닝은 Query-Positive Document-Negative Document **삼중항(Triple)**을 구성하여 진행됩니다.

### 1. 네거티브 샘플링 전략 (Negative Sampling)

| 전략 | 설명 | 활용 손실 함수 |
| :--- | :--- | :--- |
| **하드 네거티브 샘플링 (HNS)** | 쿼리 임베딩과 **유사도는 높으나 정답은 아닌** 문서를 선택합니다. 코사인 유사도 0.30~0.60 사이의 문서를 필터링하여 선정했습니다. | MarginRankingLoss, BCE Loss |
| **이지 네거티브 샘플링 (ENS)** | **랜덤하게** 문서를 선택하거나, 관련성이 매우 낮은 문서를 선택합니다. (코드에는 명시되지 않았지만 대비되는 개념으로 사용됨) | Pairwise Loss, MSE Loss |

### 2. 손실 함수 실험 (Loss Function)

두 가지 주요 손실 함수를 비교했습니다.

#### A. MarginRankingLoss (마진 기반 안정화)

| 내용 | MarginRankingLoss |
| :--- | :--- |
| **목표** | `Pos_Score` ≥ `Neg_Score` + `Margin`이 되도록 학습. |
| **장점** | 과도하게 스코어를 벌리지 않아 **점수 분포가 자연스럽게 유지**되고, 극단적인 값(0.999 vs 0.000)으로 발산하는 것을 방지합니다. (`Loss: 0.1780` → `0.0525`로 안정화됨) |
| **사용** | 하드 네거티브 삼중항을 사용하여 학습. |

#### B. BCEWithLogitsLoss (이진 분류 기반)

| 내용 | BCE Loss (with Label Smoothing) |
| :--- | :--- |
| **목표** | Positive Pair는 1(0.9)로, Negative Pair는 0(0.1)로 분류하도록 학습. |
| **특징** | **Label Smoothing**을 적용하여 Positive(0.9)와 Negative(0.1) 레이블을 사용하여 과적합을 방지하고 일반화 성능을 높입니다. |
| **사용** | 하드 네거티브 삼중항을 사용하여 학습. |



> * **그 외 실험:**  **CrossEntropyLoss**와 **MSE Loss**를 사용하여 랭킹 최적화 실험 및 **`bert-base`** 모델을 활용한 기본 모델 교체 실험도 진행되었습니다.

---

## 🔎 평가 함수 (Scoring Function)

파인튜닝된 Reranker의 성능을 확인하기 위해 `calculate_similarity` 함수를 정의합니다.

