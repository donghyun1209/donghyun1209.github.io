---
title: "논문 노트 · Designing Extremely Memory-Efficient CNNs for On-device Vision Tasks (ACCV 2024)"
date: 2026-11-07 09:00:00 +0900
categories: [Edge AI, 논문 리뷰]
tags: [mobilenetv2, peak memory, mcu, stm32, input segmentation, patch tunneling, bottleneck reordering, int8]

---

## 63KB로 ImageNet을 돌린다??!!

한 줄로 요약하면, MobileNetV2 bottleneck 기반 CNN을 int8 기준 **피크 메모리 63KB**로 돌려서 ImageNet top-1 61.58%, PASCAL-VOC mAP 50.7%(65KB)를 냈고, RAM 256KB급 MCU인 STM32H7에서 실제로 실행했다는 논문이다. 같은 주에 YOLOv8n을 INT8로 줄여 봤자 3.4MB였던 걸 생각하면 자릿수부터 달랐다.\\
In one line: the paper runs a MobileNetV2-bottleneck-based CNN at a **peak memory of 63KB** (int8), reaching 61.58% ImageNet top-1 and 50.7% PASCAL-VOC mAP (65KB), and actually runs it on an STM32H7, an MCU with about 256KB of RAM. Considering that YOLOv8n was still 3.4MB after I shrank it to INT8 the same week, it was a different order of magnitude altogether.

먼저 짚어야 했던 건 피크 메모리라는 개념이다. 추론 중 한 순간에 들고 있는 양의 최댓값을 말한다. conv는 이웃 출력 픽셀들이 입력의 3×3 영역을 겹쳐 읽기 때문에 출력을 다 만들기 전엔 입력을 지울 수 없다. 그래서 레이어마다 `입력 + 출력`이 필요하고, 레이어가 끝나면 입력은 버리고 출력이 다음 입력이 된다. 이 중 가장 큰 레이어가 피크다. 원래 MobileNetV2는 3번째 레이어에서 5.6MB가 필요해서 256KB MCU에는 들어가지도 않는다.\\
The first thing to pin down was peak memory: the maximum amount held at any single moment during inference. In a conv, neighboring output pixels read overlapping 3×3 input regions, so the input can't be freed until the whole output is produced. Each layer therefore needs `input + output`; when a layer finishes, its input is dropped and its output becomes the next input. The largest of these is the peak. Plain MobileNetV2 needs 5.6MB at its third layer, so it doesn't even fit in a 256KB MCU.

논문의 설계 원칙은 세 개인데, 전부 계산량이 아니라 "한꺼번에 들고 있는 양"을 줄이는 쪽이다.\\
The paper has three design principles, and all of them reduce "how much is held at once," not the amount of computation.

---

## 이미지를 자르면 메모리가 1/4이 될 것으로 예상 했다

첫 번째 원칙인 **input segmentation**은 224×224 이미지를 4조각으로 나누고 가운데 조각을 하나 더 둔다. 조각끼리는 여백 18픽셀씩 겹치게 자른다. 4조각이니 단순하게 메모리도 1/4이겠거니 했는데, 논문의 식 `1/k + 2m/(h√k) + m²/h²`에 h=224, k=4, m=18을 넣어 보니 0.25 + 0.087 = **0.337**이 나왔다. 조각 한 변이 112+18 = 130이니 130² / 224²과 같은 값이다.\\
The first principle, **input segmentation**, splits a 224×224 image into four patches and adds one more center patch, with an 18-pixel overlap margin between them. With four patches I naively assumed memory would drop to 1/4, but plugging h=224, k=4, m=18 into the paper's formula `1/k + 2m/(h√k) + m²/h²` gave 0.25 + 0.087 = **0.337**. A patch side is 112+18 = 130, so this equals 130² / 224².

