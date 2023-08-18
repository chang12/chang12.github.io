---
layout: post
title: "analytics-as-code 에 대한 주절주절"
tags: []
---

(data) analytics 가 체계적으로 이뤄지는 모습을 지향한다.

무언가가 체계적으로 이뤄진다는 것은, 

1. best practice 가 정립 되어 있고, 
2. 그 best practice 에 따라 이뤄지는 것이다. 

best practice 를 정립하는 것은 여러가지 형태로 이뤄질 수 있다. notion 같은 곳에 문서를 잘 적는다던가.

하지만 best practice 에 따라 이뤄지도록 만들기 위해서는, code 로 관리하는 수 밖에 없다.

- 정립한 best practice 에 기반하여, 선언과 구현을 명확히 분리 한다.
- 그렇게 분리된 선언과 구현의 경계가 곧 interface 가 된다.
- 사용자에게는, 구현은 감추고, interface 에 따라 선언만 하도록 한다.

처음 software engineering 을 공부할 때 읽었던 책에서, 절차지향과 객체지향의 차이를 설명한 내용이 있었다. 이를 떠올리게 한다.

이렇게 기존에 code 로 관리되지 않던 것들을 -> code 로 관리할 때, `-as-code` 라는 suffix 를 붙인다. infrastructure 를 관리할 때 많이들 사용하는 [terraform](https://www.terraform.io/) 이 스스로를 설명할 때, infrastructure as code 라고 얘기하는 것 처럼.

그러니, 글의 첫 문장에서 얘기한 "analytics 가 체계적으로 이뤄지는 모습" 은 **analytics-as-code** 라고 할 수 있겠다.

# metrics store (metric-as-code)

가장 먼저 고민했던 것은, **metric** 이었다.

- 동일한 metric 인데, 얘기하는 사람마다 값이 다르다. 누가 집계 했는지에 따라 -> 집계하는 sql 이 달라서 -> 값이 달라진 것이다.
- 집계하는 방식이 동일해야 하는데, 그렇지 않다. ["검색" 유저 수] 와 ["질문" 유저 수] 는 집계하는 sql 의 구조가 동일해야 할 텐데 -> 그렇지 않다.

[airbnb 에서도 과거에 그러한 고통을 겪었다고.](https://medium.com/airbnb-engineering/how-airbnb-achieved-metric-consistency-at-scale-f23cc53dea70#:~:text=For%20data%20consumption,decision%20makers%20degraded.)

> For data consumption, we heard complaints from decision makers that different teams reported different numbers for very simple business questions, and there was no easy way to know which number was correct. Years ago, when Brian, our CEO, would ask simple questions like which city had the most bookings in the previous week, **Data Science and Finance would sometimes provide diverging answers using slightly different tables, metric definitions, and business logic.** Over time, even data scientists started to second guess their own data, confidence in data quality fell, and trust from decision makers degraded.

문제가 무엇일까? -> sql 을 "사람이" 작성하는 것이 문제이다. 

아주 기본적인 것이라 할 수 있는 [앱 방문 유저 수] 마저도 작성하는 사람마다 sql 이 다르다.

- 들여쓰기를 어떻게 할 것인지? 같은 단순 convention 차원의 작은 차이도 있고,
- 집계 기간 내에 유저의 dimension 값이 바뀌었을 때 -> 그럼에도 1명으로 셀 것인지? 아니면 dimension 값마다 다른 사람으로 셀 것인지? 같이 집계 값을 달라지게 만드는 큰 차이도 있다.

그래서 선언과 구현을 분리 했다.

- [사람은] yaml 로 metric 을 선언한다.
- 이에 기반하여 [program 이] sql 을 생성하여 실행한다.

이에 대한 자세한 내용을 팀 동료가 회사 블로그 글로 적었다. [Metric을 체계적으로 관리하기: Metrics Store](https://blog.mathpresso.com/metric-%EC%9D%84-%EC%B2%B4%EA%B3%84%EC%A0%81%EC%9C%BC%EB%A1%9C-%EA%B4%80%EB%A6%AC%ED%95%98%EA%B8%B0-metrics-store-ccc4dc1d6768)

# metric hierarchy

그 다음으로 고민했던 것은, **metric hierarchy** 였다.

방금 언급한 [Metric을 체계적으로 관리하기: Metrics Store](https://blog.mathpresso.com/metric-%EC%9D%84-%EC%B2%B4%EA%B3%84%EC%A0%81%EC%9C%BC%EB%A1%9C-%EA%B4%80%EB%A6%AC%ED%95%98%EA%B8%B0-metrics-store-ccc4dc1d6768) 글의 마지막 문단에서도, 다음 할일로 metric hierarchy 를 얘기 했었다.

> metrics store를 통해서, metric을 체계적으로 관리하고, 이전보다 정확하고 빠른 집계가 가능해졌습니다.
> 
> 이제는 metrics store를 전사적으로 활발히 사용할 수 있는 방법, 특히 metrics store를 통해서 **metric hierarchy** 를 만들어내는 것에 대해 고민하고 있습니다. QANDA의 각 business model 별로 가장 집중해야 하는 하나의 metric을 정의하고, 그로부터 파생되는 metric들을 tree 형태로 만들어내기를 기대하고 있습니다. 이를 통해 “어떤 metric에 집중해야 하는지”를 명확히 하는 것을 목표로 합니다.

metric 들은 동등하지 않다. parent/child 관계를 통해 hierarchy 를 이룬다.

- 이 metric 을 증가/감소 시키려는 시도 들이 -> 궁극적으로 무엇을 위함이지? (= metric 의 **parents** 에 대한 탐색)
- metric 이 증가/감소 했네. -> 원인을 알고 싶다. -> 무엇을 더 보면 좋을까? (= metric 의 **children** 에 대한 탐색)


그러한 hierarchy 에 기반한 탐색을 통해, 조직 구성원들이 아래와 같은 질문들에 대한 답을 찾기를 원한다.

- 조직의 성공을 위해 각자가 무엇을 하면 좋을지? 
- 그렇게 무언가를 했을 때 이를 어떻게 measure 하면 좋을지?
- 그렇게 measure 한 결과에 기반하여 무엇을 이어나가면 좋을지? 

이때, 그러한 내용들이 각자의 머릿 속에 파편화 되는 것이 아니라, 조직 공동의 **knowledge** 로서 누적 되기를 원했다. 

metrics store 덕분에, metric 들이 yaml file 로 관리되고 있어서, 간단하게 해결할 수 있었다.

- metric 에 대한 yaml file 에 `parent_metrics` 라는 field 를 추가하고,
- 이를 통해 metric 들을 parent/child 관계의 tree 자료구조로 만들고,
- 이를 적절한 도구로 (e.g. [markmap](https://markmap.js.org/)) render 하여 hierarchy 를 그렸다.

![2023-06-13-metric-hierarchy.png](https://raw.githubusercontent.com/chang12/chang12.github.io/master/images/2023-06-13-metric-hierarchy.png)

# dashboard-as-code

이어서 [metric 값을 조회할 수 있는 chart] 에 대해 고민 했다. 대략 아래와 같은 모습이다.

- metric 목록에서 원하는 metric 을 고른다.
- 조회할 기간을 지정한다.
- chart 의 가로 축은 날짜 이고, 세로 축은 metric 값이다. 이때 어떤 format 으로 보여줄지는 metric 선언에 포함되어 있어서 이를 따른다.
    - e.g. [구독 유저 수] 는 number 이다. 3-digits 마다 `,` 로 구분하여 `12,345` 처럼 적는다.
    - e.g. [결제 금액] 은 krw 이다. 앞에 `₩` 를 붙이고 3-digits 마다 `,` 로 구분하여 `₩12,345` 처럼 적는다.
    - e.g. [구독 지속률] 은 percentage 이다. 유효 숫자 3자리로 끊고 뒤에 % 를 붙여서 `12.3%` 처럼 적는다.
- metric 선언에 dimension 에 대한 내용도 포함 되어 있다. 그 내용에 기반하여 -> dimension 마다 원하는 조건을 가할 수 있다.
    - e.g. [방문 유저 수] metric 에 [국가] dimension 이 있다.
        - [국가] dimension 은 categorical 이다.
        - 그래서 가능한 값의 목록이 정해져 있다.
        - 그래서 multi-select dropdown 이 있고, 목록 중 원하는 값들을 선택할 수 있다.
- 기본적으로 **line chart** 로 보여준다. 
    - e.g. [구독 매출] 을 breakdown 하지 않고 조회하면 series 1개 이고, [국가] 로 breakdown 하면 여러 series 로 나눠진다.
    - 이때 metric 이 additive 한 경우에는 -> breakdown 했을 때 **stacked column chart** 로도 조회 할 수 있다.
        - e.g. [구독 유저 수] metric 은 additive 하다. 전체 [구독 유저 수] = [국가] 별 [구독 유저 수] 의 sum 과 일치 한다.
        - e.g. [구독 지속률] metric 은 additive 하지 않다. 전체 [구독 지속률] 과 < - > [국가] 별 [구독 지속률] 의 sum 이 일치하지 않는다.

[holistics](https://www.holistics.io/) 라는 business intelligence saas 를 (과거 version 인 2.7 로) 사용 중이다. 그래서 holistics 에서 이러한 모습을 구현하려고 했으나, 일부 불가능한 것들이 있었다.

- 값의 format (number, krw, percentage 등) 을 chart 를 [조회 할 때] 바꿔 볼 수 없다. chart 의 설정을 아예 바꾸어야 한다.
- visualization 방식 (line, stacked column 등) 도 마찬가지. chart 를 [조회 할 때] 바꿔 볼 수 없다. chart 의 설정을 아예 바꾸어야 한다.

그래서 data engineer 동료 분이 직접 만들었다. 회사 구성원들이 이를 통해 metric 값을 조회하고 있다.

사실 dashboard-as-code 에 대해 처음 생각했던 것은, [실험 (a/b test) 에 대한 dashboard 를 자동으로 생성하려면?] 을 고민할 때 였다. [실험 의사결정 가이드 자동화 : QXP Tracker](https://blog.mathpresso.com/%EC%8B%A4%ED%97%98-tracking-b3e5ba7b3004) 글과도 관련된 내용이다.

- [실험] 과 [실험 그룹 할당] 에 대한 공통의 mart table 들을 만들었고,
- [어느 실험에서, 유저가 어느 실험 그룹으로 할당 되었는지] 도 metrics store 에서 하나의 dimension 으로 간주하도록 통합하였다. 

그래서 아래와 같은 일관된 (bigquery) sql 로 실험 결과를 집계할 수 있다.

```sql
select
  date_kst,
  breakdown1 as experiment_group,
  value,
from
  metric.<metric name>(
    _date_range_start => '<실험 시작 yyyy-MM-dd>',
    _date_range_end => '<실험 종료 yyyy-MM-dd>',
    _filter => '{"experiment_id": "<실험 id>"}',
    -- 실험 시작 날짜 부터 누적으로 값을 집계하기 위하여.
    _window => '<실험 시작 yyyy-MM-dd>',
    _breakdown => 'experiment_group'
  )
order by
  date_kst
  , experiment_group
```

그러니 이를 적절히 잘 감싸서 -> 실험 dashboard 를 자동으로 만들 수 있다. 

처음 시도 때는 superset 을 사용하려고 했었다. 하지만 superset 자체적으로 dimension/measure 에 기반한 semantic layer? 가 있고, 그걸 사용하는 것 위주로 practice 가 만들어져 있어서, sql 수준에서 원하는 걸 하는게 잘 안되었다. 그래서 포기 했었다. 

이제는 superset 없이 직접 개발하기 위한 기반을 갖췄고, 최초 구현 사례도 있으니, 쉽게 할 수 있을 것이다.

# 다음으로는?

funnel 도 data analytics 에서 많이 다루는 주제이다.

- 우리 팀이 다루는 funnel 들이 정확히 어떤 것들이 있지?
- 이 funnel 은 정확히 어떤 step 들로 구성되어 있지? 

같은 질문들에 대한 답도, 조직 공통의 knowledge = code 로 관리하고 싶다. **funnel-as-code** 라고 할 수 있겠다. 

2023 상반기에 [PAP (Product Analytics Playground)](https://playinpap.oopy.io/) 활동에 참여하면서, 관련하여 2편의 글을 작성 했었다.

1. [funnel-as-code : funnel 관련 집계를 code 로 관리하는 모습](https://playinpap.github.io/funnel-as-code-1/)
2.  [funnel-as-code : step-wise conversion rate 를 보여주는 line chart](https://playinpap.github.io/funnel-as-code-step-wise-conversion-rate-line-chart/)

글을 작성하며 간단한 poc 수준의 code 는 작성하였으나, 실제 회사 code 에 반영하진 못했다.

funnel 에 이어서 -> screen flow, retention, 등으로 생각이 이어지는데... 한편으로는 amplitude, mixpanel 같은 product analytics 도구의 [열화판] 을 만들려고 하는건가? 싶어서 뜨끔한다. 또 다른 한편으로는, 자신들에게 딱 맞는 적절한 인하우스 도구를 만들어서 회사의 성장을 도모한 [duolingo](https://blog.duolingo.com/duolingos-secret-weapon-our-beautiful-and-powerful-analytics-tools/) 와 [토스](https://www.youtube.com/watch?v=8ZhnUgylQgo&t=887s) 의 사례를 떠올리며 -> 자연스러운 수순인가 싶기도.

metrics store -> metric hierarchy -> 에 이어서는 **metric review** 에 대해 생각하고 있다. 인상깊게 읽었던 [Working Backwards](https://www.amazon.com/Working-Backwards-Insights-Stories-Secrets/dp/1250267595) 책에서 얘기하는 WBR (Weekly Business Review) 를 상기하면서. DMAIC (Define -> Measure -> Analyze -> Improve -> Control) 에서 Improve 와 Control 이 제대로 원활하게 이뤄지는 모습을 직접 경험해본 적 없다. 이를 위해서는 그전에 Analyze 를 정말 잘해야 한다는 것을 절실히 느끼고 있다. metrics store 는 Measure 를 체계적으로 하기 위함이고, 그 기반 위에서 Analyze 를 제대로 하려면 -> metric hierarchy 와 metric review 를 계속해서 번갈아 반복하며 이해도를 높혀야 하지 않을까.

회사에서 notion 을 쓰고 있다. 회사는 3개의 business group (+ 1개의 foundation group) 들로 구성되어 있다. 공통은 아니고 business group 마다 제각각이긴 하지만, 아무튼 business group 들에서 진행하는 project 들은 모두 notion database 로 관리되고 있다. project 에 누가 참여하는지? property 정도는 공통으로 존재하고. project 마다 목표 (objective? goal?) 가 있을 것이다. 그리고 대부분은 정량적으로 measurable 해야 = 어떤 metric 을 증가/감소 시키기 위함인지 명확해야 할 것이다. 그래야 사후에 actual impact 를 measure 할 수 있고, 이를 사전의 estimation 과 비교하며 학습하여, 더 훌륭한 조직이 될 수 있으니. 그렇게 metrics store, metric hierarchy, metric review 같은 metric 얘기들이 -> project, people 과도 참조 관계로 한데 어우러진다. 

이정도 까지 이르게 된다면... **company-as-code** 라고 할 수 있지 않을까.
