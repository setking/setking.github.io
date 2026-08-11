---
title: MapReduce
author: setKing
date: 2026-06-07 11:33:00 +0800
categories: [MIT 6.824]
tags: [论文, 分布式, 原理]
pin: true
math: true
mermaid: true
---

<center>MapReduce</center>

### 核心理念

**MapReduce 的本质并非简单的“排序+统计”，而是一种将“分治”思维应用于海量数据处理的通用范式。**

在单机无法处理 PB 级数据的时代，如何让成千上万台廉价的商用服务器协同工作？MapReduce 给出的答案是：

1. **拆分（Divide）**：将输入数据切分成等大小的分片，分配给多台机器并行执行 Map 阶段。
2. **编排（Orchestrate）**：通过底层的 **Shuffle（混洗）** 机制，自动将 Map 产出的无序键值对，按 Key 进行**分区（Partition）**和**排序（Sort）**，精准地路由到对应的 Reduce 节点。
3. **归约（Conquer）**：Reduce 节点拉取属于自己分区的所有有序数据流，进行全局归并，最终产出结果文件。

这一过程完美诠释了分布式系统如何将**无序的输入**，编排为**有序、分区的中间结果**，并最终**汇聚为全局输出**。

#### 实现

### 1. Master：中心化的调度大脑

Master 负责维护任务的**状态机**（Idle / InProgress / Completed）和 Worker 的健康状况。

> _注：为控制篇幅，代码中省略了超时重试（Time-out Retry）机制，后续版本将加入该容错逻辑。_

```
type MapTask struct {
	Id        int
	FileName  string
	Status    string // 任务状态 "Idle", "InProgress", "Completed"
	StartTime time.Time
}

type ReduceTask struct {
	Id        int
	Status    string // 任务状态 "Idle", "InProgress", "Completed"
	StartTime time.Time
	NMap      int
}

type Master struct {
	files       []string
	nReduce     int
	mapTasks    []MapTask
	reduceTasks []ReduceTask
	mu          sync.Mutex

}


func (m *Master) server() {
	err := rpc.Register(m)
	if err != nil {
		log.Fatal("register error:", err)
	}
	listener, err := net.Listen("tcp", ":8085")
	if err != nil {
		log.Fatal("listen error:", err)
	}
	defer listener.Close()
	fmt.Printf("Master server started at %s\n", listener.Addr())
	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Println("accept error:", err)
			continue
		}
		go rpc.ServeConn(conn)
	}
}


func (m *Master) Done() bool {
	for _, task := range m.mapTasks {
		if task.Status != "Completed" {
			return false
		}
	}
	for _, task := range m.reduceTasks {
		if task.Status != "Completed" {
			return false
		}
	}
	return true
}


func MakeMaster(files []string, nReduce int) *Master {
	m := Master{
		files:       files,
		nReduce:     nReduce,
		mapTasks:    make([]MapTask, len(files)),
		reduceTasks: make([]ReduceTask, nReduce),
	}
	for i, file := range files {
		m.mapTasks[i] = MapTask{
			Id:        i,
			FileName:  file,
			Status:    "Idle",
			StartTime: time.Now(),
		}
	}
	for i := 0; i < nReduce; i++ {
		m.reduceTasks[i] = ReduceTask{
			Id:        i,
			Status:    "Idle",
			StartTime: time.Now(),
			NMap:      len(files),
		}
	}
	go m.server()
	return &m
}
```

我实现的master`server`注册和启动rpc，`MakeMaster`初始化构建map和reduce，下面是rpc的实现：

```
type TaskCompletedArgs struct {
	TaskId    int
	TaskType  string   // "Map" or "Reduce"
	FileNames []string // intermediate file names
}
type TaskRequest struct {
	// Define the fields for the map request
}

type TaskReply struct {
	TaskId   int
	FileName string
	NReduce  int
	NMap     int
	TaskType string // "Map" or "Reduce"
}

func (m *Master) GetTask(args *[]string, reply *TaskReply) error {
	m.mu.Lock()
	defer m.mu.Unlock()
	fmt.Printf("Master: Received task request at %v\n", time.Now())
	for id := range m.mapTasks {
		task := &m.mapTasks[id]
		if task.Status == "Idle" {
			task.Status = "InProgress"
			task.StartTime = time.Now()
			reply.TaskId = id
			reply.FileName = task.FileName
			reply.NReduce = m.nReduce
			reply.TaskType = "Map"
			return nil
		}
		if task.Status == "InProgress" {
			reply.TaskType = "Wait"
		}
	}
	for i, task := range m.reduceTasks {
		if task.Status == "Idle" {
			m.reduceTasks[i].Status = "InProgress"
			m.reduceTasks[i].StartTime = time.Now()
			reply.TaskId = i
			reply.TaskType = "Reduce"
			reply.NMap = len(m.mapTasks)
			reply.NReduce = m.nReduce
			return nil
		}
	}
	return nil
}

func (m *Master) TaskCompleted(args *TaskCompletedArgs, reply *TaskReply) error {
	m.mu.Lock()
	defer m.mu.Unlock()
	if args.TaskType == "Map" {
		task := m.mapTasks[args.TaskId]
		if task.Status == "InProgress" {
			task.Status = "Completed"
			m.mapTasks[args.TaskId] = task
		}
	} else if args.TaskType == "Reduce" {
		task := m.reduceTasks[args.TaskId]
		if task.Status == "InProgress" {
			task.Status = "Completed"
			m.reduceTasks[args.TaskId] = task
			reply.NMap = len(m.mapTasks)
			return nil
		}
	}
	return nil
}

```

`GetTask`用于获取任务和推进任务状态，`TaskCompleted`用于推进任务完成

