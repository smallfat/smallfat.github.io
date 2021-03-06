---
title: PostDB - WAL日志空洞管理模块设计
tags: WAL 
category: /小书匠/日记/2021-07
renderNumberedHeading: true
grammar_cjkRuby: true
---


# 功能
### 总体功能

![绘图](./attachments/1626283921116.drawio.html)

### 存储区间列表
- 此列表记录了所有已完成持久化的数据的区间
- 区间以lsn索引，包括上标和下标
- 上标和下标都带有lsn，以及原始record所指向的前一个record的lsn信息
- 在系统初始化时，创建存储区间列表
- 在系统正常退出时，保存存储区间列表至硬盘系统配置
- 在系统recovery时，恢复存储区间列表。具体恢复流程见异常处理
- 每次对buffer进行空洞扫描时，将buffer的区间[first_lsn, last_lsn]合并到存储区间列表
- 每次回放完成后，更新存储区间列表由低至高的第一个区间，使之下标lsn >=NCL (与回放无关系)

### 空洞发现

- 空洞发现算法

![绘图](./attachments/1626064228154.drawio.html)

- 已有空洞与新空洞

### 空洞填充
- 流程

![绘图](./attachments/1626068651977.drawio.html)

- peer查找(数据寻址) 
  - 目前storage node之间采用何种数据同步协议尚未确定，这里假设采用广播
  - 通过向其他peer广播空洞区间，找到包含此数据的peer

- 获取数据
	- 向上述peer发送请求，获取数据
	- 区间信息可以包括page信息，在请求的时候以页为单位，进行优化
	- 一致性与发送请求的时机； < SGDL才能发送？
- NCL判定
	- NCL即存储区间列表由低至高第一个区间的上标lsn

- 超时重传	- 如果获取数据请求超时没有得到回复，则考虑peer disable的可能性；重新进行peer 查找并获取数据

- 填充空洞数据
	- 调用既有接口，将空洞数据填充到日志持久化存储中
	- 填充之前先判断存储区块列表中是否包含相应区间，包含则说明已填充完毕，无需再填充


### 异常处理
###### 主机宕机
主机宕机有可能造成存储区间列表不能正常退出序列化，从而影响重启后正常的功能。因此需要重新构建存储区间列表
- 流程

![绘图](./attachments/1626285222578.drawio.html)

- 从持久化日志存储中重构存储区间列表
  - 从NCL开始读取日志record，一直读到最新(???遇到最后一条记录就不读了)
  - 判断record->prev是否与前一个record的lsn相等。若不相等，则存在空洞
  - 最终重构出与当前日志相匹配的存储区间列表
  - recovery时，系统会发送一个truncate命令给我，我需要提供一个接口，把lsn后面的区间全部清除

###### 网络故障
- 本模块在进行空洞填充时会通过网络发送接受日志数据
- 发生网络故障时，比如网络中断/网络超时等，目前会延迟空洞填充补齐的时间，并不影响系统的功能性和健壮性。




