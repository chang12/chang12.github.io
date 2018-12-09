---
layout: post
title: MySQL 의 Geospatial Index (2)
tags:
  - mysql
  - geospatial
---

이전 글인 [MySQL 의 Geospatial Index (1)](https://chang12.github.io/mysql-geospatial-index-1/) 에서 MySQL 5.7.5 버전의 InnoDB 스토리지 엔진부터 지원하는 geospatial index 를 활용해서 쿼리 속도를 높히는 방법에 대해 알아봤었습니다. 이번 글에서는 작성 시점 기준으로 최신 메이저 버전인 MySQL 8.0 에서 어떤 차이가 있는지 알아보겠습니다.

# SRS (Spatial Reference System)

SRS 는 지구타원체를 2차원으로 표현하기 위한 좌표계 입니다. 3차원 상의 지구타원체를 완벽하게 2차원으로 표현할 수 있는 방법은 존재하지 않으므로, 저마다 목적에 따른 투영법 (map projection) 을 정의하고, 그에 따라 지구타원체 표면 위의 점들을 2차원 좌표계 상의 점으로 매핑합니다. 세계 지도를 그리는데 쓰이는 것으로 유명한 메르카트로 도법이 그 예입니다.

# MySQL 5.x 의 SRS

MySQL 5.x 버전에서도 SRS 를 지정할 수 있습니다. 테이블을 만들때 spatial data type 의 컬럼을 선언하고, insert 문으로 데이터를 넣을때 **SRID** 를 지정하면 됩니다. SRID 는 Spatial Reference Identifier 의 약자로 SRS 의 식별자입니다. 예를 들어 GPS 의 기준이 되는 WGS84 시스템은 SRID 4326, 단순 직교 좌표계는 SRID 0 입니다.

```sql
-- spatial data type 중 하나인 point 타입으로 컬럼 선언
create table place (p point not null, spatial index (p));

-- SRID 4326 의 point record 를 insert
insert into place (p) values (ST_GeomFromText("point(0.0 0.0)", 4326));
```

[ST_SRID](https://dev.mysql.com/doc/refman/5.7/en/gis-general-property-functions.html#function_st-srid) 함수로 컬럼값의 SRID 를 확인할 수 있습니다.

```sql
mysql> select ST_SRID(p) from place;
+------------+
| ST_SRID(p) |
+------------+
|       4326 |
+------------+
```

하지만 5.x 버전에서 SRS 를 지정하는건 메타데이터를 저장하는 정도의 의미에 그칠뿐, 실제로 연산에 사용되진 않습니다. 예를 들어 `point(0.0, 0.0)` 과 `point(3.0, 4.0)` 두 점 사이의 거리를 계산하는데 SRID 가 4326 이라면, (x, y) 좌표는 (longitude, latitude) 를 뜻하고, 이는 아프리카 대륙 남서쪽 대서양 위의 두 점이 되고 거리는 556km 입니다. 하지만 [ST_Distance](https://dev.mysql.com/doc/refman/5.7/en/spatial-relation-functions-object-shapes.html#function_st-distance) 함수는 SRS 를 인식하지 못한채 직교 좌표계 위의 두 점으로 간주하여 5.0 을 반환합니다. 따라서 별도의 [ST\_Distance\_Sphere](https://dev.mysql.com/doc/refman/8.0/en/spatial-convenience-functions.html#function_st-distance-sphere) 함수를 사용해야 합니다.

```sql
mysql> SET @g1 = ST_GeomFromText("point(0.0 0.0)", 4326);
Query OK, 0 rows affected (0.00 sec)

mysql> SET @g2 = ST_GeomFromText("point(3.0 4.0)", 4326);
Query OK, 0 rows affected (0.00 sec)

mysql> select ST_Distance(@g1, @g2), ST_Distance_Sphere(@g1, @g2);
+-----------------------+------------------------------+
| ST_Distance(@g1, @g2) | ST_Distance_Sphere(@g1, @g2) |
+-----------------------+------------------------------+
|                     5 |            555810.7203217451 |
+-----------------------+------------------------------+

```

# MySQL 8.x 의 SRS

MySQL 8.0.11 은 5108 개의 SRS 에 대한 카탈로그와 더불어 출시되었습니다. 또한 테이블을 생성할 때 컬럼의 SRID 값을 선언할 수 있습니다. 만약 컬럼의 SRID 값과 다른 SRID 의 값을 insert 하려고 시도하면 에러가 발생합니다.

```sql
create table place (p point not null SRID 4326, spatial index (p));
-- Query OK, 0 rows affected (0.09 sec)

insert into place (p) values (ST_GeomFromText("point(0.0 0.0)", 4326));
-- Query OK, 1 row affected (0.02 sec)

insert into place (p) values (point(0.0, 0.0));
-- ERROR 3643 (HY000): The SRID of the geometry does not match the SRID of the column 'p'. The SRID of the geometry is 0, but the SRID of the column is 4326. Consider changing the SRID of the geometry or the SRID property of the column.

insert into place (p) values (ST_GeomFromText("point(0.0 0.0)", 0));
-- ERROR 3643 (HY000): The SRID of the geometry does not match the SRID of the column 'p'. The SRID of the geometry is 0, but the SRID of the column is 4326. Consider changing the SRID of the geometry or the SRID property of the column.
```

그리고 몇가지 spatial function 이 SRS 를 고려하도록 개선되었습니다. SRS 를 식별하는 함수들의 목록은 [Geography in MySQL 8.0](https://mysqlserverteam.com/geography-in-mysql-8-0/) 글에서 확인할 수 있습니다. **ST_Distance** 도 그 중 하나입니다. SRID 4326 의 point 인 경우 알아서 great circle distance 를 계산해주는 것을 확인할 수 있습니다 (ST\_Distance\_Sphere 함수와 값이 다른 이유는 사용하는 지구타원체의 차이 때문).

```sql
mysql> SET @g1 = ST_GeomFromText("point(0.0 0.0)", 4326);
Query OK, 0 rows affected (0.01 sec)

mysql> SET @g2 = ST_GeomFromText("point(3.0 4.0)", 4326);
Query OK, 0 rows affected (0.00 sec)

mysql> select ST_Distance(@g1, @g2), ST_Distance_Sphere(@g1, @g2);
+-----------------------+------------------------------+
| ST_Distance(@g1, @g2) | ST_Distance_Sphere(@g1, @g2) |
+-----------------------+------------------------------+
|     555093.4785976429 |            555812.7069210884 |
+-----------------------+------------------------------+
```

# 아쉬운 점

지난번 글에서도 그렇고, 분석할 때 `어떤 point 로 부터 거리가 d 이내` 조건이 자주 필요합니다. MySQL 에서는 [ST_Buffer](https://dev.mysql.com/doc/refman/8.0/en/spatial-operator-functions.html#function_st-buffer) 함수를 사용할 수 있는데, 5.x 버전에서는 직교좌표계만 지원하기 때문에 동서남북으로 d 만큼 떨어진 위도/경도 값을 역산해서 넓은 범위로 spatial index 를 태워서 추린 뒤, 직접 ST\_Distance\_Sphere 함수로 거리를 계산하는 수 밖에 없었습니다.

그래서 MySQL 8.x 에서 함수들이 SRS 를 인식하도록 구현을 추가했다고 했을때 가장 기대했던 건 ST_Buffer 함수 였습니다. 하지만 아쉽게도 작성 시점 최신인 8.0.13 버전 기준으로 ST\_Buffer 함수는 SRID 4326 같은 geographic SRS 를 아직 지원하지 않습니다. [Detecting Incompatible Use of Spatial Functions before Upgrading to MySQL 8.0](https://mysqlserverteam.com/detecting-incompatible-use-of-spatial-functions-before-upgrading-to-mysql-8-0/) 에서 지원하지 않는 spatial function 들의 목록을 확인할 수 있습니다.

```sql
select ST_Buffer(ST_GeomFromText("point(0.0 0.0)", 4326), 100);
-- ERROR 3618 (22S00): st_buffer(POINT, ...) has not been implemented for geographic spatial reference systems.
```

# 마치며

MySQL 8.x 으로 업그레이드 되면서 SRID 가 메타데이터 이상의 의미를 지니게 되었습니다. 덕분에 SRID 를 명시하는 것 만으로 일부 연산들이 별도의 처리 없이도 기대하던 결과값을 반환하게 되었습니다. geographic SRS 에서 아직 지원되지 않는 spatial function 들이 있다는 아쉬움은 남아있지만, 최근 릴리즈인 [MySQL 8.0.13](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-13.html) 에서 [ST\_Validate](https://dev.mysql.com/doc/refman/8.0/en/spatial-convenience-functions.html#function_st-validate) 함수의 구현이 새로 추가되었듯이 시간이 지나면서 앞으로 다가올 릴리즈들에서 해소되지 않을까 기대됩니다.


# 레퍼런스

* [MySQL Server Blog 의 Upgrading to MySQL 8.0 with Spatial Data](https://mysqlserverteam.com/upgrading-to-mysql-8-0-with-spatial-data/)
* [MySQL Server Blog 의 Geography in MySQL 8.0](https://mysqlserverteam.com/geography-in-mysql-8-0/)
* [MySQL Server Blog 의 Detecting Incompatible Use of Spatial Functions before Upgrading to MySQL 8.0](https://mysqlserverteam.com/detecting-incompatible-use-of-spatial-functions-before-upgrading-to-mysql-8-0/)