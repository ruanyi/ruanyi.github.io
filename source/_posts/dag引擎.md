---
title: dag引擎
abbrlink: 394f6975
date: 2024-02-27 21:10:21
tags: dag、Serverless Workflow、并发调度
categories: 媒体处理
---

# 背景

在实际媒体处理的业务场景中，复杂业务逻辑往往涉及多个步骤、多个模块、多个原子化动作，组合、协同工作，才能顺利完成。而任务编排通过将多个函数组织成有序的执行流程，使得开发者能够更自然地表达和管理复杂的业务逻辑。这种能力，业界称之为：工作流(
Workflow）or dag引擎（本文无特殊说明，用dag引擎代表）。举些例子：

## 图片计算流程图

![](/images/thumb_img_meta.png)

## 切片加速流程图

![](/images/slice_dag_1.png)

## 转码流程图

![](/images/tc_dag_2.png)

# 需求分析

## 数据流向 -> 任务编排

从数据流向来看，分为：

1、直线型pipeline：一个步骤的输出直接传递给下一个步骤的输入。

2、dag型pipeline：多个步骤的输出，当做下一个步骤的输入 + 一个步骤的输出，当做多个步骤的输入。

## 同步 or 异步

在实际的媒体处理业务场景中，由于计算耗时长的原因，一般很少使用：dag引擎来编排同步处理业务。大部分都是走：异步回调的模式。因此，所实现的任务编排，需支持：异步回调的风格。

## 异步回调

一般异步回调，都会有：notify_url字段 + 唯一id存在，建议：业务方不强依赖于：唯一id，而是通过：notify_url里带上业务自身的唯一id来处理。

例如：notify_url：http://xxxxx.xxxx.com/api/v1/media/notify?id=xxxx

注意：唯一id，是相对而言的，最好由各自确保各自的id唯一，并不由别人来确保自身id的唯一。

## 并行调度

如文档：[视频切片并行加速](/posts/4d3edad6.html)所述视频处理加速方法之一：切片并行加速，它其实也是一种任务编排，因此，它可由DAG引擎去编排整个任务。举个例子：
![](/images/slice_dag_1.png)

1、视频meta信息提取 与 音频提取，它们2个是并行处理，可事先画出来。
2、并行切片与并行gpu处理，里面处理视频的个数，无法事先画出来，只能用虚线来表示。
3、视频切片并行加速，重：并行调度，弱编排。

## 执行时间

基于成本的考量，部分任务，业务侧虽然已经触发，但是，可允许“24”个小时再回调，因此，节点的执行，分为：准实时执行 + 延时执行。

## 重试机制

如之前描述：由于媒体处理，重cpu or gpu计算，非常容易卡死 + 不涉及事务，又由于cpu计算 or gpu计算执行多次，不改变结果，因此，可设计重试机制：

严格意义上讲：只存在：节点重试，并没有任务重试，对于业务方而言，任务重试，相当于重新触发新的任务，只是再通知业务方时，需使用同一个：唯一键，来告知业务方结果。_

 重试类型 | 说明                                     | 备注 |
------|----------------------------------------|----|
 超时重试 | 超过一定时间后，重新触发执行，一般情况下，由CB体系触发           |    |     
 失败重试 | 失败后，自动重试，支持：代码自动重试 + cb体系触发重试 + 手动触发重试 |    |

# cb体系

如：重试机制上描述的一样，换个角度再次审视下：由于中间各个步骤，包括了大量的：cpu or
gpu计算服务，非常容易卡死、奔溃，导致整体任务长时间未收到回调，因此，需要check bill
system，定时、延时触发check任务状态，假设超过一定的时间阈值，重试
or 设置整个任务为失败。流程图如下：

![](/images/dag_cb_1.png)

## 业界情况

 业界产品       | 作用                                                          | 功能说明 | 备注 |
------------|-------------------------------------------------------------|------|----|
 Celery     | 严格意义上讲：它不应该是是dag引擎，它更多的是：分布式任务队列系统，用于处理异步任务和消息传递            |      |    | 
 Dask       | 严格意义上讲：它也不是：dag引擎，Dask 是一个用于并行计算的灵活工具，特别适合处理大规模数据和分布式计算     |      |    |
 osworkflow | 一个轻量化的流程引擎                                                  |      |    |
 activiti   | 由alfresco软件开发，目前最高版本activiti 7                              |      |    |
 camunda    | Camunda基于activiti5，所以其保留了PVM，最新版本Camunda7.15，保持每年发布2个小版本的节奏 |      |    |

备注：
1、对于：媒体处理而言，任务编排与并行调度放在一起实现，是更合理的一种方式。

# 概要设计

设计一款满足：媒体处理场景下的工作流引擎，包括以下模块：
![](/images/dag_1.png)

## 节点设计

 节点类型   | 作用                               | 执行次数           | 备注 |
--------|----------------------------------|----------------|----|
 开始节点   | 执行dag引擎的开始动作                     | 执行一次           |    | 
 结束节点   | 执行dag引擎后的结束动作                    | 执行N次           |    |
 路由节点   | 上一个节点执行完后，根据一定的路由执行下一个节点         |                |    |
 同步脚本节点 | 同步等待处理结果，短时间之内即可处理完，因此，同步等待结果    |                |    |
 异步脚本节点 | 处理时长太长，无法同步等待，后续回调，找到对应的执行节点继续处理 |                |    |
 并行节点   | 某个节点，有多个输出结果，下一个节点需并行处理          |                |    |
 子业务流节点 | 某个节点，有多个输出结果，下一个节点需并行处理          |                |    |
 回调节点   | 回调通知业务方处理结果                      | 就算没有执行到，也得执行一次 |    |

备注：
1、回调节点，当需要支持：多种类型的回调时使用，但是：接口协议中，也需要设计回调，也就是说：当整个dag引擎检测到：卡死在某个接口中时，触发协议中的回调接口。

## 任务追踪

 名词      | 解释 | 关系                            | 说明                    | 备注                                   |
---------|----|-------------------------------|-----------------------|--------------------------------------|
 task    | 任务 | 一次请求，对应一笔任务                   | 一笔切片请求，对应一笔任务         | 可使用UUID生成唯一的：task_id                 |
 event   | 事件 | task 与event 是一对多的关系           | 一个dag节点，对应一个事件        | event_id = task_id + "_" + node_name |
 segment | 片  | event 与 segment 是一对多的关系       | 一个视频片，对应一个event       | segment_id = event_id + "_" + order  |
 trace   | 跟踪 | trace 与 event、segment 是一对多的关系 | 一次cpu、gpu计算，对应一次trace | 可使用UUID生成唯一的：trace_id                |

## 执行计划优化

业务dag与代码执行dag往往有差异。因此，需要对：dag进行剪枝。

# 库表设计

 表名        | 作用                         | id生成策略                               | 备注 |
-----------|----------------------------|--------------------------------------|----|
 Pipeline表 | 描述业务所期许的：步骤集 + 流程          |                                      |    |
 task表     | 记录一次任务的执行过程，记录任务的请求参数 + 结果 | 可使用UUID生成唯一的：task_id                 |    |
 event表    | 记录节点事件的执行过程                | event_id = task_id + "_" + node_name |    |
 segment表  | 记录节点分片的执行过程                | segment_id = event_id + "_" + order  |    |
 trace表    | 记录所有cpu、gpu计算的入参 + 出参      | 可使用UUID生成唯一的：trace_id，也可由第三方计算服务生成   |    |

再次说明：
1、task + event + segment + trace，记录的是dag引擎的执行过程中的临时数据，执行完毕后，除了问题分析外，作用不太大。
2、不一定：非得使用mysql来存储数据，也可以采用：redis来存储数据。
3、根据task_id 和一些业务信息，完全可以计算出：event_id和segment_id。

## 数据清理

基于成本的考量，这些数据该如何自动过期呢？

1、若采用mysql存储：则可使用ab库 or 启动一定是任务，定期删除旧数据。
2、若采用redis做流程控制，则使用redis的数据过期机制来处理。

## Pipeline，配置表，采用：文档模型：cloud_process_pipeline

一般情况下，不太可能分库分表。

 字段名            | 类型          | 描述                            |
----------------|-------------|-------------------------------|
 id             | long        | 库表id，自增                       |
 create_time    | datetime    | 创建时间                          |
 modify_time    | datetime    | 修改时间                          |
 config_name    | varchar(64) | 英文名字                          |
 config_version | varchar(12) | 版本号                           |
 last_version   | int(1)      | 0：最新版 <br/> 1、非最新版<br/>       |
 config_meta    | text        | json config                   |
 config_type    | int(1)      | 1:规则引擎 <br/>2:dag引擎 <br/>3、指令 |
 config_status  | int(1)      | 0：正式运行 <br/> 2、草稿<br/>9、废弃 <> |

## 运行时，任务表：cloud_process_task

分库分表id为：task_id

 字段名         | 类型            | 描述                           |
-------------|---------------|------------------------------|
 id          | long          | 库表id，自增                      |
 create_time | datetime      | 创建时间                         |
 modify_time | datetime      | 修改时间                         |
 task_id     | varchar(64)   | 自动生成的UUID，索引，唯一              |
 task_params | text          | 任务请求参数                       |
 config_id   | long          | 配置表                          |
 config_meta | text          | json config                  |
 task_result | text          | 任务结果表                        |
 notify_url  | varchar(2048) | 回调关联信息，强烈建议：对于业务而言，具有唯一性     |
 total_cost  | int(10)	      | 任务处理总耗时                      |
 task_status | char(1)       | 0：成功 <br/>1：正在进行 <br/>9：运行失败 |

备注:

1、任何一个task，建议关联上所执行的pipeline配置，否则，容易出现：pipeline 更改后，对应的task未及时修正的情况。
2、如果需要整个任务重试，则建议：重新生成一个新的task,notify_url不变即可。
3、回调的协议，建议：双方约定好即可。

## 运行时，事件表：cloud_process_event

分库分表id为：task_id

 字段名          | 类型           | 描述                                             |
--------------|--------------|------------------------------------------------|
 id           | long         | 库表id，自增                                        |
 create_time  | datetime     | 创建时间                                           |
 modify_time  | datetime     | 修改时间                                           |
 task_id      | varchar(64)  | 关联：cloud_process_task的task_id                  |
 trace_id     | varchar(64)  | 可为空，不一定有值，若有值，则关联：cloud_process_trace的trace_id |
 event_id     | varchar(128) | event_id = task_id + "_" + node_name           |
 node_name    | varchar(64)  | 节点名称                                           |
 node_type    | int(2)       | 节点类型,参考[节点设计](#节点设计)             。             |
 event_status | char(1)      | 0：成功 <br/>1：正在进行 <br/>9：运行失败                   |
 exec_cnt     | int(10)      | 执行次数，大部分情况下：1次,重试次数 = 执行次数 - 1                 |
 latch_cnt    | int(10)      | 当latch_cnt == in_degree时，当前节点，可继续往下执行          |
 segment_cnt  | int(10)      | 分片数量,0：如果没有分片，则分片数量为：0,有分片，则输入对应的分片数量          |
 event_result | text         | 事件结果表                                          |
 total_cost   | int(10)	     | 事件处理总耗时                                        |

## 运行时，分片表：cloud_process_segment

分库分表id为：task_id

 字段名            | 类型           | 描述                                   |
----------------|--------------|--------------------------------------|
 id             | long         | 库表id，自增                              |
 create_time    | datetime     | 创建时间                                 |
 modify_time    | datetime     | 修改时间                                 |
 task_id        | varchar(64)  | 关联：cloud_process_task的task_id        |
 event_id       | varchar(128) | event_id = task_id + "_" + node_name |
 segment_id     | varchar(256) | segment_id = event_id + "_" + order  |
 trace_id       | varchar(64)  | 关联：cloud_process_trace的trace_id      |
 segment_type   | int(2)       | 正常情况下，跟它所对应的node_type一模一样            |
 exec_cnt       | int(10)      | 执行次数，大部分情况下：1次,重试次数 = 执行次数 - 1       |
 segment_order  | int(10)      | 分片顺序号                                |
 segment_status | int(10)      | 0：成功 <br/>1：正在进行 <br/>9：运行失败         |
 segment_result | text         | 分片结果表                                |
 total_cost     | int(10)	     | 分片处理总耗时                              |

## 运行时，跟踪表：cloud_process_trace

每请求一次，则新增一条记录，分库分表id为：task_id

 字段名             | 类型             | 描述                            |
-----------------|----------------|-------------------------------|
 id              | long           | 库表id，自增                       |
 create_time     | datetime       | 创建时间                          |
 modify_time     | datetime       | 修改时间                          |
 task_id         | varchar(64)    | 关联：cloud_process_task的task_id |
 trace_id        | varchar(128)   | 计算服务的跟踪id，唯一id                |
 cal_type        | int(2)         | 计算服务的类型                       |
 trace_status    | int(10)        | 0：成功 <br/>1：正在进行 <br/>9：运行失败  |
 request_params  | text           | 请求参数                          |
 request_url     | varchar(2048)	 | 请求地址                          |
 response_result | text	          | 响应报文                          |
 callback_result | text	          | 回调报文                          |
 total_cost      | int(10)	       | 分片处理总耗时                       |

备注：

1、虽然将task_id作为分库分表的主键，但是，查询该表时，入参为：task_id、trace_id

# redis key设计

![](/images/dag_redis_key_1.png)

备注：
1、Pipeline，配置表，强烈建议：由mysql来兜底，PS：当然前期，可以不设计该：key。
2、task、event、segment、trace表，本来就是临时数据，方便任务管理、追踪，因此，不建议存在mysql。

## 