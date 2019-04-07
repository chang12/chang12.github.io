---
layout: post
title: 이전 트랜잭션의 결과를 덮어씌우는 문제를 해결하려면?
tags:
  - transaction
  - mysql
  - hibernate
---


API 서버에서 남긴 로그와, DB 상태가 일치하지 않는 경우를 만났습니다. API 서버의 Controller 는 HTTP 요청에 실린 파라미터 값을 가지고 테이블의 레코드의 컬럼 값을 업데이트 하고, 이에 대한 로그를 JSON 으로 남깁니다. 그런데 로그에는 파라미터 값이 적혀있고, DB 의 해당 레코드의 해당 컬럼은 null 인 상황입니다. DBMS 는 MySQL 5.7 입니다.

## 정확히 무슨 일이 벌어진거지?

해당 레코드를 업데이트 하는 Controller 들의 로그들을 나열해보니, 문제가 된 로그 14ms 뒤에 다른 Controller 에서 해당 레코드의 다른 컬럼을 업데이트 하는 요청에 대한 로그가 있었습니다. 설명을 위해 가상의 테이블 `create table item (id int, a int, b int, primary key (id))` 을 가정하겠습니다. 시간순으로 상황을 정리해보면 아래와 같습니다.

* 태초에 `id = 1, a = null, b = null` 인 `item` 레코드 `i1` 존재.
* 트랜잭션 T1 에서 `i1` select (`a = null, b = null`).
* 트랜잭션 T2 에서 `i1` select (`a = null, b = null`).
* 트랜잭션 T1 에서 `i1.a = 100` 로 assign 후 update (`a = 100, b = null`).
* 트랜잭션 T2 에서 `i1.b = 200` 로 assign 후 update (`a = null, b = 200`).
* `i1` 은 `id = 1, a = null, b = 200` 으로 update 됨. 따라서 로그 (`a = 100`) 와 DB (`a = null`) 가 불일치.

코드에서 트랜잭션의 isolation level 을 신경쓰지 않았으므로 default 값이 쓰였고, MySQL 5.7 DBMS 의 default isolation level 은 `REPEATABLE READ` 입니다. [MySQL 5.7 DBMS 의 `REPEATABLE READ` isolation level 은 일반 select 문을 사용할 경우 lock 을 걸지 않으므로](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html), 트랜잭션 T1 가 `i1` 을 select 한 뒤에 트랜잭션 T2 도 `i1` 을 select 할 수 있었습니다.

## 상황을 재현해보려면?

### MySQL DBMS

