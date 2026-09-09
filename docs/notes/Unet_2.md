## 20. 지금까지 이해한 U-Net

앞에서 U-Net은 다음과 같이 이해했다.

```text
Input Image
    ↓
Encoder
    ↓
Downsampling
    ↓
Semantic Feature
    ↓
Decoder
    ↓
Upsampling
    ↓
Segmentation Mask
```

그리고 encoder의 high-resolution feature를 decoder에 skip connection으로 전달한다.

```text
Encoder Feature
128 × 128 × 64

Decoder Feature
128 × 128 × 128

        ↓ Concatenate

128 × 128 × 192
        ↓
      Conv
        ↓
더 정확한 prediction
```

여기서 중요한 것은,

```text
H × W
=
이미지의 spatial axis

C = feature channel
```

이었다.

즉 segmentation U-Net에서 encoder / decoder가 압축하고 복원하는 대상은

```text
2D Image Space
```

였다.

---

# 21. 그러면 Robot에서는?

원래 U-Net은 segmentation 모델로 나왔다.

```text
Input Image
    ↓
U-Net
    ↓
Predicted Mask

        vs

정답 Segmentation Mask
```

그런데 UMI에서는 우리가 원하는 output이 segmentation이 아니다.

우리가 원하는 것은 예를 들어,

```text
현재 카메라 이미지
현재 EE 상태
현재 gripper 상태
        ↓
Policy
        ↓
앞으로 EE가 어떻게 움직일 것인가
```

이다.

즉 output은 Future Action Chunk여야 한다.

그렇다면 의문은 다음과 같다.

> 이미지를 H×W로 줄였다 늘리는 U-Net이 어떻게 미래 action chunk를 출력할 수 있는가?

---

# 22. UMI

UMI는 policy learning에 Diffusion Policy를 사용한다.

그리고 공개된 UMI 코드에는 실제 training config로

```text
train_diffusion_unet_timm_umi_workspace
```

가 존재한다.

그 안에서 사용하는 diffusion network는

```text
ConditionalUnet1D
```

이다.

즉 우리가 segmentation에서 본 2D U-Net과 

완전히 같은 입력 형식을 사용하는 것이 아니라,

> U-Net의 encoder-decoder + skip connection 구조를  
> 1차원 시계열에 적용한 Temporal U-Net

이라고 이해된다.

---

# 23. U-Net in both Segmentation and UMI

Segmentation U-Net

```text
Image
H × W × C
```

UMI / Diffusion Policy U-Net

```text
T = Time / Action Horizon
D = 한 timestep의 Action Dimension

Action Sequence
T × D
```

이다.

즉 대응시키면,

```text
Segmentation U-Net

H, W = 이미지 spatial axis

Temporal U-Net

T = 시간 axis
```

가 된다.

---

# 24. UMI에서 Action 하나는 무엇인가?

UMI config에서는 action shape를

```text
shape: [10]
```

으로 정의한다.

개념적으로는,

```text
Position        3
Rotation 6D     6
Gripper Width   1
-----------------
Total          10
```

이다.

따라서 한 timestep의 action은 대략

```text
a_t = [x, y, z, r1, r2, r3, r4, r5, r6, gripper]
```

와 같은 10차원 vector가 된다.

UMI는 특히 robot 절대 좌표보다

```text
relative trajectory representation
```

을 사용하여 서로 다른 robot embodiment로 transfer하기 쉽게 만든다.

---

# 25. Action Chunk는 결국 2차원 배열이다

앞으로 T step을 예측한다고 해보자.

```text
a_t
a_t+1
a_t+2
...
a_t+T-1
```

각 action이 10차원이므로 전체 action chunk는

```text
T × 10
```

형태가 된다.

예를 들어 action horizon이 16이면,

```text
16 × 10
```

이다.

즉 이미지가 H × W라는 spatial domain을 가진다면, 

action chunk는 T라는 temporal domain을 가진다고 볼 수 있다.

그리고 이미지 입력에서 각 위치마다 RGB 3채널이 존재하듯, 

action sequence에서는 각 timestep마다 

position, rotation, gripper로 이루어진 10차원의 action 값이 존재한다.

이 값들은 이후 1D convolution을 거치면서 feature channel로 변환된다.

---

# 26. 1D Convolution

Segmentation U-Net에서는,

```text
3 × 3 Conv
```

가 이미지의

```text
x, y 주변 pixel
```

을 보면서 feature를 만든다.

Temporal U-Net에서는,

```text
1D Conv
```

가 action sequence의

```text
t-1, t, t+1
```

같은 주변 timestep을 보면서 feature를 만든다.

즉,