처음엔 0.087을 손실이라고 생각했는데, 다시 보니 성능 손실이 아니라 여백 때문에 일부러 더 내는 메모리 비용이었다. 실제로 여백이 0이면 61.31%, 18이면 63.84%였고(Table 6), 가운데 조각을 빼면 63.84%에서 62.32%로 떨어졌다(Table 4). 가운데 조각을 추가해도 조각을 하나씩 처리하고 크기도 같으니 피크 메모리는 그대로다. 대신 계산량은 조각이 4개에서 5개가 되면서 0.337 × 5 ≈ 원본의 1.68배가 된다.\\
At first I read the 0.087 as a loss, but on a second look it's not a performance loss. With a margin of 0 accuracy is 61.31%, with 18 it's 63.84% (Table 6), and removing the center patch drops it from 63.84% to 62.32% (Table 4). Adding the center patch doesn't change peak memory, since patches are processed one at a time and are the same size. Computation, though, goes from 4 to 5 patches: 0.337 × 5 ≈ 1.68× the original.

그럼 k를 늘리면 더 줄지 않을까 싶어서 k=16으로도 계산해 봤다. 조각 한 변이 56+18 = 74라 비율은 0.109로 줄지만, 이상적인 1/k와 비교하면 k=4일 때 1.35배였던 게 1.75배로 벌어진다. 여백이 차지하는 비중이 커져서 줄어드는 폭이 점점 작아지는 것이다. 조각이 작을수록 한 조각이 보는 정보가 줄어 정확도도 떨어지고, 그걸 막으려고 여백을 키우면 메모리 이득을 다시 깎아 먹는다. 공짜는 아니었다.\\
I wondered whether a larger k would shrink it further, so I worked out k=16 as well. A patch side becomes 56+18 = 74 and the ratio drops to 0.109, but compared with the ideal 1/k, the overhead grows from 1.35× at k=4 to 1.75×. The margin takes up a larger share, so each step saves less. Smaller patches also see less information, lowering accuracy, and enlarging the margin to compensate eats back into the memory savings. It's not free.

---

## 합치기를 끝까지 미뤄야 자른 의미가 있다

이미지를 잘라 놓고 레이어 하나 지나서 바로 다시 이어 붙이면 큰 feature map이 그대로 부활한다. 그래서 두 번째 원칙인 **patch tunneling**은 조각마다 독립된 터널(bottleneck 여러 개)로 끝까지 돌려서 작은 벡터 하나만 남기고, 중간 feature map은 버린다. 터널1을 끝내고 v1만 남긴 뒤 터널2를 돌리는 식으로 다섯 조각을 처리하고, 마지막에 벡터 다섯 개를 더해서 FC로 분류한다. 어느 순간에도 조각 크기의 feature map만 메모리에 있다.\\
If you split the image and then stitch it back right after one layer, the large feature map comes right back. So the second principle, **patch tunneling**, runs each patch through its own independent tunnel (several bottlenecks) all the way to the end, keeping only a small vector and discarding the intermediate feature maps. It finishes tunnel 1 and keeps just v1, then runs tunnel 2, and so on for all five patches; at the end the five vectors are summed and classified by an FC layer. At no point is anything larger than a patch-sized feature map in memory.

합칠 때 concat이 아니라 더하기를 쓰는 이유도 숫자로 보니 바로 이해됐다. 길이 3짜리 벡터 다섯 개를 concat하면 길이 15가 되지만, 같은 칸끼리 더하면 길이 3 그대로다. 길이가 그대로면 메모리도, 다음 FC 레이어의 가중치 수(입력 길이 × 클래스 수)도 그대로다. 대신 concat은 "몇 번째 조각에서 온 값인지"가 자리로 남는데, 더하기는 그 정보가 섞여서 사라진다.\\
Seeing it in numbers also made it clear why the merge is a sum rather than a concat. Concatenating five length-3 vectors gives length 15, while summing them element-wise keeps length 3. Keeping the length means memory and the next FC layer's weight count (input length × number of classes) stay the same. The catch is that concat preserves "which patch did this value come from" by position, while summing mixes that information away.

