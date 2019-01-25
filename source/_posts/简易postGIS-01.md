---
title: '''如何用sql实现地理坐标之间的距离查询、周边查询及范围查询？'''
date: 2019-01-25 18:16:24
tags:
---

# 如何用sql实现地理坐标之间的距离查询、周边查询及范围查询？


## 基于postgresql的几何空间计算
postgresql提供了非常便捷的空间关系语法，常用的包括：
A与B的距离用“A <-> B”，
判断A在B之内用“A <@ B”
其他包括平移、缩放、相接、上下、左右、平行、中心点、最近点等各种你能想到的空间关系，感觉非常适合用来支撑图形计算。具体参见：https://www.postgresql.org/docs/9.6/functions-geometry.html

（1）计算两个点之间的距离：
点A：point(4, 3)
点B：point(10, 32)
sql如下：
select point(4, 3) <-> point(10, 32);

（2）判断点A是否在面B之内：
点A：point '(1,1)'
面B：circle '((0,0),2)'
select point '(1,1)' <@ circle '((0,0),2)' 

（3）查找距离线路3之内的所有点

CREATE TABLE "test"."bus_stop" (
  "id" int4 NOT NULL,
  "bus_stop" point,
  "stop_name" varchar(255) COLLATE "pg_catalog"."default"
)
;

CREATE TABLE "test"."walk_line" (
  "id" int4 NOT NULL,
  "walk_line" path,
  "line_name" varchar(255) COLLATE "pg_catalog"."default"
)
;

SELECT 
a.stop_name,
b.line_name,
a.bus_stop <-> b.walk_line as dist
from bus_stop as a,walk_line as b
where a.bus_stop <-> b.walk_line < 3

美中不足的是，postgresql只支持几何类型，不支持地理类型，也就是不支持经纬度。要支持经纬度，需要postgis插件的支持。

## 基于postgis的地理空间计算

在postgis中有两种主要的数据类型，geometry和geography，常规功能优先使用geography类型。geometry是经典数据类型，功能非常强大，geography是postgis后来推出的数据类型，使用方便。两者之间的具体区别可见功能矩阵文档。

建表：
CREATE TABLE ptgeogwgs(gid serial PRIMARY KEY, geog geography(POINT) );
INSERT INTO ptgeogwgs (gid, geog) VALUES (1,ST_GeogFromText('POINT(120.787 30.122)'));
INSERT INTO ptgeogwgs (gid, geog) VALUES (2,ST_GeogFromText('POINT(121.101 31.508)'));
SELECT gid,geog,st_astext(geog) from ptgeogwgs;

点之间距离查询：
SELECT st_distance(p1.geog, p2.geog) from ptgeogwgs as p1,ptgeogwgs as p2 where p1.gid>p2.gid ;

周边查询：
SELECT p1.gid,p2.gid from ptgeogwgs as p1,ptgeogwgs as p2 where p1.gid>p2.gid
and st_dwithin(p1.geog, p2.geog, 1000000)；

点与线之间距离及周边查询：
create table linegeowgs (gid serial PRIMARY KEY, geog geography(LINESTRING));
INSERT INTO linegeowgs (gid, geog) VALUES (1,ST_GeogFromText('LINESTRING(121.11 31.22,121.22 31.13,121.33 31.433)'));
SELECT st_distance(p1.geog, p2.geog) from ptgeogwgs as p1,linegeowgs as p2
where st_dwithin(p1.geog,p2.geog,24000)

相关概念的介绍：
点、线、面一般利用ST_GeogFromText从WKT(well known text)创建，常见的WKT有：
- POINT(0 0) ——点
- LINESTRING(0 0,1 1,1 2) ——线
- POLYGON((0 0,4 0,4 4,0 4,0 0),(1 1, 2 1, 2 2, 1 2,1 1)) ——面

距离计算有多种方式，对geography来说直接使用st_distance可以满足需求，细节区别参见官方文档

详细文档参见：https://postgis.net/docs/reference.html

## postgis的安装
最便捷的安装方法莫过于docker：
docker pull mdillon/postgis:9.6
docker run --name postgis -p 5432:5432 -e POSTGRES_PASSWORD=12345 -d mdillon/postgis:9.6

安装好直接可用，不需要再create extension postgis


