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
9. precision 을 높이기 위해서는, 일단 precision 을 측정할 수 있어야 한다.
10. 그러려면 “실제 값” 을 알아야 하니, 생성 결과 n개를 domain 전문가에게 전달하여 분류를 요청했고, 이를 통해 “실제 값” 을 얻었다.
11. binary classifier 는 llm api 를 사용해서 만들었다. prompt engineering 인 것이다. input 을 포함한 적절한 instruction 을 적고, boolean field 로 구성된 json 을 output 으로 달라고 하는 것이다.
```python
input = ...
message = create_message(input)
llm_response = classify_using_llm(message)
is_xxx: bool = llm_response["is_xxx"]
```
12. 10 의 input 들을 llm 에 기반한 binary classifier 에 넘겨서 boolean 값을 얻는다.
13. 12 의 boolean 값을 < - > 10 에서 확보한 ground truth 과 비교 하여, false positive 인 것들을 나열한다.
14. 그 중 하나를 골라서, `create_message` 를 호출하여 llm 에게 넘길 prompt 를 얻는다.
15. (openai 의 llm 을 사용한다면) chatgpt 에 copy & paste 하여 false positive 가 되는 것을 재현한다.
16. 이를 true negative 로 바꿀 idea 를 고민하며 true negative 가 될 때 까지 prompt 를 수정한다.
17. 수정한 prompt 로 다시 classifier 를 돌렸을 때, precision 이 높아졌다면 수정한 prompt 를 취하고, 아니라면 기존 prompt 를 유지한다. 
18. 원하는 수준의 precision 을 달성할 때 까지 12 ~ 17 을 반복한다. 
19. 이렇게 하니까, 리듬감 있게 집중할 수 있었다.
