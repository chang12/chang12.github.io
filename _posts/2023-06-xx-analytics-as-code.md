(data) analytics 가 체계적으로 이뤄지는 모습을 지향한다.

무언가가 체계적으로 이뤄진다는 것은, [1] best practice 가 정립 되어 있고, [2] 그 best practice 에 따라 무언가가 이뤄지는 것이다. best practice 를 정립하는 것은 여러가지 형태로 이뤄질 수 있다. notion 같은 곳에 문서를 잘 적는 게 될 수도 있고. 하지만 best practice 에 따라 이뤄지도록 만들기 위해서는 code 로 관리하는 수 밖에 없다는 생각이다.

- 정립한 best practice 에 기반하여, 선언과 구현을 명확히 분리 한다.
- 분리하는 과정에서 interface (= 선언 되어야 하는 것) 이 정해진다.
- best practice 를 따르도록 구현한다.
- 사용자에게는, 구현은 감추고, interface 에 따라 선언만 하도록 한다.

처음 software engineering 을 공부할 때 읽었던 책에서, 절차지향과 객체지향의 차이에 대해 설명한 내용, 을 떠올리게 한다.

이렇듯, 무언가가 체계적으로 이뤄지도록 만들기 위해서, 기존에 code 로 관리되지 않던 것들을 -> code 로 관리하는 모습, 을 얘기할 때 `-as-code` 를 붙이는 듯 하다. infrastructure (특히 aws 같은 cloud 서비스 상의) 를 관리할 때 많이들 사용하는 terraform 이 스스로를 설명할 때 infrastructure as code 라고 얘기하는 것 처럼. 

그런 점에서 글의 첫 문장에서 얘기한 "*analytics 가 체계적으로 이뤄지는 모습*" 은 **analytics-as-code** 라고 할 수 있겠다.

# metrics store (metric-as-code)

가장 먼저 고민했던 것은 metric 이었다. 

- 동일한 metric 인데, 얘기하는 사람마다 값이 다르다. 누가 집계 했는지에 따라 -> 집계하는 sql 이 달라서 -> 값이 달라진 것이다.
- 완전히 동일한 metric 은 아니나, 집계하는 방식이 동일해야 하는 metric 들에 대해서 (e.g. [검색] 유저 수, 와 [질문] 유저 수), 집계하는 방식 (정확히는, 집계하는 sql 의 구조) 가 동일하지 않다.

airbnb 에서도 과거에 그러한 고통을 겪었다고.

> For data consumption, we heard complaints from decision makers that different teams reported different numbers for very simple business questions, and there was no easy way to know which number was correct. Years ago, when Brian, our CEO, would ask simple questions like which city had the most bookings in the previous week, Data Science and Finance would sometimes provide **diverging answers using slightly different tables, metric definitions, and business logic.** Over time, even data scientists started to second guess their own data, confidence in data quality fell, and trust from decision makers degraded.
> 
> [How Airbnb Achieved Metric Consistency at Scale (Part-I: Introducing Minerva — Airbnb’s Metric Platform)](https://medium.com/airbnb-engineering/how-airbnb-achieved-metric-consistency-at-scale-f23cc53dea70#:~:text=For%20data%20consumption,decision%20makers%20degraded.)

sql 을 전부 사람이 작성하는 것, 이 문제라고 생각 했다. 아주 기본이라 할 수 있는 [앱 방문 유저 수] 같은 걸 집계하는 sql 을 작성하게 했을 때도, 작성하는 사람마다 sql 이 다양하다. 집계되는 값이 다르지 않는 선에서 sql 의 구조에 차이가 있기도 하고, 집계되는 값이 달라지는 차이가 있기도 하고 (e.g. 집계 기간 내에 유저의 attribute 가 바뀌었을 때 -> 그럼에도 1명으로 셀 것인지 or attribute 값마다 1명 씩으로 셀 것인지). 

그래서 선언과 구현을 분리 했다.

- 사람은 metric 에 대한 정의를 yaml 로 선언 한다.
- 선언된 내용에 기반하여, 집계하는 sql 을 code 가 생성하여 실행 한다.

이에 대한 자세한 내용을 팀 동료가 회사 블로그 글로 적어주었었다.

[Metric을 체계적으로 관리하기: Metrics Store](https://blog.mathpresso.com/metric-%EC%9D%84-%EC%B2%B4%EA%B3%84%EC%A0%81%EC%9C%BC%EB%A1%9C-%EA%B4%80%EB%A6%AC%ED%95%98%EA%B8%B0-metrics-store-ccc4dc1d6768)
