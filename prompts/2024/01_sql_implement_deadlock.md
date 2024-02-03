## MySQL 架构图

MySQL 架构图包含组件如下：

- 包含三个大组件：客户端 client，server 层，engine 层，每个大组件分别框起来
- server 层包含：连接器、查询缓存、分析器、优化器、执行器
- engine 层包含：InnoDB、MyISAM、Memory 等多种并列

组件连接关系如下：

- client 请求 连接器
- 连接器先去查询缓存查询，若有缓存直接返回；否则请求分析器
- 分析器请求优化器
- 优化器请求执行器
- 执行器请求 engine 层

@startuml
!theme toy
rectangle "Client" as client
rectangle "Server" as server {
rectangle "连接器" as connector
rectangle "查询缓存" as queryCache
rectangle "分析器" as analyzer
rectangle "优化器" as optimizer
rectangle "执行器" as executor
}
rectangle "Engine" as storageEngine {
rectangle "InnoDB" as innodb
rectangle "MyISAM" as myisam
rectangle "Memory" as memory
}

client -r-> connector
connector -r-> queryCache
connector -d-> analyzer : 未命中
analyzer -r-> optimizer
optimizer -r-> executor
executor -r-> storageEngine
@enduml

## binlog 写磁盘过程

用 plainUML 画一下 MySQL binlog 写磁盘过程的组件图。

包含三大组件：MySQL，Page Cache，磁盘；MySQL 又包含两个 Thread，每个 Thread 都有各自的 binlog cache。

每个 binlog cache 调用 write 方法写入 Page Cache，Page Cache 调用 fsync 方法写入磁盘。

## update 语句执行流程图

用 plainUML 画一下 MySQL update 语句执行流程图，流程如下：

1、读取 user_id=1 一行
2、数据页在内存中？
- 是，则直接走第 3 步
- 否，则从磁盘读入内存，再走第 3 步
3、返回该行数据
4、把该行 money 字段加 100
5、写入新值
6、把新值更新到内存
7、写 redo log prepare 状态
8、写 binlog
9、更新 redo log commit 状态

其中 1、4、5、8 是 Server 层执行器执行的，其他步骤都是由 InnoDB 引擎执行的，两个不同组件执行的步骤用不同颜色区分
