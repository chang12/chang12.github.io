---
layout: post
title: Python 으로 지도 위에 정사각 격자를 그리려면?
tags: [Geography]
---

Python 으로 지도 위에 정사각 격자를 그리려고 합니다. 정확히는 서울시를 1km x 1km 크기의 정사각 격자로 나누려고 합니다.

## 어떤 패키지를 사용할까?

우선 jupyter notebook 환경에서 그리고 싶었습니다. "jupyter map visualization" 정도의 키워드로 구글링을 해보니, 뭔가 jupyter 쪽 공식 사이트 처럼 보이는 [widget 소개 페이지가 있고](https://jupyter.org/widgets), 거기에 지도 관련하여 [ipyleaflet](https://github.com/jupyter-widgets/ipyleaflet) 과 [gmaps](https://github.com/pbugnion/gmaps) 가 있습니다. 좀 더 찾아보니 [folium](https://github.com/python-visualization/folium) 도 있습니다. 작성 시점에 3762 개로 GitHub Stars 가 가장 많은 folium 을 선택합니다.

## 격자를 그리려면?

"folium rectangle grid" 정도의 키워드로 구글링을 해서 가장 위에 뜬 [Analysing Geographic Data with Folium](https://www.jpytr.com/post/analysinggeographicdatawithfolium/) 글에서 지도 위에 격자를 그렸습니다. jupyter notebook 으로 코드를 그대로 옮기고 좌표만 바꿔보니 잘 그려집니다.

![pic1-copy-and-paste-to-seoul-grid.jpg](/images/2019-03-23-pic1-copy-and-paste-to-seoul-grid.png)

위의 글에서는 좌하단/우상단 (위도, 경도) 를 받아서 각도를 등간격으로 나눠 격자를 그렸습니다. 이를 직교 좌표계에서 원하는 미터 간격으로 그리도록 수정하고 싶었습니다.

## 어느 직교 좌표계를 사용할까?

뭔가 공식적인 사이트처럼 보이는 https://epsg.io/ 에 접속해서 "korea" 로 검색해서 나오는 여러 CRS 중에서 [Korea 2000 / Central Belt 2010 (EPSG:5186)](https://epsg.io/5186) 을 선택했습니다. 최신이고, 여러 Belt 들이 있는데 그 중 Central 이 낫지 않을까 싶었고, 상세 페이지에 `Legally mandated CRS from 2010-01-01.` 라고 적혀있는 게 마음에 들었습니다.

## CRS 간의 좌표 변환은 어떻게 할까?

[folium GitHub 의 issue #481](https://github.com/python-visualization/folium/issues/481) 를 보면 `folium.Map` 은 EPSG:4326 (= 위도/경도) 의 좌표만 처리할 수 있는 듯 합니다. 따라서 격자 계산은 EPSG:5186 에서 하더라도 최종적으로 EPSG:4326 좌표로 변환해줘야 합니다. [Stack Exchange 의 Converting projected coordinates to lat/lon using Python? 의 답변](https://gis.stackexchange.com/a/78944) 을 보니 `pyproj` 패키지를 쓰면 되겠습니다.

```python
from pyproj import Proj, transform

proj_4326 = Proj(init='epsg:4326')
proj_5186 = Proj(init='epsg:5186')

p_4326 = (126.738394, 37.425886)
p_x_5186, p_y_5186 = transform(proj_4326, proj_5186, p_4326[0], p_4326[1]) # (176844.5078359241, 536310.6085672465)
```

EPSG:5186 은 미터 단위 좌표계이므로, 원하는 미터 간격의 격자를 그릴 수 있게 되었습니다.

```python
n = 50 # 50x50
meters = 1000 # 1km

for x, y in itertools.product(range(n), range(n)):
    grid_lower_l = transform(proj_5186, proj_4326, p_x_5186 + (x + 0) * meters, p_y_5186 + (y + 0) * meters)
    grid_lower_r = transform(proj_5186, proj_4326, p_x_5186 + (x + 1) * meters, p_y_5186 + (y + 0) * meters)
    grid_upper_r = transform(proj_5186, proj_4326, p_x_5186 + (x + 1) * meters, p_y_5186 + (y + 1) * meters)
    grid_upper_l = transform(proj_5186, proj_4326, p_x_5186 + (x + 0) * meters, p_y_5186 + (y + 1) * meters)
```

## 시간이 너무 오래 걸리는데...?

1km 간격으로 50x50 격자를 그리는데 4-5분이 걸렸습니다. 추후 목표하는 바를 이루기 위해서는 수천장을 그려야 하는데 소요 시간이 너무 길다고 느껴집니다. 다행히 pyproj 문서 중 [Optimize Transformations](http://pyproj4.github.io/pyproj/html/optimize_transformations.html) 를 읽고 해소할 수 있었습니다. CRS 간의 변환을 위한 transformer 를 만드는 게 비싼 연산이라 매 변환때 마다 만드는게 부담됬나 봅니다.

```python
from pyproj import Transformer

n = 50 # 50x50
meters = 1000 # 1km

transformer_5186_4326 = Transformer.from_proj(5186, 4326)

for x, y in itertools.product(range(n), range(n)):
    grid_lower_l = transformer_5186_4326.transform(p_x_5186 + (x + 0) * meters, p_y_5186 + (y + 0) * meters)
    grid_lower_r = transformer_5186_4326.transform(p_x_5186 + (x + 1) * meters, p_y_5186 + (y + 0) * meters)
    grid_upper_r = transformer_5186_4326.transform(p_x_5186 + (x + 1) * meters, p_y_5186 + (y + 1) * meters)
    grid_upper_l = transformer_5186_4326.transform(p_x_5186 + (x + 0) * meters, p_y_5186 + (y + 1) * meters)
```

본래 4-5분 걸리던 코드의 소요 시간이 2초로 줄어들었습니다.

## 전체 코드와 결과

```python
import itertools

import folium
from pyproj import Transformer, transform

transformer_5186_4326 = Transformer.from_proj(5186, 4326)

p_4326 = (126.738394, 37.425886) # 구글 지도 보고 적당히 고른 서남단 좌표.
meters = 1000 # 1km
n = 50 # 50x50

p_x_5186, p_y_5186 = transform(4326, 5186, p_4326[0], p_4326[1])

geo_jsons = []

for x, y in itertools.product(range(n), range(n)):
    grid_lower_l = transformer_5186_4326.transform(p_x_5186 + (x + 0) * meters, p_y_5186 + (y + 0) * meters)
    grid_lower_r = transformer_5186_4326.transform(p_x_5186 + (x + 1) * meters, p_y_5186 + (y + 0) * meters)
    grid_upper_r = transformer_5186_4326.transform(p_x_5186 + (x + 1) * meters, p_y_5186 + (y + 1) * meters)
    grid_upper_l = transformer_5186_4326.transform(p_x_5186 + (x + 0) * meters, p_y_5186 + (y + 1) * meters)

    geo_json = {
        "type": "FeatureCollection",
        "features": [
            {
                "type": "Feature",
                "geometry": {
                    "type": "Polygon",
                    "coordinates": [
                        [
                            grid_lower_l,
                            grid_lower_r,
                            grid_upper_r,
                            grid_upper_l,
                            grid_lower_l
                        ]
                    ]
                }
            }
        ]
    }

    geo_jsons.append(geo_json)

m = folium.Map(location=(p_4326[1], p_4326[0]), zoom_start=11) # folium.Map 은 (lat, lng) 로 location 을 받음에 유의!
for i, geo_json in enumerate(geo_jsons):
    gj = folium.GeoJson(geo_json)
    m.add_child(gj)
m.save('index.html')
```

![pic2-seoul-1km-50x50-grid.jpg](/images/2019-03-25-pic2-seoul-1km-50x50-grid.jpg)

## 레퍼런스

* [Analysing Geographic Data with Folium](https://www.jpytr.com/post/analysinggeographicdatawithfolium/)
* [Stack Exchange 의 Converting projected coordinates to lat/lon using Python? 의 답변](https://gis.stackexchange.com/a/78944)
* [pyproj 2.1.2 documentation 중 Optimize Transformations](http://pyproj4.github.io/pyproj/html/optimize_transformations.html)
