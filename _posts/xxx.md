1. llm 으로 무언가 생성하는 module 을 만들어 b2b 로 제공하는 일에 참여하고 있다.
2. 고객사의 니즈를 들어보니, [i] 요청 건들 중 처리하지 못하고 실패하는 경우가 생기는 것은 괜찮고, [ii] 시간이 오래 걸리는 것도 괜찮으나, [iii] 정상적으로 요청이 처리 되었는데, 생성된 결과물에 하자가 있는 것은 치명적이라고.
3. 그래서 생성된 결과물에 하자가 있는지 검증하는 단계를 module 에 추가하기로 했다. 하자가 있는지 / 없는지 boolean 을 return 하는 `binary classifier` 인 것이다.
4. binary classifier 의 결과를 4가지 경우로 분류할 수 있다.

|  | 실제 값 = true | 실제 값 = false |
| --- | --- | --- |
| classifier 의 값 = true | true positive (tp) | false positive (fp)  |
| classifier 의 값 = false | false negative (fn) | true negative (tn) |

5. 요청 건들 중 처리하지 못하고 실패하는 경우 = 검증 단계에서 하자가 있다고 판단한 경우 = 전체 중 classifier 의 값이 false 인 경우의 비율 = (fn + tn) / (tp + fp + fn + tn) = 1 - (tp + fp) / (tp + fp + fn + tn) = 1 - `prevalence` 이다.
6. 정상적으로 요청이 처리 되었는데, 생성된 결과물에 하자가 있는 경우 = classifier 의 값이 true 인데 실제로는 false 인 경우의 비율 = fp / (tp + fp) = 1 - tp / (tp + fp) = 1 - `precision` 이다.
7. 그러니 고객사의 니즈는 = prevalence 는 낮아도 괜찮지만 & precision 은 거의 1 에 가까워야 한다, 라고 표현할 수 있다는 것을 새롭게 알게 되었다.
8. 그러니 내가 해야할 일은 precision 이 거의 1 에 가까운 binary classifier 를 만드는 것이다.