```text
2D CNN:

현재 pixel 주변의 spatial pattern을 본다
```

였다면,

```text
1D Temporal CNN:

현재 action 주변의 temporal pattern을 본다
```

가 된다.

---

# 27. Canny → CNN에서 이해했던 방식으로 다시 보면

앞에서 CNN을 다음처럼 이해했다.

```text
Conv = 주변 spatial pattern을 보고 feature를 추출 / 조합
```

Temporal U-Net에서는 이것이 그대로

```text
Temporal Conv = 주변 timestep들의 action pattern을 보고 temporal feature를 추출 / 조합
```

으로 바뀐 것이다.

예를 들어 action sequence가,

```text
접근
접근
감속
gripper close
위로 이동
```

이라면,

1D convolution은 시간축에서

```text
접근 → 감속
감속 → grasp
grasp → lift
```

와 같은 local temporal pattern을 학습할 수 있다.

---

# 28. 그러면 Encoder의 Downsampling은 무엇을 줄이는가?

Segmentation U-Net에서는,

```text
256 × 256
    ↓
128 × 128
    ↓
64 × 64
```

처럼 이미지 spatial resolution을 줄였다.

Temporal U-Net에서는,

```text
T
↓
T/2
↓
T/4
```

처럼 시간축 resolution을 줄인다.

예를 들어,

```text
16 timestep
    ↓
8 timestep
    ↓
4 timestep
```

처럼 볼 수 있다.

즉 segmentation에서

```text
가까운 pixel-level detail
→ 더 넓은 image context
```

를 보게 됐다면,

Temporal U-Net에서는

```text
짧은 순간의 action 변화
→ 더 긴 시간 범위의 action context
```

를 보게 된다.

---

# 29. Receptive Field

앞에서 CNN의 receptive field가 깊어질수록 커진다고 이해했다.

Segmentation에서는

```text
초기 layer
→ 주변 edge / texture

깊은 layer
→ 더 넓은 object / semantic context
```

Temporal U-Net에서는

```text
초기 layer
→ 가까운 timestep 사이의 motion 변화

깊은 layer
→ 더 긴 action sequence의 구조
```

예를 들어,

```text
초기 temporal feature
=
지금 EE가 왼쪽으로 움직이고 있다

더 깊은 temporal feature
=
접근 후 감속하고 grasp로 연결되는 motion pattern
```

처럼 볼 수 있다.

---

# 30. Decoder는 무엇을 Upsampling하는가?

Segmentation U-Net의 decoder는

```text
64 × 64
↓
128 × 128
↓
256 × 256
```

로 다시 pixel resolution을 복원했다.

Temporal U-Net에서는,

```text
4 timestep
↓
8 timestep
↓
16 timestep
```

처럼 다시 action time resolution을 복원한다.

결국 최종적으로는 처음과 같은 horizon을 가진

```text
T × Action Dimension
```

을 출력할 수 있어야 한다.

즉 decoder의 목적도 달라진다.

```text
Segmentation Decoder = pixel output 복원

Temporal Decoder = timestamp action output 복원
```

---

# 31. Skip Connection

UMI가 사용하는 Diffusion Policy의 `ConditionalUnet1D` 코드에서도

down path의 feature를 저장한 후,

```text
h.append(x)
```

decoder에서

```text
torch.cat((x, h.pop()), dim=1)
```

로 concatenate한다.

즉 앞에서 segmentation U-Net에서 이해했던

```text
Encoder Feature
        +
Decoder Feature
        ↓
Channel Concatenate
```

구조가 그대로 있다.

단지 이제 feature가

```text
Image Spatial Feature
```

가 아니라

```text
Temporal Action Feature
```

라는 점이 다르다.

> 그러면 Temporal U-Net에서 Skip Connection을 통해 보존되는 정보는?

Segmentation에서는 Encoder에서 깊은 layer로 갈수록

```text
Semantic Information ↑
Spatial Detail ↓
```

였기 때문에 skip connection이

```text
정확한 pixel 위치
edge
boundary
texture
```

등을 decoder에 다시 제공했다.

Temporal U-Net에서는 downsampling하면서

```text
긴 시간 범위의 action structure
```

는 얻지만,

```text
정확히 어느 timestep에서 방향이 바뀌는가?

언제 gripper가 닫히는가?

언제 속도가 줄어드는가?
```

timestamp 단위의 detail이 약해질 수 있다.

그래서 skip connection은 decoder에

```text
timestamp 단위 temporal feature
```

를 다시 제공한다.

---

# 33. 정리

이제 구조를 나란히 보면,

