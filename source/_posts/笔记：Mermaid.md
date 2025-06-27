---
title: 笔记：Mermaid
date: 2025-03-19
categories:
  - Mermaid
tags: 
author: 霸天
layout: post
---
```
graph TD
    subgraph 客户端
	    A[用户请求] -->|"DNS 解析"| B["NGINX"]
    end

    subgraph 前端
	    B --> C["NGINX1(主)"]
	    B --> D["NGINX2"]
	    B --> E["NGINX3"]
    end

    subgraph 后端
	    C --> F["Redis集群"]
	    D --> F["Redis集群"]
	    E --> F["Redis集群"]
	    F --> G["Redis1(主)"]
	    F --> H["Redis2"]
	
	    C --> I["Kafka集群"]
	    D --> I["Kafka集群"]
	    E --> I["Kafka集群"]
	    
	    I --> J["Kafka1(主)"]
	    I --> K["Kafka2"]
    end
```
```mermaid
graph TD
    subgraph 客户端
	    A[用户请求] -->|"DNS 解析"| B["NGINX"]
    end

    subgraph 前端
	    B --> C["NGINX1(主)"]
	    B --> D["NGINX2"]
	    B --> E["NGINX3"]
    end

    subgraph 后端
	    C --> F["Redis集群"]
	    D --> F["Redis集群"]
	    E --> F["Redis集群"]
	    F --> G["Redis1(主)"]
	    F --> H["Redis2"]
	
	    C --> I["Kafka集群"]
	    D --> I["Kafka集群"]
	    E --> I["Kafka集群"]
	    
	    I --> J["Kafka1(主)"]
	    I --> K["Kafka2"]
    end
```




```
graph TD
    A[(0,0)] -->|选物品1| B[(2,3)]
    A -->|不选物品1| C[(0,0)]
    B -->|选物品2| D[(5,7)]
    B -->|不选物品2| E[(2,3)]
    C -->|选物品2| F[(3,4)]
    C -->|不选物品2| G[(0,0)]
    D -->|选物品3| H[(9,14)]
    D -->|不选物品3| I[(5,7)]
    E -->|选物品3| J[(6,10)]
    E -->|不选物品3| K[(2,3)]
    F -->|选物品3| L[(7,11)]
    F -->|不选物品3| M[(3,4)]
    G -->|选物品3| N[(4,7)]
    G -->|不选物品3| O[(0,0)]

```



![](image-20250423144408338.png)





```
graph TD
    A((Start)) --> B((Process))

```

```mermaid
graph TD
    A((Start)) --> B((Process))

```





```mermaid
graph TD
    A((0,0)) -->|x1 = 1| B((5,4))
    A -->|x1 = 0| C((0,0))
    B -->|x2 = 1| D((8,8))
    B -->|x2 = 0| E((5,4))
    C -->|x2 = 1| F((3,4))
    C -->|x2 = 0| G((0,0))
    D -->|x3 = 1| H((10,11))
    D -->|x3 = 0| I((8,8))
    E -->|x3 = 1| J((7,7))
    E -->|x3 = 0| K((5,4))
    F -->|x3 = 1| L((5,7))
    F -->|x3 = 0| M((3,4))
    G -->|x3 = 1| N((2,3))
    G -->|x3 = 0| O((0,0))
    H -->|x4 = 1| P((11,12))
    H -->|x4 = 0| Q((10,11))
    I -->|x4 = 1| R((9,9))
    I -->|x4 = 0| S((8,8))
    J -->|x4 = 1| T((8,8))
    J -->|x4 = 0| U((7,7))
    K -->|x4 = 1| V((6,5))
    K -->|x4 = 0| W((5,4))
    L -->|x4 = 1| X((6,8))
    L -->|x4 = 0| Y((5,7))
    M -->|x4 = 1| Z((4,5))
    M -->|x4 = 0| AA((3,4))
    N -->|x4 = 1| AB((3,4))
    N -->|x4 = 0| AC((2,3))
    O -->|x4 = 1| AD((1,1))
    O -->|x4 = 0| AE((0,0))

```















