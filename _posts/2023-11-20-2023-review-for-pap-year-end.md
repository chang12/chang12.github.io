---
layout: post
title: "PAP 2주년 행사를 위해 적어보는 지난 2023년의 기억들"
tags: []
---

회사에서 나의 시간을 사용하는 행태가 많이 바뀌었다. 2022년 하반기부터 2023년 상반기까지 data 직군의 많은 동료분들이 여러 사정으로 회사를 떠났다. 그리고 조직 구조 변경에 따라, data analyst 의 manager 가 나에서 -> data analyst 가 속한 business group 의 leader 로 바뀌었다. 그 전 까지는 managing 을 나의 주요 역할과 의무로 여기며 대부분의 시간을 사용했었고, 그것만으로도 벅찼었는데, 이러한 변화들을 거치며, 많은 시간이 생겼다.

상반기에는 **measure** 와 **metric review** 를 주요한 주제로 삼았다. 인상 깊게 읽었던 [Working Backwards](https://www.amazon.com/Working-Backwards-Insights-Stories-Secrets/dp/1250267595) 책에서 얘기하는 DMAIC 에 입각해서 사고한 결과였다. DMAIC = Define -> Measure -> Analyze -> Improve -> Control 이다. 이 중에서 Improve 와 Control 이 우리가 흔히 얘기하는 **가시적인 성과** 이다. 그러한 성과를 잘 내기 위해서는 이전 단계들이 충실히 수행 되어야 한다는 게 책에서 이해한 교훈이었다. 풀고자 하는 문제는 명확히 정의 (Define) 되어있다는 가정하에, 이어서, 시도하는 것들의 impact 를 명확히 측정 (Measure) 해야하고, 그와 더불어 주기적인 metric review 를 통해 business 를 정량적으로 이해 (Analyze) 해야 한다고 생각했다. 그래서 2가지를 시도 했다.

1. business group 들에서 진행되는 project 들을 한데 모아서 notion 에 score board 라는 database 를 만들었다. notion api 에 기반한 간단한 code 를 작성하여, business group 들에서 진행되는 project 들이 담긴 notion database 들의 내용을 copy & paste 하여 하나의 notion database 에 모으고, 그렇게 모인 각 project 들의 impact 가 어떠한지 기록하는 것이다.
2. business 중 하나를 골라서 스스로 weekly metric review 를 했다.

결과적으로 둘 다 실패했다.

1. 다른 사람들을, 내가 생각하는 방향으로 바꾸는 것은, 정말로 어려운 일이다. 일단 사람이 바뀐다는 것 자체가 어려운 일인데, 스스로 어떤 motivation 을 가지고 변화하는 게 아니라, 외부에서 누군가의 영향으로 바뀌는 것은 훨씬 더 어려운 일이다. 지금 생각나는, 그나마 가능성이 있는 방법은 2가지이다. 첫번째는, 어떤 system 을 만들어서, 그 system 을 통해서만 일이 가능하도록 만들고 & 내가 생각하는 방향을 따르지 않을 경우 system 에서 일이 진행되지 않도록 coding 해버리는 것이다. 실험 이라는 한정된 scope 에서는, 실험 플랫폼을 통해서 그렇게 하는 것이 가능했다. 하지만 전사의 project 들이 진행되는 방식을 일원화하고 그렇게 system 으로 제어하는 것은 불가능하다. 맞는지도 모르겠고. 두번째는 바꾸고자 하는 사람의 manager 를 설득하여, 나와 같은 생각을 가지게 하고, 그래서 내가 생각하는 방향으로 manager 가 그 사람에게 얘기하도록 하는 것이다. 일단 내 스스로 나의 생각에 대해 엄청나게 높은 수준의 논리와 확신을 갖추어야 하고, 그걸 가지고 엄청나게 많은 커뮤니케이션을 해야한다. 하지만 그러한 확신과 열정이 없어서, 시도조차 하지 못했다.
2. 해당 business 에 대해 나 스스로 100% 설득이 되지 못하여, 애정과 열정을 가지지 못했다. 그래서 일련의 review 들은 수행했으나 -> 그에 기반하여 다른 사람들을 설득하여 improve 를 시도하는 데에 실패했다.

그랬을 때... 이어서 무엇을 해야할까? 를 생각했다.

1. 종합해봤을 때, 나의 생각을 잘 정리하고 좋은 방식으로 커뮤니케이션하여 다른 사람들을 설득하는 데 실패했으니 -> 이러한 역량을 발전시켜서 다시 시도하는 것이 보편적으로 올바른 방향일 것이다. 하지만 그쪽으로 마음이 가지 않았다. 그래서, 최대한 다른 사람들에게 dependency 가 걸리지 않고, 혼자서 (혹은 정말 최소한의 도움으로) 할 수 있는 쪽을 생각했다.
2. 존재하는 business 들에 직접적으로 관련되어 있지 않더라도, 내가 생각하기에 정말로 중요한 본질에 가까운 무언가가 있을까? 를 생각했다.

[유저들의 꾸준한 학습을 유도하는 것] 을 목표로 삼고, 첫번째 시도로 [꾸준한 학습을 유도하는 push campaign] 에 대한 일련의 실험들을 하고 있다. 그 과정에서 맞닿뜨리는 push system 에 대한 개선안들을 동료 data engineer 분께 요청하고 있고. 궁극적으로는 [How Duolingo reignited user growth](https://www.lennysnewsletter.com/p/how-duolingo-reignited-user-growth) 글에 나오는 것들 중에서 bench marking 해볼 수 있는 모든 것들을 다 시도해보는 것, 이 목표라고 할 수 있겠다. 실험들에서 일부 성과들이 있었으나, 충분하다는 생각이 들진 않는다. 다만 계속해서 떠오르는 & 시도할 각이 보이는 BACK LOG 들이 있어서 계속 시도하고 있다. 그렇게 아직은 [진행 중] 이어서 그런지... [회고] 에 대해서는 특별한 생각이 들지 않는다. 일단은 어느쪽이던 일단락 될 때 까지 계속 해보는 걸로.