> **GetTask 的设计思路**：为了防止多个 Worker 抢同一个任务，Master 在分配任务时必须加锁（`mu.Lock()`）。分配时遵循**“Map 优先，Reduce 靠后”**的原则，只有所有 Map 都进入 `InProgress` 或 `Completed` 状态后，Reduce 任务才会被分发。

### 2. Worker：任务的执行者

Worker 通过 RPC 向 Master 请求任务，并根据返回的 `TaskType` 执行 Map 或 Reduce 逻辑。

```
func Worker(mapf func(string, string) []KeyValue,
	reducef func(string, []string) string,
) {
	rpcClient := client()
	for {
	 // 1. 请求任务
		reply := TaskReply{}
		req := []string{}

		err := rpcClient.Call("Master.GetTask", &req, &reply)
		if err != nil {
			log.Println("rpc call error:", err)
			continue
		}
		 // 2. 执行任务
		switch reply.TaskType {
		case "Map":
			doMapTask(reply, mapf)
			args := TaskCompletedArgs{TaskId: reply.TaskId, TaskType: "Map"}
			err := rpcClient.Call("Master.TaskCompleted", &args, &TaskReply{})
			if err != nil {
				log.Println("rpc call error:", err)
			}
		case "Reduce":
			doReduceTask(reply, reducef)
			args := TaskCompletedArgs{TaskId: reply.TaskId, TaskType: "Reduce"}
			err := rpcClient.Call("Master.TaskCompleted", &args, &TaskReply{})
			if err != nil {
				log.Println("rpc call error:", err)
			}
		default:
			log.Printf("Unknown task type: %s", reply.TaskType)
		}
	}
}
```

### 3. 关键机制：Map 端的分区与排序

在 `doMapTask` 中，我们需要手动实现论文里提到的“Map 端框架”职责：

- **分区**：通过 `ihash(Key) % NReduce` 决定当前键值对去往哪个 Reducer。
- **排序**：在写入磁盘前，对每个分区内的数据按 Key 排序，方便 Reduce 端进行归并。

```
func doMapTask(task TaskReply, mapf func(string, string) []KeyValue) {
	file, err := os.Open(task.FileName)
	if err != nil {
		log.Fatalf("cannot open %v", task.FileName)
	}
	content, err := io.ReadAll(file)
	if err != nil {
		log.Fatalf("cannot read %v", task.FileName)
	}
	file.Close()
	partitions := make([][]KeyValue, task.NReduce)
	kva := mapf(task.FileName, string(content))
	for _, kv := range kva {
		partition := ihash(kv.Key) % task.NReduce
		partitions[partition] = append(partitions[partition], kv)
	}
	for i := 0; i < task.NReduce; i++ {
		sort.Slice(partitions[i], func(a, b int) bool {
			return partitions[i][a].Key < partitions[i][b].Key
		})
	}
	for i := 0; i < task.NReduce; i++ {
		tmpFile, err := os.CreateTemp(".", "mr-tmp-*")
		if err != nil {
			log.Fatalf("cannot create temp file: %v", err)
		}
		enc := json.NewEncoder(tmpFile)
		for _, kv := range partitions[i] {
			if err := enc.Encode(kv); err != nil {
				log.Fatalf("cannot encode kv pair: %v", err)
			}
		}
		tmpFile.Close()
		finalName := fmt.Sprintf("mr-%d-%d", task.TaskId, i)
		err = os.Rename(tmpFile.Name(), finalName)
		if err != nil {
			log.Fatalf("cannot rename temp file: %v", err)
		}
	}
}
```

### 4. 数据处理：Reduce的归并排序

_在 `doReduceTask` 中，我们需要手动实现 Reduce 端的核心逻辑_：

- **归并**：_从所有 Map 生成的中间文件中，读取属于自己分区（`mr-\*-[当前TaskId]`）的数据，在内存中按 Key 进行归并（Merge），最后交给用户定义的 `reducef` 函数处理。”_。

```
func doReduceTask(task TaskReply, reducef func(string, []string) string) {
	var decoders []*json.Decoder
	for i := 0; i < task.NMap; i++ {
		fileName := fmt.Sprintf("mr-%d-%d", i, task.TaskId)
		file, err := os.Open(fileName)
		if err != nil {
			log.Fatalf("cannot open intermediate file: %v", err)
		}
		decoders = append(decoders, json.NewDecoder(file))
		file.Close()
	}
	kvs := make(map[string][]string)
	for _, dec := range decoders {
		for {
			var kv KeyValue
			if err := dec.Decode(&kv); err == io.EOF {
				break
			} else if err != nil {
				log.Fatalf("cannot decode kv pair: %v", err)
			}
			kvs[kv.Key] = append(kvs[kv.Key], kv.Value)
		}
	}
	outputFile, err := os.Create(fmt.Sprintf("mr-out-%d", task.TaskId))
	if err != nil {
		log.Fatalf("cannot create output file: %v", err)
	}
	defer outputFile.Close()
	for key, values := range kvs {
		output := reducef(key, values)
		fmt.Fprintf(outputFile, "%v %v\n", key, output)
	}
}

```

### 总结与展望

从以上代码不难看出MapReduce的master负责调度，worker负责推进任务的状态，`doMapTask`的分区体现了分布式系统的分治思维，通过对数据的分区和排序让数据通过固定的类别在reduce阶段归并计算展示了**分布式系统如何将无序的输入，编排为有序、分区的中间结果，并最终汇聚为全局输出**。

> **当前实现的局限**：目前的 Master 缺乏超时处理机制。如果 Worker 在 `InProgress` 状态下意外崩溃，任务将永远无法完成。后续我计划在 Master 的轮询中检查 `StartTime`，对超时任务进行重新分配，以此模拟真实的分布式容错场景。
