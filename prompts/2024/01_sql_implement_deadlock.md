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
