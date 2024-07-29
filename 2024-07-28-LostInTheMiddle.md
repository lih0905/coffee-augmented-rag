## [논문 리뷰] Lost in the Middle: How Language Models Use Long Contexts

스탠포드, 버클리, Samaya AI에서 작성한 [Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/pdf/2307.03172) 논문을 퀵하게 요약한 내용 및 본문의 실험을 Llama-3.1-8b 모델에 대해 실행해본 결과입니다.

## 요약

- Multi-document question answering과 key-value retrieval 두가지 task를 이용하여 분석한 결과, 전체 context에서 relevant information의 위치가 어디인 지에 따라 모델의 성능이 크게 달라짐
- Relevant information의 위치가 가장 처음 혹은 나중일 때 성능이 높으며, 가운데 쪽에 위치할 경우 아예 content 없이 질문만 한 경우보다 낮아지기도 함
- RAG 성능 개선을 위해서는 (1) 효율적인 document reranking(relevant information을 앞쪽에 배치) (2) ranked list truncation이 필요
- Llama-3.1-8b 모델은 본문에서 실험한 모델들을 훨씬 상회하는 성능을 보이나 U-커브는 여전히 발생

## Multi-Document Question Answering

* 주어진 질문에 대해 단 한 개의 정답을 추출할 수 있는 문서를 포함한 k개의 문서를 context로 구성함
  * 정답 문서의 위치 / 정답 미포함 문서 갯수(k-1)를 바꿔가며 성능을 평가함
  * 모델은 MPT-30B-Instruct(8129토큰), LongChat-13B(16K), GPT-3.5-Turbo(4K, 16K), Claude-1.3(8K, 100K)를 사용함
* Relevant information이 처음과 끝부분에 있을 때 성능이 높게 나타나며, 가운데로 가면 급격하게 성능 저하
  ![image1](https://github.com/lih0905/coffee-augmented-rag/blob/main/images/240728/image1.png)
* 짧은 문장에 대해 학습 시킨 후 긴 문장으로 확장한 모델은 짧은 문장 이하의 길이에서는 원래 모델과 확장 모델의 성능 차이가 거의 나타나지 않음

## LLM의 입력 정보로부터의 추론 능력 확인

* 언어에 대한 이해를 배제하고 추론 능력만을 확인하기 위해 UUID들로 이루어진 key-value셋을 구축, 임의의 key를 입력으로 받았을 때 해당 value를 리턴하도록 데이터셋을 구축
* Claude-1.3은 거의 완벽하게 다 맞추는 반면 GPT는 데이터의 길이가 길 경우 U-커브가 나타남
  ![image2](https://github.com/lih0905/coffee-augmented-rag/blob/main/images/240728/image2.png)

## 왜 정보 위치의 변화에 취약한 지에 대한 추론

* 모델의 구조(decoder-only, encoder-decoder) 차이로 인해 이런 결과가 나오는지 알아보기 위해 encoder-decoder 구조의 모델(Flan)을 사용해보니, 데이터셋의 길이가 짧은 경우에는 위치 변화에 강건하나 길어지면서 U-커브 다시 기록
* 질문을 입력의 마지막 뿐 아니라 처음에도 넣어서 확인해본 결과, key-value task의 성능은 크게 올라가지만 qa 태스크에는 거의 차이가 없으며 오히려 위치 강건성은 더 나빠짐
* Instruct 파인 튜닝한 모델은 원본 모델에 비해서 성능이 상당히 개선되며, 특히 성능이 낮은 중간 부분에서의 개선폭이 큼

## Context 길이에 따른 성능

* 입력 document의 길이가 20개를 넘어가면 성능 개선 폭이 미미해짐
  ![image3](https://github.com/lih0905/coffee-augmented-rag/blob/main/images/240728/image3.png)

## Llama-3.1-8b 모델로 논문 task 수행한 결과

* Closed-book/oracle 정확도

  * 본문에 나온 기존 모델들 결과
    ![image5](https://github.com/lih0905/coffee-augmented-rag/blob/main/images/240728/image5.png)

  * Llama-3.1-8b(+Instruct) 모델을 이용해서 재실험한 결과

    * 사이즈가 작은 모델임에도 Oracle은 GPT3.5보다도 뛰어난 성능을 보임

    |                       | Closed-book | Oracle |
    | --------------------- | ----------- | ------ |
    | Llama-3.1-8b          | 34.5%       | 81.9%  |
    | Llama-3.1-8b-Instruct | 44.0%       | 89.1%  |

* 20 Total Retrieved Documents (~4K tokens)

  * 본문에 나온 Llama-2 모델의 성능 평가
    ![image4](https://github.com/lih0905/coffee-augmented-rag/blob/main/images/240728/image4.png, width=250)

  * Llama-3.1-8b(+Instruct) 모델을 이용해서 재실험한 결과

    * 기존 가장 좋은 성능을 보이던 Llama-2-70b-chat 보다도 훨씬 좋은 성능을 보이지만 U-커브는 여전히 발생
    * Llama-3.1-8b-Instruct 모델은 U-커브는 완화되나 여전히 문맥 처음에 등장할 때 가장 성능이 뛰어남

    | Index                 | 1st   | 5th   | 10th  | 15th  | 20th  |
    | --------------------- | ----- | ----- | ----- | ----- | ----- |
    | Llama-3.1-8b          | 68.9% | 55.4% | 55.4% | 55.3% | 63.7% |
    | Llama-3.1-8b-Instruct | 73.8% | 67.4% | 66.4% | 66.0% | 67.9% |

    !['image6'](https://github.com/lih0905/coffee-augmented-rag/blob/main/images/240728/image6.png, width=300)