```text
Segmentation U-Net:

Image
H × W × C
    ↓
2D Conv
    ↓
Spatial Downsampling
    ↓
Bottleneck
    ↓
Spatial Upsampling
    ↓
Skip Connection
    ↓
H × W × Class
```

```text
Temporal U-Net

Action Chunk
T × D
    ↓
1D Temporal Conv
    ↓
Temporal Downsampling
    ↓
Bottleneck
    ↓
Temporal Upsampling
    ↓
Skip Connection
    ↓
T × D
```

가 된다.

즉,

> U-Net의 핵심은 segmentation이 아니라  
> multi-scale encoder-decoder + skip connection 구조였고,  
> 어느 axis에 convolution/downsampling을 적용하느냐에 따라  
> 이미지가 아니라 action sequence도 처리할 수 있다.

---

# 34. vision 정보는?

지금까지 살펴본 걸 다시 정리하자면

```text
Action Chunk
    ↓
U-Net
    ↓
Action Chunk
```

이다.

그런데 실제 UMI에서 넣고 싶은 것은

```text
Camera Image
+
Current Robot State
```

이다.

즉,

> 현재 무엇을 보고 있는지

가 action prediction에 들어가야 한다.

여기서 UMI / Diffusion Policy의 Conditioning이 등장한다.

---

# 35. Vision Encoder / U-Net in UMI

UMI에서 카메라 image 자체를 Temporal U-Net에 그대로 넣는 것이 아니다.

구조를 크게 나누면,

```text
Camera Image
    ↓
Vision Encoder
    ↓
Vision Feature
```

와,

```text
Noisy Action Chunk
    ↓
Conditional 1D U-Net
    ↓
Action Output
```

이 별도로 존재한다.

UMI 논문에서는 task에 따라

```text
ResNet-34
ViT-B/16
ViT-L/14
```

같은 visual encoder를 사용한다.

즉,

```text
Vision Encoder = 현재 scene을 feature로 표현

Temporal U-Net = 미래 action sequence를 생성 / 복원
```

으로 역할이 분리된다.

---

# 36. segmentation U-Net과 차이 

Segmentation U-Net은 보통,

```text
Image
↓
U-Net Encoder
↓
U-Net Decoder
↓
Mask
```

였다.

하지만 UMI Diffusion Policy에서는 더 정확하게,

```text
Image
↓
ResNet / ViT
↓
Vision Feature
                ┐
                │ condition
                ▼
Noisy Action → Temporal U-Net → Predicted Noise
```

이다.

즉 visual encoder와 temporal U-Net이 분리되어 있고, 

Vision Feature를 Temporal U-Net에 Condition으로 주는 방식이다.

---

# 37. Vision Feature는 U-Net에 어떻게 들어가는가?

Diffusion Policy의 U-Net에서는
Vision feature를 condition으로 사용한다.

개념적으로,

```text
Camera
↓
ViT / ResNet
↓
Visual Feature
        +
Current EE / Gripper State
        ↓
Vision Condition V_t
```

를 만든다.

그리고 이 condition을 Temporal U-Net의 여러 residual block에 전달한다.

여기서 대표적으로 사용하는 방식이 FiLM이다.

---

# 38. FiLM

FiLM은 현재 상황에 맞게 feature를 조절한다고 보면 된다.

Temporal U-Net 내부 feature를 F라고 하자.

Vision Feature로부터

```text
scale
bias
```

를 만들어,

```text
F' = scale × F + bias
```

처럼 U-Net feature를 조절한다.

이를 직관적으로 풀어보자면,

```text
똑같은 action sequence라도

현재 컵이 왼쪽에 있으면
→ 왼쪽으로 접근하는 action 쪽 feature를 강화

현재 컵이 오른쪽에 있으면
→ 오른쪽으로 접근하는 action 쪽 feature를 강화
```

하도록 condition을 주는 것이다.

즉,

> Vision feature가 action 자체가 되는 것이 아니라,  
> action denoising 과정이 현재 Vision Feature에 맞는 방향으로 가도록 조건을 건다.

---

# 39. Label

Segmentation에서는 dataset에

```text
Image
+
Segmentation Mask
```

가 있었다.

```text
Image
↓
U-Net
↓
Predicted Mask

vs

Ground Truth Mask
```

였다.

UMI에서는 demonstration을 통해

```text
Observation + Future Action Chunk
```

를 만든다.

예를 들면,

```text
현재 Camera Image
현재 EE Pose
현재 Gripper
        +
앞으로 사람이 실제 수행한
Future EE Trajectory
Future Gripper State
```

이다.

즉 supervised data pair 자체는,

```text
X = Observation

Y = Future Action Chunk
```

로 볼 수 있다.

