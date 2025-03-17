# LLM_FineTurning

# HAI 내에서 진행하는 해당 프로젝트에 대한 문서임.
1. 서버 메뉴얼을 필독 후 토치 또는 텐서의 원리를 이해 한 이후에 접근 하는 것을 추천하는 바임
2. 기본적으로 사용하는 Unsloth과 그에 딸려오는 Transformer 등 프로그램이 사용하는 CUDA의 원리를 이해하고 시작하여야 함
3. 사용하게 되는 Model 은 DeepSeek-R1-Distill-Qwen-32B-unsloth-bnb-4bit 이며 해당 모델은 R1의 증류 모델이고 32B이며 4Bit 양자화가 된 모델이라는 뜻임

4. 202032003 길경민


# 참조한 논문
https://arxiv.org/pdf/2502.07316

# DeepSeek R1 정리
DeepSeek의 추론엔진  

흔히 아는 LLM의 추론 방식은 총 두가지 입니다. <br/>
수학문제 데이터 : 문제를 주고 이를 수학적으로 추론해가며 과정을 학습하는 방법 <br/>
Chain-of-Thought 데이터 : 사람 생각처럼 해결 방식 트리를 사용하는 방식

DeepSeek 는 이걸 Codel/O 로 학습을 시킨게 차이점임 (추론엔진)
맥락에서의 논리 흐름 계획 상태공간 탐색 의사결정 트리 순회 등을 특징으로 봄
즉 이걸 통해서 기본 LLM들의 코드 문법으로 부터의 분리를 해내게 되었음.

결국은 위에서 기술한 전통적 학습법은 결국 일반화의 한계가 존재하게 됩니다.

Input/Output 을 예측하여서 CoT 추론을 사용하게 된다면,
이는 기존 문법에서 벗어나서 구조화된 추론을 가능하게 됩니다.

또한 동시에 절차적 엄밀성을 유지하여 다양한 패턴의 학습이 가능하게 됩니다.
허나 이는 수학적 추론, 코드 구성 등에서 이점을 가지나 그 외의 추론에서는 성능 저하를 가져오는 문제가 있습니다.
그 제외되는 추론의 항목은 일반적인 자연어 CoT 데이터는 추론이 이미 명확하게 정의되어 있지 않아 일관성이 부족할 수 있으며,
그리고 수학 문제는 특정한 유형의 추론에만 집중하기 때문에 더 다양한 논리적 사고를 학습하기 어렵다. 라는 문제를 가지는 것 입니다.

기존 방법과의 차별점
기존 CoT는 사람이 작성한 데이터를 사용하여 일관성이 없을 수 있고 편향된 성능을 가지는 문제가 있었으나,
하지만 CodeI/O에서는 i/o 관계를 이용하여 CoT 자동 생성을 하게 되었고,
코드 실행과정은 논리적으로 엄밀 다양한 추론 패턴을 학습이 가능해졌습니다.

문제 제기
기존의 LLM은 수학 문제 해결이나 코드 생성과 같은 특정 분야에서는 풍부한 학습 데이터를 통해 성능을 향상시켜 옴 허나 논리적 추론, 과학적 추론, 상징적 추론 등 다양한 추론 학습에서는 학습 데이터의 부족과 단편성으로 성능 향상에 한계가 있었습니다.

제안된 방법
Code i/o 를 이용하여 코드에 내제된 다양한 추론 패턴을 체계적으로 추출하여 자연어 형태의 i/o 예측 데이터로 변환하는 방법을 제안하였는데

    def factorial(n):
            if n == 0:
                return 1
            return n * factorial(n-1)
            
    print(factorial(3))

이걸 자연어로 이해시킨다고 생각해야 합니다 이는 하단의 글처럼 풀이가 가능합니다.

factorial(3)을 호출한다. <br/>
3 != 0이므로, 3 * factorial(2)을 계산해야 한다. <br/>
factorial(2)을 호출한다. <br/>
2 != 0이므로, 2 * factorial(1)을 계산해야 한다. <br/>
factorial(1)을 호출한다. <br/>
1 != 0이므로, 1 * factorial(0)을 계산해야 한다. <br/>
factorial(0)을 호출한다. <br/>
0 == 0이므로, factorial(0) = 1을 반환한다. <br/>
이를 이용해, factorial(1) = 1 * 1 = 1이 된다. <br/>
이를 이용해, factorial(2) = 2 * 1 = 2가 된다. <br/>
이를 이용해, factorial(3) = 3 * 2 = 6이 된다. <br/>

이를 통하여 모델이 코드의 구문에 얽매이지 않은 채 논리적 흐름 계획, 상태 공간 탐색,의사 결정 트리 순회, 모듈식 분해 등의 범용 추론 능력을 학습이 가능해지는 이야기 입니다.

