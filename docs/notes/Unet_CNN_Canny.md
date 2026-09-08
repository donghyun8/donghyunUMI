# Understanding CNN with Canny detection 

Canny Edge Detector를 구현할 때는 이미지에서 어떤 특징을 뽑을지 사람이 직접 정한다.

대표적인 예가 Sobel filter다.

```text
Gx Kernel              Gy Kernel

-1  0  1               -1 -2 -1
-2  0  2                0  0  0
-1  0  1                1  2  1

Vertical 방향 변화     Horizontal 방향 변화
```

원본 이미지에 서로 다른 kernel을 적용하면,

```text
Input Image
   │
   ├─ Sobel Gx → Vertical Edge Feature
   └─ Sobel Gy → Horizontal Edge Feature
```

즉, Canny를 구현하면서 이미 하나의 이미지에서 여러 kernel을 사용해 서로 다른 특징을 추출한다는 구조를 사용하고 있었다.

---

## 2. CNN도 기본 구조는 비슷하다

CNN 역시 하나의 convolution kernel만 사용하는 것이 아니다.

여러 개의 학습 가능한 kernel을 사용해서 여러 종류의 feature map을 만든다.

```text
Input Image
   │
   ├─ Kernel 1 → Feature Map 1
   ├─ Kernel 2 → Feature Map 2
   ├─ Kernel 3 → Feature Map 3
   └─ ...
```

초기 layer에서는 결과적으로 edge, direction, color contrast, texture, curve 같은 비교적 단순한 local pattern에 반응하는 kernel이 학습될 수 있다.

차이는 다음과 같다.

```text
Canny
→ 사람이 kernel을 직접 설계

CNN
→ 학습 과정에서 kernel weight를 데이터로부터 학습
```

---

## 3. Feature Map

원본 이미지는 당연히 2차원 데이터이다.

```text
(x, y)
```

RGB 이미지라면 각 위치에는

```text
(x, y)
    ↓
[R, G, B]
```

라는 3개의 값이 있다.

CNN의 첫 convolution 이후 channel이 64개가 되었다면

```text
(x, y)
    ↓
[f1, f2, f3, ... f64]
```

처럼 각 위치가 64차원의 feature vector로 표현된다.

즉,

```text
Spatial Dimension = x, y
Feature Dimension = channel
```

을 구분해야 한다.

CNN이 깊어지면서 표현 차원이 높아진다는 것은 공간 자체가 고차원이 된다는 뜻이 아니라,
같은 spatial position을 더 많은 feature channel로 설명한다는 의미에 가깝다.

---

## 4. 다음 Conv는 이전 Feature들을 다시 조합한다

Canny에서는 `Gx`, `Gy`를 구한 뒤 사람이 직접 다음 단계를 설계한다.

```text
Gradient Magnitude
NMS
Threshold
Hysteresis
```

CNN에서는 다음 convolution layer가 이전 layer에서 생성된 여러 feature channel을 다시 입력으로 사용한다.

예를 들어 이전 layer에

```text
F1 = vertical edge feature
F2 = horizontal edge feature
F3 = texture feature
```

가 있다면 다음 layer의 output feature 하나는 개념적으로:

```text
F1의 주변 pattern + F2의 주변 pattern + F3의 주변 pattern
```

을 학습된 weight로 조합해 만든다.

수식으로는

```text
Y_k = Σ_c (W_k,c * X_c) + b_k
```

- `c`: input channel
- `k`: output channel
- `W_k,c`: input channel `c`를 output channel `k`로 조합할 때 사용하는 kernel

> Conv 자체는 이전 feature들을 학습된 방식으로 선형 결합하는 연산이라고 볼 수 있다.

---

## 5. 하나의 Output Channel을 만들 때 실제로 무슨 일이 일어나는가?

입력 feature map channel이 3개이고 kernel size가 3 × 3이라면,
새로운 output channel 하나를 만들 때:

```text
Channel 1용 3×3 kernel
        +
Channel 2용 3×3 kernel
        +
Channel 3용 3×3 kernel
        ↓
     모두 합산
        ↓
Output Feature Map 1개
```

가 된다.

따라서 단순히 feature map 전체에 숫자 하나씩 곱하는 것이 아니라,
각 channel의 주변 spatial pattern까지 함께 보면서 새로운 feature를 만든다.

---

## 6. 왜 Layer가 깊어질수록 더 복잡한 Feature를 만들 수 있는가?

첫 layer에서,

```text
edge
curve
texture
```

같은 특징을 만들었다고 하자.

