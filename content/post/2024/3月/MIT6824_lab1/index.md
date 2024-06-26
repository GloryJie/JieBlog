---

title: "MIT6824实验1-MapReduce设计"
slug: "MIT6824_lab1"
description:
date: "2024-03-24T21:37:07+08:00"
lastmod: "2024-03-24T21:37:07+08:00"
image: cover.png
math:
license:
hidden: false
draft: false
categories: 
    - 课程学习
tags:
    - MIT6.824
    - 分布式

---

> 封面是《斗罗大陆2绝世唐门》中的凌落宸，用冰技能, 图片由AI生成，完成图片放在附录中


课程首页以及实验说明相关介绍链接在附录中。


## 课程理解


简单的说，就是用Go来实现一个单机多进程版本的MapReduce。(MapReduce计算模型可以去搜索相关文章了解)。会有如下两种类型的进程

- Coordinator协调者：作为一个协调着，协调Map、Reduce的任务交给不同的Worker去执行，协调Map、Reduce阶段（执行完所有Map任务之后才可以去执行Reduce任务）
- Worker执行器：任务执行者，执行Map、Reduce任务

起来一个MR任务后，只会有一个Coordinator进程，可以有多个Worker进程。需要去考虑和解决下面的几个问题

- Coordinator如何感知worker？
- 所有Map任务执行完成后，方可进行Reduce任务，Coordinator如何进行协调分配任务？
- Map任务会产生临时文件作为输出，需要收集所有的临时文件内容作为Reduce的输入，两种任务之间如何配合？
- Worker可能会crash，正在处理的任务该如何处理？
- Worker执行任务时间可能过长，如何处理？

<a name="KOmGo"></a>
## 课程前置知识
1、MapReduce的运行机制<br />2、Go内置的RPC通信<br />3、Go的锁保护机制<br />4、Go本地的定时任务time.Ticker
<a name="NTwEP"></a>
## 任务运行机制
Coordinator的执行入口在mrcoordinator.go文件中，逻辑就是创建一个Coordinator结构体实例，之后循环执行Done()方法，来判断Coordinator是否已经结束，进而判断整个MR任务是否结束。

Worker的执行入口在mrworker.go文件中，逻辑就是加载Map、Reduce的函数，之后调用worker的Worker方法。当前Worker进程具体该如何执行Map、Reduce任务，需要自己在Worker方法内区实现。

实验只需要修改src/mr里边的三个文件，在里边书写自己的逻辑代码

- coordinator.go
- worker.go
- rpc.go
<a name="lE49e"></a>
## 实现设计
这里先明确一下概念，对启动一个MR任务称之为Job，而Map任务、Reduce任务称为Task。后面都用英文来表示。
<a name="aWetF"></a>
### Coordinator的设计
一个Coordinator需要知道MR Job当前需要处理的files，并且有需要几个Reduce（执行MakeCoordinator 的时候会传入）。

由于一个job会存在map、Reduce阶段，这里给job设计了几个阶段（状态），用整型来表示。如下

| 阶段 | 阶段值 |
| --- | --- |
| 初始阶段 | 1 |
| Map阶段 | 2 |
| Reduce阶段 | 3 |
| 等待Worker下线阶段 | 4 |
| job结束 | 5 |


Coordinator需要知道当前有哪些Worker存在，判断其Worker的存活，进而判断任务是否还在继续执行。所以需要用一个结构体来表是worker节点，判断存活的依据简单的心跳处理。

Coordinator需要推进map、reduce任务的执行，需要维护task列表。（具体task的设计看下面）这里分别为两种类型的task设立两个列表，使用map存放。

因为执行完map任务后，才可以执行reduce任务，这里简单起见，判断各个阶段任务完成的标记使用计数来表示。所以而外需要两个统计数值：map任务完成数量、reduce任务完成数量。

多个Worker请求，需要并发修改task结构体的状态以及统计任务数等，会需要互斥访问，还需要一个mutex进行数据保护。

总结上述，一个Coordinator内包含的结构如下。
```go
type Coordinator struct {
    // Your definitions here.
    files               []string
    reduceNum           int
    jobStage            int
    workerNodeMap       map[int]*workerNode
    mapTaskMap          map[int]*TaskResult
    reduceTaskMap       map[int]*TaskResult
    mapTaskDoneCount    int
    reduceTaskDoneCount int
    taskMutex           sync.Mutex
}
type workerNode struct {
    workerId          int
    lastHeartBeatTime int64
}

const JOB_STAGE_INIT = 1
const JOB_MAP_STAGE = 2
const JOB_REDUCE_STAGE = 3
const JOB_END = 5

const MAP_TASK = "map"
const REDUCE_TASK = "reduce"

```

### Task的设计
mapTask、reduceTask都有很大的相似性，外出来看都是输入、输出。所以这里设计都使用同一个结构体来表示两种阶段的任务。使用一个type字段进行区分即可。这里也给task进行了状态的划分。（这里假设task没有失败的可能）。

| 阶段 | 阶段值 |
| --- | --- |
| 待执行 | 1 |
| 执行中 | 2 |
| 执行完成 | 3 |


```go
const TASK_PENDING = 1
const TASK_EXECUTING = 2
const TASK_COMPLETED = 3

type TaskResult struct {
	TaskId           int    // 由Coordinator进行分配
	TaskType         string // map or reduce
	Status           int
	FilePath         []string // input file path
	ResultPath       []string // output file path
	ExecuteSuccess   bool     // execute result
	WorkerId         int      // execute worker
	NReduce          int
	StartExecuteTime int64
}

type HeartbeatResult struct {
	WorkerId int
	Shutdown bool  // 当true时直接退出进程
}
```

