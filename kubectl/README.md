# Introduction

本节从创建一个pod 的命令发起, 大致分析一下kubectl 的执行流程。分析过程中，将会看到kubectl 中几个重要的概念

1. Builder结构体
2. 各种Visitor结构体，及结构体中实现的Visit 方法
3. RESTMapper 接口及各种RESTMapper 的实现
4. 包装yaml 数据的info 数据结构