그림으로 설명합니다.

            ROOT
             |
           [x < 5]
           /    \
       YES      NO
       /         \
    [x < 3]      [x < 8]
     /    \      /    \
    A      B    C      D

DFS/BFS 같은 순회
Root → Left → Left (A) → Backtrack → Right (B) → Backtrack → Right (C) → Right (D)
Root → Left → Right → Left Child (A, B) → Right Child (C, D)

이걸 CodeI/O 에서 접근하는 방식으로 설명합니다.

    def decision_tree(x):
        if x < 5:
            if x < 3:
                return "A"
            else:
                return "B"
        else:
            if x < 8:
                return "C"
            else:
                return "D"
    
    print(decision_tree(2))  # "A"
    print(decision_tree(4))  # "B"
    print(decision_tree(6))  # "C"
    print(decision_tree(9))  # "D"

트리 구조로 보게 된다면.

             x < 5 ?
            /        \
        Yes          No
        /             \
     x < 3 ?         x < 8 ?
     /    \         /      \
    A      B       C        D

학습 과정 <br/>
모델은 주어진 code and TestCase 를 기반으로 자연어 형태의 CoT 추론을 통하여 
i/o 를 예측하도록 훈련이 됩니다. 이 과정에서 모델은 구체적인 구문에서 벗어나, 절차적 엄밀성을 유지하면서도 다양한 추론 패턴을 내재화 하게 됩니다.

예시 상황 <br/>
문제 : 주어진 정수 N에 대해 1부터 까지의 모든 수 중에서 홀수만 출력하는 함수를 작성하십시오. 그리고 리스트로 출력하십시오

1안 <br/>
우선 input을 5로 줍니다
그러면 당연하게도 [1,3,5] 라는 값을 반환하게 될 겁니다.

2안<br/>
input = 5 로 동일합니다.

문제 이해 = 상단과 동일합니다
입력 분석 : 입력값은 5입니다. -> 이는 함수가 1부터 input값 까지 수를 검사해야 한다는 뜻 
출력 예측: 1,3,5 는 홀수니까 (//2) 해서 나누면 당연히 홀 짝 구분이 될 테니까 이를 List에 append 해주는 과정을 합니다

    def get_odds(N):
    return [x for x in range(1, N+1) if x % 2 == 1]

즉 이 함수는 1부터 N(input) 까지의 수를 반복하고 홀수만 얻어내고 List로 Return 해 주는 함수입니다.

이런 방식으로 두가지로 서로 나누어 학습하는 과정을 가지는 원리라고 정리가 가능합니다.

검증 및 성능 향상 <br/>
에측된 i/o 을 실제 정답과 비교하거나 code를 재실행해서 결과를 검증함
해당 과정을 통하여 CoT를 다단계로 수정하는 CodeI/O++ 방법도 추가적으로 제시합니다.

# 정리
제안 된 CodeI/o 방식은 상징적, 과학적, 논리적, 수학적 추론 작업에서 일관된 성능의 향상을 확인하였습니다.
이는 다양한 추론 패턴을 자연어 형태로 학습하여 모델의 전반적인 추론 능력이 향상된다 라는 내용입니다

# FINE_TURNING IN HAI
LLM을 학습시키는 방법은 지도학습/보강학습/보상학습 등의 방법론이 여러가지가 있습니다.<br/>
하지만 이번 파인튜닝에서 사용할 부분은 지도학습 입니다. 즉 기본 DataSet에 가져온 Data를 Push 하여 Prompt를 주는 상황에서의 답을 더 정확하게 만드는 행위 입니다.<br/>
이를 위해서 우리는 HuggingFace 라는 Ai 플랫폼을 활용 할 것이며 이곳에서 디스틸 된 모델을 다운받고 거기에 원하는 1차 데이터셋을 튜닝 + 딥시크의 부족한 언어 성능을 한국어 셋으로 튜닝<br/>
그 이후에 한국 의료정보와 기타 필요한 DataSet을 Turning 후에 이를 Quant 하여 최종 모델 축소까지를 목표로 하는 프로젝트입니다<br/>

1. 사용을 위한 한국어 데이터셋의 불러오기 과정 코드
   
        import os
        import pandas as pd
        
        dir_path = './kor_eng' # 엑셀 파일이 저장된 디렉터리
        files = os.listdir(dir_path)
        print(files)
        
        merge_df = pd.DataFrame()
        
        for file in files:
            df = pd.read_excel(f'{dir_path}/{file}')
            df = df[['원문', '번역문']]
            merge_df = pd.concat([merge_df, df])
        
        merge_df.columns = ['ko', 'en']
        merge_df.to_csv('./dataset.csv', index=False)

해당 코드를 사용하기 전에 우선 AI허브에서 원하는 번역 파일을 다운 받아야만 합니다. <br/><br/>
https://www.aihub.or.kr/aihubdata/data/view.do?currMenu=115&topMenu=100&aihubDataSe=data&dataSetSn=126 <br/>