<a name="Y3gUb"></a>
### Worker的设计
worker比较简单，主要是和Coordinator进行通信，执行map、reduce任务。<br />由于map、reduce类型的task都是worker执行，中间文件的处理也落到了worker身上。下面是处理各个阶段的逻辑

- map阶段：worker会根据reduce的数量n，创建n个临时文件，并将map产生的kv结果，分散写入到这n个临时文件中。之后，将这n个临时文件路径作为当前map任务的ResultPath，上报给Coordinator。
- Coordinator协调阶段：将map产出的临时文件，写入到对应的Reduce任务的FilePath中作为输入
- reduce阶段：worker读取所有FilePath中的文件后，做一个sort、merge，执行Reduce函数。将产出写入到mr-out-taskId的文件中（这里简单起见，不写入tmp file了）

这里和实验推荐的一样，将map生成的中间KV数据作为json结构写入到临时文件中。
<a name="P0h4E"></a>
### RPC通信设计
Coordinator和worker之间，是采用不断请求的模式（pull）。Coordinator不会主动给worker发送数据。<br />这里设计了三种的接口交互

- worker定时上报心跳给Coordinator
- worker不断地从Coordinator中获取任务执行
- worker执行完一个任务后，上报执行结果给Coordinator

基本如下图所示：<br />
![RPC通信设计](design.png)

<a name="QilfT"></a>
### 较重要的定时任务
定时任务主要是解决Worker获取到任务后，但是Worker宕机的情况，这时候Worker执行的task需要交给其他Worker去执行。<br />设计上，**通过判定Worker的心跳超时时间来判定Worker是否宕机**，以及按照实验首页说的，简单的判断如果task执行时间超过了10s，也认为该任务需要切换Worker执行。<br />下面是Coordinator的两个定时任务逻辑

- 扫描Worker列表，如果上一次心跳时间超过了预定的超时时间，那么就从Worker列表中剔除
- 扫描task列表，任务处于正在执行中，如果其执行的Worker并不在Worker的列表中，或者执行时间超过了10s，则重置该任务为待执行状态

这样就可以重置Task的状态，存活的Worker就可以获取得到Task执行。

<a name="iLmct"></a>
### Coordinator执行阶段推进
在执行完所有的Map任务之后，才可以执行Reduce任务。设计上，**将任务推进放到了Coordinator中接受任务执行结果的逻辑中**。如果所有的Map任务执行完成了，那么就将JOB_STAGE修改成Reduce阶段，这样Worker就可以获取到Reduce任务去执行。

只有Coordinator知道整体的完成情况，但是Worker不知道。Worker可以简单的设计为如果RPC调用失败，就认为Coordinator结束了，自己也跟着退出。或者有某种RPC结果能让Worker知道当前需要退出。

这里设计通过Worker的心跳返回结果来告知Worker下一步的执行，如果收到shutdown命令，那么就直接退出。Coordinator在Reduce阶段结束，就直接走到end阶段。<br />


## 踩坑
对于go语言不太熟练，一些常见的坑踩了不少

- 切片slice、哈希表map的遍历修改、删除元素
- go内置rpc使用，对于默认值是如何处理的（这里对于各种状态，建议避开默认值）

这里有个坑调试了很久，就是worker在去请求任务的时候，创建的TaskResult结构体是放在了for循环外边，导致在导致每次调用获取到的结果会掺杂了上次调用的结果😭，还有默认值不修改的问题。

下面放点代码示例
```go
func getTaskFromCoordinator(workerId int) TaskResult {
    // 问题就在这里，只创建了一次，内部for循环会共用一个实例，多次请求和掺杂上一次请求的结果
    // result := &TaskResult{}  
    for {
        // 正确应该放在循环里边
        result := &TaskResult{}
        ok := call("Coordinator.GetTaskToExecute", &workerId, result)
        if ok {
            if result.TaskId > -1 {
                return *result
            }
            time.Sleep(time.Second * 2)
        } else {
            log.Fatal("获取任务失败")
        }
    }
}
```

导致的结果就是如下，明明Coordinator返回的taskId是0，为啥worker收到的taskId是-1？？？<br />原因就是**上一次请求的结果返回的就是-1，这一次返回的是0，是个默认值，go的rpc是不会传输，也就是不会去修改-1**，真是日了狗。所以这里建议对**于各种状态、id值的定义不要使用默认值！！！**<br />
![输出日志](logPic.png)

<a name="zhuFS"></a>
## 总结
这里是假设所有的Task最终在10s内都是会执行完成的，10s未完成则交给别的Worker去执行。实际上，Task的执行是会出现各种各样的情况的。而且，这里设计上每一个输入的file作为一个Map任务，会出现计算倾斜的情况。理想情况下需要平均的进行切分数据，让每个Map任务尽可能的处理相同大小的数据。

首次接触go语言来写项目，不会的就边搜索别写，效率偏慢些，debug半天。不过主要是要在关键节点输出详细的日志信息，便于观察各种数据结构的具体变更情况。


## 附录

课程首页：[https://pdos.csail.mit.edu/6.824/index.html](https://pdos.csail.mit.edu/6.824/index.html)<br />
课程实验说明：[https://pdos.csail.mit.edu/6.824/labs/lab-mr.html](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)

<div  align="center">    
 <img src="cover.jpg" width = "200" height = "100" alt="凌落宸" align=center />
</div>

### 参考