이게 정확도의 대가로 이어진다. 깊은 레이어일수록 한 픽셀이 보는 원본 범위(receptive field)가 넓어져야 "눈 + 눈 + 코 = 얼굴" 같은 걸 만들어 내는데, 터널은 아무리 깊어져도 자기 조각 밖을 못 본다. 경계에 걸친 물체는 맨 끝 더하기에서야 처음 만나고, 그 더하기는 위치 관계까지 섞어 버린다. 여백과 가운데 조각이 바로 이걸 보완하려는 장치였다. RasEyes로 치면 640×640 대신 320×320 조각 4개를 작은 YOLO 4개로 따로 돌리고 끝에서 결과만 합치는 셈인데, 화면 가운데를 가로지르는 사람은 각 YOLO가 반쪽씩만 보게 된다.\\
This is where the accuracy cost comes from. Deeper layers need a wider receptive field to compose things like "eye + eye + nose = face," but a tunnel never sees outside its own patch however deep it goes. An object straddling a border only meets itself in the final sum, and that sum mixes away the spatial relationship too. The margin and the center patch are exactly the devices meant to compensate. In RasEyes terms, it's like running four small YOLOs on four 320×320 patches instead of one 640×640 image and merging only the results at the end — a person walking across the middle of the frame would be seen only in halves by each YOLO.

---

## 채널을 하나씩 흘려도 결과가 똑같았다

세 번째 원칙인 **bottleneck reordering**은 처음 읽었을 때 가장 헷갈렸다. MobileNetV2 bottleneck은 1×1 expand로 채널을 불렸다가, 3×3 depthwise를 거쳐, 1×1 reduce로 다시 줄인다. reordering은 이걸 채널 하나씩 expand → depthwise → reduce까지 끝까지 흘리고, 출력에 자기 몫을 더한 뒤 중간값을 버리는 걸 반복한다.\\
The third principle, **bottleneck reordering**, confused me the most on first read. A MobileNetV2 bottleneck widens channels with a 1×1 expand, passes through a 3×3 depthwise, and narrows them again with a 1×1 reduce. Reordering pushes one channel at a time all the way through expand → depthwise → reduce, adds its share to the output, discards the intermediate values, and repeats.

감을 잡으려고 픽셀 1개, 채널 3개짜리로 계산해 봤다. 입력 x=2, expand 가중치 (1,2,3), depthwise는 ×1, reduce 가중치 (1,2,1)이라고 하면, 원래 방식은 중간값 (2,4,6)을 전부 들고 있다가 1×2 + 2×4 + 1×6 = 16을 계산한다. reordering은 채널1에서 2를 만들어 1×2를 더하고 버리고, 채널2에서 4를 만들어 2×4를 더하고 버리고, 채널3에서 6을 만들어 1×6을 더한다. 결과는 똑같이 16인데, 중간값은 항상 하나만 들고 있다. 채널이 200개면 원래 방식은 200개를 들고 있어야 하지만 reordering은 여전히 1개다.\\
To get a feel for it, I worked a toy case with one pixel and three channels. With input x=2, expand weights (1,2,3), depthwise ×1, and reduce weights (1,2,1), the original way holds all intermediate values (2,4,6) and computes 1×2 + 2×4 + 1×6 = 16. Reordering makes 2 for channel 1, adds 1×2, and drops it; makes 4 for channel 2, adds 2×4, and drops it; makes 6 for channel 3 and adds 1×6. The result is the same 16, but only one intermediate value is ever held. With 200 channels, the original way holds 200 values; reordering still holds 1.

이게 가능한 이유는 세 가지였다. expand의 채널 j는 입력 x만 있으면 만들 수 있고, depthwise는 정의상 채널별로 독립이며(일반 3×3 conv였다면 불가능하다), reduce는 결국 더하기라 나눠서 더해도 같다. 실제 크기로 보면 차이가 더 크다.\\
It works for three reasons: expand channel j needs only the input x, depthwise is per-channel by definition (a regular 3×3 conv would make this impossible), and reduce is ultimately a sum, so adding in pieces gives the same result. With real sizes the difference is even bigger.