다음 layer는 이 feature들을 다시 조합해서

```text
edge + edge
-> corner pattern

curve + texture
-> 특정 surface pattern
```

같은 더 복잡한 feature를 만들 수 있다.

그 다음 layer에서는 다시

```text
corner + curve + texture
```

를 조합한다.

전체적으로 보면

```text
Pixel
  ↓
Edge / Texture
  ↓
Corner / Curve
  ↓
Shape / Object Part
  ↓
Object / Semantic Feature
```

처럼 feature hierarchy가 만들어진다.

---

## 7. Activation 함수가 필요한 이유

Conv만 계속 쌓고 activation이 없다면

```text
Conv
↓
Conv
↓
Conv
```

는 결국 하나의 큰 선형 연산으로 합쳐질 수 있다.

```text
y = W3(W2(W1x))
  = Wx
```

따라서 layer를 많이 쌓아도 본질적으로는 하나의 선형 변환과 크게 다르지 않다.
-> 즉, 결국 표현력이 하나의 선형 표현에 국한된다는 의미

---

## 8. Activation의 비선형성은 특정 Feature 조합에 선택적으로 반응하게 한다

그래서 CNN은 Conv 사이에 ReLU 같은 activation을 넣는다.

```text
Conv
↓
ReLU
↓
Conv
↓
ReLU
```

ReLU는

```text
ReLU(x) = max(0, x)
```

이므로, 이를 직관적으로 보면

```text
특정 feature 조합이 충분히 강하면
-> 활성화

조건을 만족하지 못하면
-> 0
```

처럼 동작할 수 있다.

예를 들어서,

```text
Vertical Edge + Horizontal Edge + Bias
                  ↓
                ReLU
```

를 통해 특정 조합일 때만 `Corner Feature`가 활성화되는 식의 표현을 학습할 수 있다.

즉,

```text
선형 결합
= feature들을 섞는다

비선형성
= 그 조합에 선택적으로 반응할 수 있게 한다
```

라고도 이해할 수 있다.

---

## 9. Canny

Canny에서도 Sobel 결과를 단순히 더하는 것으로 끝나지 않았었다.

```text
Sobel Gx
Sobel Gy
   ↓
Gradient Magnitude
   ↓
NMS
   ↓
Threshold
   ↓
Hysteresis
```

를 거친다.

이 과정을 다시 뜯어보면, 

Sobel에서는 Kernel을 이용하여 `Gx`, `Gy`를 구하는 선형 필터 역할을 수행한다. 

하지만, `Gx` `Gy`만으로는 `실제 의미 있는 edge인가?`를 충분히 판단하기 어렵기 때문에 이후 

Gradient Magnitude -> 이 위치에서 밝기 변화가 얼마나 강한가?
Gradient Direction -> 어느 방향으로 밝기 변화가 가장 큰가?
NMS -> Edge를 가장 잘 표현하는 peak만 남겨 edge를 얇게 만든다
**Threshold* -> 임계값 이하 = Weak Edge, 이상은 Strong Edge로 변경** 
**Hystersis -> Weak Edge 주변이 Strong Edge와 연결되어 있다면 Strong Edge로, 아니라면 0으로 변경**
> 구현하면서는 당연하다는 느낌으로 구현했던 것 같은데, 
> 다시 생각하면 CNN의 ReLU와 매우 유사한 작업을 수행하는 `비선형 연산`을 수행한다
> 이런 전통적인 CV나 머신러닝을 비선형성에 대한 관점으로 바라봐본 적이 없었던 것 같다.   

CNN에서도 비슷한 관점으로 볼 수 있다.

```text
Conv
= 여러 feature response 생성 / 조합

Activation
= 특정 feature combination을 선택적으로 활성화

다음 Conv
= 활성화된 feature들을 다시 조합
```

차이는 Canny에서는 사람이 조합 방법을 직접 설계하고,
CNN에서는 어떤 feature를 뽑고 어떻게 조합할지를 loss를 통해 학습한다는 것이다.

---

## 10. Receptive Field, Feature

3 × 3 kernel, stride 1 convolution을 계속 쌓으면 receptive field는 원본 기준으로

```text
Layer 1 → 3 × 3
Layer 2 → 5 × 5
Layer 3 → 7 × 7
```

처럼 커진다.

즉 깊은 layer로 갈수록

```text
더 많은 종류의 feature를 조합
+
더 넓은 spatial context를 관찰
```

하게 된다.

이 두 가지가 함께 일어나면서 더 high-level의 semantic feature를 학습하기 쉬워진다.

