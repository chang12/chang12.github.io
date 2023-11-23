---
layout: post
title: "Working Backwards 에 입각한 SQL 작성"
tags: []
---

Amazon 의 경영 철학에 대해 다룬 [Working Backwards](https://www.amazon.com/Working-Backwards-Insights-Stories-Secrets/dp/1250267595) 책이 있다. 나는 한국어 번역판인 [순서 파괴](https://product.kyobobook.co.kr/detail/S000001687204) 로 읽었는데, 인상 깊었고 유익 했다. Amazon 의 주요 principles 에 대해 하나씩 다루는데 그 중 하나가 책의 제목인 'Working Backwards' 이다. 아주 간략히 얘기하자면, 무언가 일에 착수하기 이전에, 그 일이 완료되었을 때 낼 보도 자료 부터 작성하라는 것이다. 그 과정에서, 예를 들어 새로운 product 출시에 대한 보도 자료라면, 기존에 존재하는 경쟁 product 들에 대비하여 어떤 강점을 강조할 것이고, 그러한 강점을 통해 유저에게 어떤 새로운 효용을 제공할 수 있고, 이를 만들기 위해 어떤 강력한 기능들을 갖추었는지, ... 등을 명확히 해야 할 것이다. 이를 사전에 작성함으로써 

# 최종 결과가 무엇에 대해 unique 하고, 어떤 column 들이 필요한 지? 정한다.

예를 들어 monthly ARPU (Average Revenue Per User) 라면?

- (month) 에 대해 unique 할 것이다.
- 해당 month 의 arpu 값을 column 으로 갖추어야 한다. 편의상 currency 는 KRW 라고 하자.

|  month  | arpu\_in\_krw |
|:-------:|:-----------:|
| 2023-01 |     1034    |
| 2023-02 |     1132    |
| 2023-03 |     1099    |
|   ...   |     ...     |

SQL 틀을 잡는다. BigQuery 기준.

```sql
select
  safe_cast(null as string) as month,
  safe_cast(null as float64) as arpu_in_krw,
```

# 그 최종 결과를 만들기 위해 직전에 어떤 schema 가 필요한 지? 정한다.