| 항 (Term) | 원래 (Original) | reordering |
|---|---|---|
| X (입력, `56×56×24`) | 75,264 | 75,264 |
| Yp (expand 출력) | 56×56×**144** = 451,584 | 56×56×**1** = 3,136 |
| Yd (depthwise 출력) | 56×56×**144** = 451,584 | 56×56×**1** = 3,136 |
| Yo (출력, `56×56×24`) | 75,264 | 75,264 |
| 합 (Total) | 1,053,696 | 156,800 (≈ 1/6.7) |

MobileNetV2 중간 블록 하나(입력 `56×56×24`, expansion ratio 6)에서 가운데 두 항이 144채널에서 1채널로 줄면서 블록 전체가 약 1/6.7이 됐다. Table 3에서 expansion ratio를 2에서 8로 키워도 top-1이 58.82%에서 63.84%로 오르는데 메모리는 254KB 그대로인 것도 같은 이유다. 채널이 아무리 늘어도 피크는 1채널 분량이다. 대신 반복 횟수와 가중치 수는 늘어난다.\\
For one middle block of MobileNetV2 (input `56×56×24`, expansion ratio 6), the two middle terms shrink from 144 channels to 1, cutting the whole block to about 1/6.7. That's also why, in Table 3, raising the expansion ratio from 2 to 8 lifts top-1 from 58.82% to 63.84% while memory stays at 254KB: however many channels there are, the peak is one channel's worth. The number of iterations and weights does grow, though.

---

## 그럼 추론 시간은?

읽는 내내 걸렸던 건 속도였다. 그런데 표 1~9를 다 뒤져 봐도 latency 숫자가 없었다. 전부 메모리와 정확도뿐이다.\\
What nagged me the whole time was speed. But searching through Tables 1–9, there are no latency numbers — only memory and accuracy.

reordering은 곱셈·덧셈 총량은 같지만, 큰 행렬곱 한 번을 작은 계산 144번으로 쪼갠다. 게다가 출력 Yo(75,264개)를 144번 읽고 더하고 다시 쓰고, 입력 X도 144번 다시 읽는다. 계산량은 같아도 메모리 접근이 늘고, NPU나 GPU처럼 한 번에 많이 계산하는 하드웨어는 놀게 된다. segmentation과 tunneling은 겹치는 여백과 가운데 조각 때문에 계산량 자체가 약 1.7배다. 결국 이 논문은 **시간을 내주고 메모리를 얻는** 거래다. 단일 코어 MCU는 원래 하나씩 계산하는 구조라 손해가 상대적으로 작으니 RAM 256KB 환경에서는 맞는 거래지만, RAM이 넉넉하고 FPS가 중요한 곳이라면 방향이 반대가 된다.\\
Reordering keeps the total multiply-adds the same but splits one large matrix multiply into 144 small ones. On top of that, it reads, adds to, and writes back the output Yo (75,264 values) 144 times, and re-reads the input X 144 times too. Computation stays the same but memory traffic grows, and hardware that computes in bulk, like an NPU or GPU, sits idle. Segmentation and tunneling raise the computation itself to about 1.7× because of the overlapping margins and the center patch. In the end, the paper trades **time for memory**. A single-core MCU computes one thing at a time anyway, so the loss is relatively small and the trade makes sense at 256KB of RAM; where RAM is plentiful and FPS matters, it flips the other way.

---

## RasEyes에 가져올 수 있는 것

RasEyes가 돌아가는 Orange Pi 5는 RAM이 GB 단위라 메모리가 병목이 아니다. 게다가 RK3588 NPU는 채널 병렬 처리가 강점이라 reordering을 그대로 쓰면 오히려 느려질 것이다. 그래서 가져올 수 있는 건 기법 자체보다는 관점에 가깝다. 피크 메모리는 weight가 아니라 activation에서 나오고, 레이어별로 따져야 한다는 것.\\
The Orange Pi 5 that RasEyes runs on has gigabytes of RAM, so memory isn't the bottleneck. And since the RK3588 NPU's strength is parallel channel processing, applying reordering as-is would likely slow things down. So what I can take away is closer to a perspective than a technique: peak memory comes from activations, not weights, and it has to be examined layer by layer.