---

# 40. Diffusion에서는 Future Action을 바로 맞히지 않는다

여기서 일반 regression policy와 Diffusion Policy가 갈린다.

단순 policy라면,

```text
Observation
    ↓
Network
    ↓
Predicted Action Chunk

        vs

Ground Truth Action Chunk
```

처럼 직접 비교할 수 있다.

하지만 UMI가 사용하는 Diffusion Policy에서는
future action chunk를 그대로 output target으로 바로 예측하지 않는다.

> Demonstration Action Chunk를 x0라고 하자

전처리된 demonstration으로부터,

```text
x0 = Future Action Chunk
```

를 얻는다.

예를 들어,

```text
x0

t0 : ΔEE + grip
t1 : ΔEE + grip
t2 : ΔEE + grip
...
tT : ΔEE + grip
```

이다.

이게 현재 task에서 우리가 학습하고 싶은 실제 action distribution의 data sample이다.

---

# 42. 여기에 Noise를 넣는다

Diffusion training에서는 action chunk에 noise를 추가한다.

```text
Action Chunk x0
        +
Noise ε
        ↓
Noisy Action Chunk x_k
```

즉 Temporal U-Net이 실제로 입력으로 받는 것은

```text
완성된 미래 action
```

이 아니라

```text
Noise가 섞인 미래 action sequence
```

이다.

---

# 43. Denoising

구조는 이제 이렇게 된다.

```text
Current Observation
Camera + EE + Gripper
        ↓
Vision / State Encoder
        ↓
Condition O_t
        │
        │
        ▼
Noisy Future Action x_k
        ↓
Conditional 1D U-Net
        ↓
Predicted Noise ε_hat
```

그리고 training에서는,

```text
Predicted Noise ε_hat
        vs
실제로 넣었던 Noise ε
        ↓
Loss
```

를 계산한다.

---

# 44. Segmentation Label과 차이

Segmentation에서는,

```text
Network Output = Pixel Prediction

Training Target = Segmentation Mask
```

였다.

Diffusion Policy에서는,

```text
Network Output = Action Chunk에 들어간 Noise Prediction

Training Target = 실제로 추가했던 Noise ε
```

이다.

즉,

> Conditional U-Net이 직접 맞히는 label은 future action 자체가 아니라 noise다.

---

# 45. 수집한 Action Chunk는 정답이 아닌가?

여기서 두 층으로 나눠서 생각해야 한다.

### Task Ground Truth

```text
Human Demonstration
↓
Preprocessing
↓
Future Action Chunk x0
```

이것은

> 이 observation에서 사람이 실제로 수행한 미래 행동

이라는 supervision이다.

---

### Diffusion Training Target

그 `x0`에 random noise를 넣고,

```text
ε
```

를 만들어 network가 그 noise를 맞히게 한다.

즉 한 training iteration의 직접적인 loss target은

```text
ε
```

이다.

하지만 그 noise prediction problem 자체가

```text
실제 demonstration action x0
```

를 바탕으로 만들어진다.

---

# 46. 전체 흐름

```text
[Data Collection]

Human UMI Demonstration
        ↓
RGB / Pose / Gripper
        ↓
SLAM + Coordinate Processing
        ↓
Relative EE Trajectory
        ↓
Future Action Chunk x0
```

```text
[Training]

x0 + Random Noise ε
        ↓
       x_k
        │
        │
Camera / Current State
        ↓
Vision Encoder
        ↓
    Condition
        │
        ▼
Temporal 1D U-Net
        ↓
Predicted Noise ε_hat
        ↓
     ε와 비교
        ↓
  Backpropagation
```

---

# 47. Inference

학습할 때는,

```text
Real Action
→ Noise를 추가
→ Noise를 맞히도록 학습
```

Inference에서는 real future action을 모른다.

그래서 처음에는 그냥

```text
Random Noise
```

에서 시작한다.

```text
Random Action Noise
        ↓
Current Observation으로 Condition
        ↓
1D U-Net Denoising
        ↓
조금 더 action 같은 sequence
        ↓
다시 Denoising
        ↓
...
        ↓
Future Action Chunk
```

가 된다.

즉,

> U-Net이 segmentation mask를 한 번에 복원하는 것이 아니라,  
> noise 형태의 action sequence를 여러 단계에 걸쳐  
> 현재 observation에 맞는 future action sequence로 복원한다.

---

# 48. Why Predict Action Chunk?

만약 매 순간 action 하나만 예측하면,

```text
현재 frame
↓
a_t

다음 frame
↓
a_t+1

다음 frame
↓
a_t+2
```

처럼 된다.

