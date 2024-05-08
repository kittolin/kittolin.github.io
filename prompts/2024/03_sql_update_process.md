## binlog 写磁盘过程

用 plantUML 画一下 MySQL binlog 写磁盘过程的组件图。

包含三大组件：MySQL，Page Cache，磁盘；MySQL 又包含两个 Thread，每个 Thread 都有各自的 binlog cache。

每个 binlog cache 调用 write 方法写入 Page Cache，Page Cache 调用 fsync 方法写入磁盘。

@startuml
package "MySQL" {
  component "Thread 1" as thread1 {
    component "binlog cache" as binlogCache1
  }
  component "Thread 2" as thread2 {
    component "binlog cache" as binlogCache2
  }
}

component "Page Cache" as pageCache

database "磁盘" as disk

binlogCache1 --> pageCache : write()
binlogCache2 --> pageCache : write()

pageCache --> disk : fsync()
@enduml

## update 语句执行流程图

用 plantUML 画一下 MySQL update 语句执行流程图，流程如下：

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

@startuml
:<color:green>读取 user_id=1 一行;

if (<color:blue>数据页在内存中?) then (是)
else (否)
  :<color:blue>从磁盘中读入内存;
endif

:<color:blue>返回该行数据;
:<color:green>把该行 money 字段加 100;
:<color:green>写入新值;
:<color:blue>把新值更新到内存;
:<color:blue>写 redo log prepare 状态;
:<color:green>写 binlog;
:<color:blue>更新 redo log commit 状态;
@enduml