Docker 로 띄우면 편리합니다. [docker hub 의 mysql 페이지](https://hub.docker.com/_/mysql) 를 참고해서 `mysql:5.7.25` 이미지를 사용합니다.

```sh
docker pull mysql:5.7.25
docker run -p {port_you_want}:3306 --name {name_you_want} -e MYSQL_ROOT_PASSWORD={password_you_want} -d mysql:5.7.25
```

### Gradle 프로젝트

문제의 상황을 별도의 Gradle 프로젝트로 간단히 재현합니다. Kotlin + Spring Boot + Hibernate 구성입니다. [Spring Initializr 페이지](https://start.spring.io/) 에서 Gradle Project + Kotlin + Spring Boot 2.1.4 를 선택하고 Dependencies 에서 JPA 를 추가했습니다. MySQL JDBC Driver 를 위해 `'mysql:mysql-connector-java:8.0.15'` dependency 를 추가하고, `application.properties` 에 DB 접속을 위한 설정값을 명시하고, 테스트 코드를 작성합니다. 위에 시간순으로 정리한 상황을 재현하기 위해 `CountDownLatch` 로 Thread 의 코드 실행 순서를 제어했습니다.

```kotlin
@Autowired
lateinit var factory: EntityManagerFactory

@Test
@Rollback(false)
fun testConcurrentTransactions() {
    // Thread 코드 실행 순서 제어를 위한 CountDownLatch 들.
    val t1SelectLatch = CountDownLatch(1)
    val t2SelectLatch = CountDownLatch(1)
    val t1CommitLatch = CountDownLatch(1)

    val t1 = thread {
        val em = factory.createEntityManager()
        em.transaction.begin()
        val item = em.find(Item::class.java, 1)
        t1SelectLatch.countDown()
        t2SelectLatch.await()
        item.a = 100
        em.persist(item)
        em.transaction.commit()
        t1CommitLatch.countDown()
    }
    val t2 = thread {
        t1SelectLatch.await()
        val em = factory.createEntityManager()
        em.transaction.begin()
        val item = em.find(Item::class.java, 1)
        t2SelectLatch.countDown()
        t1CommitLatch.await()
        item.b = 200
        em.persist(item)
        em.transaction.commit()
    }

    // Thread 코드 실행 완료 때 까지 Test 메서드 종료되지 않도록.
    t1.join()
    t2.join()
}
```

재현됬는 지 결과를 확인합니다.

```
mysql> select * from item where id = 1;
+----+------+------+
| id | a    | b    |
+----+------+------+
|  1 | NULL |  200 |
+----+------+------+
1 row in set (0.00 sec)
```

## 어떻게 해결할까?

근본적인 해결책은 [Ditto Kim 님의 블로그 글](https://blog.sapzil.org/2017/04/01/do-not-trust-sql-transaction/) 에 잘 정리되어 있습니다. 트랜잭션의 isolation level 을 높이거나, 일반 select 이 아닌 `select for update` 구문을 활용해서 트랜잭션 T1 의 select 때 해당 레코드에 lock 을 걸게 만들거나 혹은 테이블에 version 컬럼이나 timestamp 컬럼을 추가한 뒤 optimistic locking 을 써서 트랜잭션 T2 가 실패해서 rollback 된 뒤 적절히 retry 되도록 유도하는 것입니다.

그러나 이번 경우에만 한정지어 생각해보면, 트랜잭션 T2 의 의도는 `i1.b = 200` 로 업데이트 하는 것인데 반해, update 쿼리에는 자신이 읽어들인 `i1.a = null` 까지 `set` 절에 포함시켜 버린 게 아쉽습니다. 이는 hibernate 의 `EntityManager#persist` 의 default 동작인데, `Entity` 클래스에 `@DynamicUpdate` annotation 을 달아서 값이 바뀐 컬럼만 `set` 절에 포함시키도록 동작을 변경할 수 있습니다.

```kotlin
@Entity
@DynamicUpdate
data class Item (
    @Id
    var id: Int? = null,

    var a: Int? = null,
    var b: Int? = null
)
```

콘솔 출력에서 동작의 차이를 확인할 수 있습니다.

```sh
# @DynamicUpdate X : update 때 a, b 컬럼 모두 set
Hibernate: select item0_.id as id1_0_0_, item0_.a as a2_0_0_, item0_.b as b3_0_0_ from item item0_ where item0_.id=?
Hibernate: select item0_.id as id1_0_0_, item0_.a as a2_0_0_, item0_.b as b3_0_0_ from item item0_ where item0_.id=?
Hibernate: update item set a=?, b=? where id=?
Hibernate: update item set a=?, b=? where id=?

# @DynamicUpdate O : update 때 변경된 컬럼만 set
Hibernate: select item0_.id as id1_0_0_, item0_.a as a2_0_0_, item0_.b as b3_0_0_ from item item0_ where item0_.id=?
Hibernate: select item0_.id as id1_0_0_, item0_.a as a2_0_0_, item0_.b as b3_0_0_ from item item0_ where item0_.id=?
Hibernate: update item set a=? where id=?
Hibernate: update item set b=? where id=?
```

바뀐 결과도 확인할 수 있습니다.

```
mysql> select * from item where id = 1;
+----+------+------+
| id | a    | b    |
+----+------+------+
|  1 |  100 |  200 |
+----+------+------+
1 row in set (0.00 sec)
```

## 레퍼런스

* [MySQL 5.7 Reference Manual -
 14.7.2.1 Transaction Isolation Levels](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html)
* [SQL 트랜잭션 - 믿는 도끼에 발등 찍힌다](https://blog.sapzil.org/2017/04/01/do-not-trust-sql-transaction/)
* [Hibernate Tips: How to exclude unchanged columns from generated update statements](https://thoughts-on-java.org/hibernate-tips-exclude-unchanged-columns-generated-update-statements/)