그러면 각 action 사이의 temporal consistency를 직접 모델링하기 어렵다.

Action chunk를 한꺼번에 생성하면,

```text
[a_t, a_t+1, ..., a_t+T]
```

전체의 관계를 함께 볼 수 있다.

Temporal U-Net의 convolution도 이 time axis를 따라 동작하므로,

```text
접근
↓
감속
↓
grasp
↓
lift
```

같은 연속적인 motion structure를 하나의 sequence로 모델링할 수 있다.

---

# 49. U-Net Bottleneck

Segmentation U-Net의 bottleneck은

```text
넓은 image context + high-level semantic representation
```

을 가진다고 이해했다.

Temporal U-Net에서는,

```text
넓은 temporal receptive field + action chunk 전체의 motion structure
```

를 가진다고 보면 된다.

예를 들어,

```text
"초반에는 접근하고 중간에는 잡고 후반에는 들어올린다"
```

와 같은 전체적인 trajectory 구조를 나타낼 수 있다.

---

# 50. Action Timing

Bottleneck에서 action structure를 얻었다면,

decoder는 다시 temporal resolution을 높인다.

그리고 skip connection으로 encoder에서 저장한 high-resolution temporal feature를 받는다.

* 관련 논문들에서 자꾸 Coarse, Fine 하는게 이제 완전히 이해가 되었다.
> Receptive Field를 최대로 넓힌 bottleneck 에서는 국소적인 정확한 특징보다 전체적인 조잡한 패턴을,
> 각 Encoder 단계에서는 비교적으로 더 국소적인 정확한 특징 -> 이걸 Fine 이라고 표현

```text
Coarse Motion Plan
        +
Fine Temporal Feature
        ↓
     Decoder
        ↓
정확한 timestep별 action
```

개념적으로,

```text
Bottleneck: 이 chunk는 접근 → grasp → lift

Skip Feature:  
- Gripper close는 이 timestep 근처
- 여기서 velocity가 감소한다

        ↓

Decoder:
각 timestep의 구체적인 action sequence
```

처럼 볼 수 있다.

---

# 51. 전체 architecture

```text
                    ┌──────────────────────┐
Camera Image ──────►│ Vision Encoder       │
                    │ ResNet / ViT         │
                    └──────────┬───────────┘
                               │
Current EE / Gripper ──────────┤
                               │
                         Observation
                          Condition
                               │
                     FiLM / Conditioning
                               │
                               ▼

Random / Noisy Future Action Chunk
[T × Action Dimension]
               │
               ▼
        ┌───────────────┐
        │ Temporal      │
        │ U-Net Encoder │
        └──────┬────────┘
               │
          Downsample
               │
               ▼
          Bottleneck
               │
               ▼
        ┌───────────────┐
        │ Temporal      │◄──── Skip Features
        │ U-Net Decoder │
        └──────┬────────┘
               │
               ▼
      Predicted Noise Sequence
               │
               ▼
        Diffusion Denoising
               │
          반복 수행
               │
               ▼
       Future Action Chunk
```

---

# 52. 수집기

UMI 수집기는 단순히

```text
영상 데이터 수집 장치
```

가 아니다.

학습에 필요한 두 종류의 정보를 동시에 만든다.

### Observation

```text
RGB
Current EE / Gripper State
```

### Supervision

```text
Future EE Trajectory
Future Gripper Action
```

즉 demonstration 하나에서 시간축을 shift하면,

```text
현재 observation
        ↓
앞으로 실제 사람이 수행한 action
```

이라는 training pair를 계속 만들 수 있다.

---

# 53. Canny → CNN → U-Net → UMI

Canny에서는,

```text
Image
↓
사람이 설계한 Kernel
↓
Edge Feature
↓
사람이 설계한 비선형 처리
↓
Edge Map
```

이었다.

CNN에서는,

```text
Image
↓
학습된 Conv
↓
Feature Hierarchy
↓
Loss가 유용한 feature를 결정
```

로 바뀌었다.

Segmentation U-Net에서는,

```text
Image
↓
Multi-scale CNN Encoder
↓
Semantic Context
+
Skip Spatial Detail
↓
Decoder
↓
Pixel Mask
```

가 되었다.

UMI의 Diffusion Policy에서는 U-Net 구조를 시간축으로 가져와,

```text
Noisy Action Sequence
↓
Multi-scale Temporal Encoder
↓
Long-range Action Context
+
Skip Temporal Detail
↓
Decoder
↓
Action Noise Prediction
```

을 수행한다.

그리고 별도의 vision encoder가

```text
"현재 상황에서 어떤 action sequence가 맞는가?"
```

를 condition으로 제공한다.