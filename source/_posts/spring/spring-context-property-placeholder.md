---
title: spring源码阅读-context:property-placeholder
date: 2017-04-27 10:59:09
tags:
    - spring
categories:
    - spring
thumbnail:
---
![context:property-placeholder](/images/spring/context-property-placeholder.png)
<!-- more -->
~~~
location: 文件路径,多个文件用逗号分割
properties-ref:
file-encoding: 文件编码
order: 多个placeholder configurer顺序
ignore-resource-not-found: 如果属性文件找不到是否忽略，如果找不到会抛出运行时异常，默认不忽略
ignore-unresolvable: 是否忽略解析不到的属性，如果不忽略，找不到将抛出异常
local-override:

~~~
