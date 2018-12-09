---
layout: post
title: MySQL 의 Geospatial Index (1)
tags:
  - mysql
  - geospatial
---

[MySQL InnoDB 스토리지 엔진은 버전 5.7.5 부터 geospatial index 를 지원합니다.](https://dev.mysql.com/doc/relnotes/mysql/5.7/en/news-5-7-5.html) geospatial index 를 사용하고, where 절의 조건을 적절히 설정하면 쿼리 속도를 높힐 수 있습니다. 간단한 위도/경도 샘플 데이터를 생성해서 테이블을 만들고, 특정 위치로 부터 특정 거리 이내에 존재하는 레코드의 개수를 세는 간단한 쿼리에 대해 성능이 어떻게 개선되는지 확인해보겠습니다.

# 샘플 데이터 테이블 준비

아래의 DDL 로 `place` 테이블을 생성합니다. 2차원 직교 좌표평면 위의 점을 나타내는 **point** 타입의 `location` 컬럼을 선언하고, `location` 컬럼에 대한 **spatial index** 를 선언합니다. 스토리지 엔진으로 InnoDB 를 사용합니다.

```sql
create table place (
	id int,
	location point not null,
	primary key (id),
	spatial index (location)
)
```

<br>
간단한 python script 로 mysql script 파일을 생성합니다. 양재역과 청담역을 대각선 꼭지점으로 하는 직사각형 내부의 위도/경도를 랜덤하게 50000개 생성하고, 테이블에 insert 하는 sql 파일을 만드는 python script 입니다.

```python
import random

with open("place.sql", "w") as file:
    file.write("insert into place (id, location)\n")
    file.write("values\n")
    for id in range(50000):
        # 청담역 127.0479813, 37.5188072
        # 양재역 127.0317433, 37.4845142
        lng = random.uniform(127.0317433, 127.0479813).
        lat = random.uniform(37.4845142, 37.5188072)
        file.write(f"({id}, point({lng}, {lat})),\n")
    lng = random.uniform(127.0317433, 127.0479813)
    lat = random.uniform(37.4845142, 37.5188072)
    file.write(f"({id + 1}, point({lng}, {lat}));")

# insert into place (id, location)
# values
# (0, point(127.04298707366922, 37.49216541029978)),
# (1, point(127.03177502235926, 37.484975063227104)),
# ...
# (9999, point(127.042389103822, 37.498510910299245)),
# (9999, point(127.03440857599458, 37.512764805693074));
```

<br>
mysql shell 로 sql 을 실행해서 데이터를 insert 합니다.

```
$ mysql -h {db_endpoint} -u {user_name} --password

mysql> source {path_to_place.sql}
Query OK, 50001 rows affected (1.90 sec)
Records: 50001  Duplicates: 0  Warnings: 0
```

# 쿼리 속도 비교

**선릉역** (127.048995, 37.504506) 과 거리가 200m 미만인 레코드의 개수를 세는 쿼리를 기준으로 삼아봅니다. 거리는 [ST\_Distance\_Sphere](https://dev.mysql.com/doc/refman/5.7/en/spatial-convenience-functions.html#function_st-distance-sphere) 함수를 사용해서 **minimum spherical distance** 를 계산합니다. 실습 환경은 AWS RDS t2.micro / MySQL DBMS 5.7.23 입니다.

먼저 spatial index 를 사용하지 않은 경우입니다. 모든 레코드에 대해 기준점과 거리를 계산해서 200m 미만인 경우를 count 합니다.

```
mysql> select count(ST_Distance_Sphere(location, point(127.048995, 37.504506)) < 200 or null) from place;
+---------------------------------------------------------------------------------+
| count(ST_Distance_Sphere(location, point(127.048995, 37.504506)) < 200 or null) |
+---------------------------------------------------------------------------------+
|                                                                             253 |
+---------------------------------------------------------------------------------+
1 row in set (0.06 sec)
```

<br>
다음으로 spatial index 를 사용하기 위해 where 절에 [Within](https://dev.mysql.com/doc/refman/5.7/en/spatial-relation-functions-mbr.html#function_within) 를 사용해서 적당한 boundary 를 잡아줍니다. MySQL 5.7 버전의 `point` 타입은 2차원 직교 좌표평면 위의 점이라는 사실을 상기합니다. 따라서 기준점으로 부터 동서남북 사방으로 200m 떨어진 위도/경도 점을 계산한 다음에 이를 아우르는 직사각형을 boundary 로 잡으면 됩니다. python 의 `geopy` 패키지의 도움을 받아 기준점으로부터 특정 방향으로 특정 거리만큼 떨어진 위도/경도 점을 계산합니다.

```python
# pip install geopy

import geopy
from geopy.distance import VincentyDistance

origin = geopy.Point(37.504506, 127.048995). # 선릉역

VincentyDistance(kilometers=0.2).destination(origin, 0.0)  # 북쪽 Point(37.50630801059796, 127.048995, 0.0)
VincentyDistance(kilometers=0.2).destination(origin, 90.0)  # 동쪽 Point(37.50450597834414, 127.05125692835642, 0.0)
VincentyDistance(kilometers=0.2).destination(origin, 180.0). # 남쪽 Point(37.502703988850925, 127.048995, 0.0)
VincentyDistance(kilometers=0.2).destination(origin, 270.0). # 서쪽 Point(37.50450597834414, 127.04673307164359, 0.0)
```

<br>
왼쪽 아래 point 와 오른쪽 위 point 로 **LINESTRING** 을 만들고, [Envelope](https://dev.mysql.com/doc/refman/5.7/en/gis-general-property-functions.html#function_envelope) 함수를 써서 **POLYGON** 으로 만들어 where 절에 사용합니다.

* 왼쪽 아래 point = (최소 X, 최소 Y) = (서쪽 경도, 남쪽 위도) = (127.04673307164359, 37.502703988850925)
* 오른쪽 위 point = (최대 X, 최대 Y) = (동쪽 경도, 북쪽 위도) = (127.05125692835642, 37.50630801059796)

```
mysql> select count(ST_Distance_Sphere(location, point(127.048995, 37.504506)) < 200 or null) from place where Within(location, Envelope(GeomFromText("LINESTRING(127.04673307164359 37.502703988850925, 127.05125692835642 37.50630801059796)")));
+---------------------------------------------------------------------------------+
| count(ST_Distance_Sphere(location, point(127.048995, 37.504506)) < 200 or null) |
+---------------------------------------------------------------------------------+
|                                                                             253 |
+---------------------------------------------------------------------------------+
1 row in set, 3 warnings (0.01 sec)
```

<br>
spatial index 를 사용한 경우 소요시간이 약 **10ms** 로, 사용하지 않은 경우의 **60ms** 보다 짧아지는 것을 확인할 수 있습니다. **explain** 로 확인해보니, 전체 테이블을 처리하는 대신 spatial index 의 도움을 받아 약 251개의 일부 레코드만 처리하는 것을 알 수 있습니다.

```
mysql> explain select count(ST_Distance_Sphere(location, point(127.048995, 37.504506)) < 200 or null) from place;
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows  | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------+
|  1 | SIMPLE      | place | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 48968 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+------+-------+----------+-------+
1 row in set, 1 warning (0.01 sec)

mysql> explain select count(ST_Distance_Sphere(location, point(127.048995, 37.504506)) < 200 or null) from place where Within(location, Envelope(GeomFromText("LINESTRING(127.04673307164359 37.502703988850925, 127.05125692835642 37.50630801059796)")));
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | place | NULL       | range | location      | location | 34      | NULL |  251 |   100.00 | Using where |
+----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
1 row in set, 4 warnings (0.01 sec)
```

# 마치며

MySQL InnoDB 스토리지 엔진 5.7.5 부터 지원하는 geospatial index 를 활용해서 쿼리 속도를 높히는 방법에 대해 알아봤습니다. 다음 글에서는 이와 관련하여 MySQL 8.0 으로 업그레이드 되면서 바뀐 부분에 대해서 알아보도록 하겠습니다.

# 레퍼런스

* [MySQL 5.7 Reference Manual 11.5.2.3 Point Class](https://dev.mysql.com/doc/refman/5.7/en/gis-class-point.html)
* [MySQL 5.7 Reference Manual 11.5.9 Creating Spatial Indexes](https://dev.mysql.com/doc/refman/5.7/en/creating-spatial-indexes.html)
* [MySQL 5.7 Reference Manual 12.16.12 Spatial Convenience Functions](https://dev.mysql.com/doc/refman/5.7/en/spatial-convenience-functions.html#function_st-distance-sphere)
* [MySQL 5.7 Reference Manual 12.16.9.2 Spatial Relation Functions That Use Minimum Bounding Rectangles](https://dev.mysql.com/doc/refman/5.7/en/spatial-relation-functions-mbr.html#function_within)
* [MySQL 5.7 Reference Manual 12.16.7.1 General Geometry Property Functions](https://dev.mysql.com/doc/refman/5.7/en/gis-general-property-functions.html#function_envelope)