---

## 11. U-Net Skip Connection

U-Net encoder에서는 convolution과 downsampling을 반복한다.

```text
256 × 256
    ↓
128 × 128
    ↓
64 × 64
    ↓
32 × 32
```

깊은 layer로 갈수록, 

```text
Receptive Field ↑
Feature Complexity ↑
Semantic Information ↑
```

가 되는 대신, 

```text
Spatial Resolution ↓
Fine Detail ↓
```

가 된다.

즉 깊은 feature는

```text
"이 영역은 세포다"
```

라는 의미 판단에는 강하지만

```text
"세포 경계가 정확히 어느 pixel인가"
```

라는 정보는 상대적으로 약하다.

---

## 13. Decoder가 Upsampling한다고 잃어버린 Detail이 자동 복원되지는 않는다

Encoder 깊은 레이어에서의 feature가

```text
32 × 32
```

까지 줄었다가 decoder에서

```text
32 × 32
   ↓
64 × 64
   ↓
128 × 128
```

로 upsampling돼도, 단순히 크기를 키운다고 원래 정보가 돌아오는 것은 아니다.

그래서 decoder에는 encoder가 가지고 있던 고해상도 local information이 필요하다.

---

## 15. Spatial Position

Encoder와 decoder의 같은 scale feature는 같은 원본 이미지 좌표계를 기반으로 한다.

예를 들어,

```text
Encoder Feature
128 × 128 × 64

Decoder Feature
128 × 128 × 128
```

이라면 같은 `(x, y)` 위치끼리 대응시킬 수 있다.

```text
Encoder (x, y)
+
Decoder (x, y)
```

즉 skip connection이 가능한 핵심은 같은 scale에서 position이 유지되기 때문이다.

---

## 16. Skip Connection은 실제로 무엇을 하는가?

U-Net에서는 보통 encoder feature와 decoder feature를
channel 방향으로 concatenate한다.

```text
H x W x C
Height x Width x Channel 

Decoder Feature
128 × 128 × 128

Encoder Skip Feature
128 × 128 × 64

        ↓ Concatenate

128 × 128 × 192
```

특정 위치 `(x, y)`만 보면,

```text
Decoder:
[d1, d2, ..., d128]

Encoder:
[e1, e2, ..., e64]

        ↓

[d1, ..., d128, e1, ..., e64]
```

라는 192차원 feature vector가 만들어진다.

---

## 17. Concatenate

Skip connection 자체가 semantic 정보와 spatial 정보를 자동으로 융합해주는 것은 아니다.

Concatenate는 그냥

```text
128 channels + 64 channels = 192 channels
```

로 정보를 이어 붙이는 연산일 뿐이다.

그 자체로 세포와 같은 의미가 생기는 것은 아니다.

---

## 18. Skip Feature를 실제로 사용하는 것은 그 뒤의 Conv다

Concatenate 이후 다시 convolution을 적용한다.

```text
Decoder Feature
      +
Encoder Skip Feature
      ↓
Concatenate
      ↓
Conv
      ↓
ReLU
      ↓
Conv
      ↓
ReLU
```

이 Conv가 앞에서 이해한 것과 동일하게 192개의 channel을 학습된 weight로 조합한다.

개념적으로,

```text
Decoder Channel
"이 영역은 세포일 가능성이 높음"

+

Encoder Channel
"이 위치에 강한 edge가 있음"

+

Encoder Channel
"여기서 texture가 바뀜"

        ↓ Conv + ReLU

"이 위치의 edge가 세포의 실제 경계일 가능성이 높음"
```

같은 새로운 feature를 만들 수 있다.

---

## 19. 정리

Skip feature를 붙인다고 자동으로 유용해지는 것이 아니다.

U-Net 전체는 segmentation loss를 최소화하도록 end-to-end로 학습된다.

```text
Prediction
vs
Ground Truth Mask
        ↓
Segmentation Loss
        ↓
Backpropagation
```

이 과정에서 concatenate 이후 Conv는

```text
어떤 decoder channel이 semantic 정보에 유용한가?

어떤 skip channel이 boundary 정보에 유용한가?

어떤 channel 조합이 세포와 배경을 구분하는 데 도움이 되는가?
```

를 weight에 반영한다.

따라서

```text
Skip Connection
= 정보를 제공

Conv
= 그 정보를 어떻게 조합할지 학습

Segmentation Loss
= 어떤 조합이 좋은지 방향을 제시
```

라고 이해할 수 있다.

---