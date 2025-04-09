---
layout: post
title: "mixpanel 도입의 impact 를 어떻게 measure 할 수 있을까?"
tags: []
---

mixpanel (혹은 다른 product analytics saas) 도입의 impact 를 어떻게 measure 할 수 있을까?

조직이 mixpanel (혹은 다른 product analytics saas) 를 도입하려는 목적은 무엇일까? 에서 얘기를 시작할 수 있겠다.

조직이 as-is 로는 어떠한 상황인데, mixpanel (혹은 다른 product analytics saas) 를 도입하여, 어떤 to-be 로 변화하고 싶은지? 를 생각해보면 그게 곧 도입하려는 목적이라고 할 수 있겠다. 

아마도 bigquery 같은 sql 기반의 data warehouse 를 사용하고 있는 조직일 것이다. 그러니 데이터를 집계하고 분석하려면 sql 을 작성해야 할 것이다. 그러니 sql 을 작성할 줄 모르는 구성원들은 데이터를 집계하고 분석하고 싶을 때 sql 을 작성할 줄 아는 구성원들에게 요청하고 있을 것이다. ad-hoc 한 경우라면 작성된 sql 이 slack 에서 공유 되거나, 실행 결과가 google sheet 같은 곳에 붙여져 공유 될 것이고, 지속적으로 봐야하는 경우이고 조직에서 bi (business intelligence) saas 를 쓰고 있다면 거기에 report 혹은 dashboard 가 만들어져 link 가 공유될 것이다.

그러한 request 에서 response 까지의 end-to-end time 이 길고, 이를 줄이기 위함일 것이다. product / business 를 개선하는 iteration 에서 데이터를 집계하고 분석하는 것도 한 과정을 차지할 것이니, 그 시간을 줄이면 전체 개선 iteration 의 시간도 줄일 수 있을 것이다. 그리고 sql 을 작성할 줄 아는 구성원들 관점에서는, 기존에 sql 을 작성할 줄 모르는 구성원들의 요청을 받아 대신 처리해주는 데 사용하던 시간을 다른 곳에 사용할 수 있게 될 것이다.

그러니, 데이터를 집계하고 분석하는 task 마다 시작 시각과 종료 시각을 알 수 있으면, 필요한 것들을 measure 할 수 있을 것이다. 하지만 그렇게 정말 timestamp 2개를 수집하는 것은 현실적으로 불가능할 것이다. 개인적으로는 google sheet 에 30분 마다 무엇을 했는지 간단히 한 줄로 적고, 만약 그게 회사 task 인 경우에는 해당 task 의 notion page link 를 적으니, 각 task 마다 내가 사용한 시간을 30분 단위로는 알 수 있다. 하지만 이를 모든 구성원들에게 강요하기는 힘들 것이다. 그러니 데이터를 집계하고 분석하는 task 를 notion database 한 곳에 모으고, 시작 날짜와 종료 날짜를 적고, 종료 했을 때 이 task 에 몇시간 정도 사용했는지 대략적으로라도 적게 하는 정도로 하는 것이 최선일 것 같다. 누가 요청 했고 누가 처리 했는지도 남기면 좋을 것이다. mixpanel (혹은 다른 product analytics saas) 를 도입하는 중에 조직 구성원들의 입사/퇴사가 유의미하게 크게 발생하여 전체 volume 이 늘어나거나/줄어들 수 있으니.

아무쪼록 이 정도의 데이터를 수집하면, 어느 기간 동안 task 를 처리하는 데 사용된 전체 시간이 얼마이고, 각 task 마다 처리되는 데 까지 걸린 시간을 일 단위로는 집계할 수 있으니, mixpanel (혹은 다른 product analytics saas) 도입 전/후로 추이를 확인하여 도입의 impact 를 어느정도 measure 할 수 있을 것이다. 
