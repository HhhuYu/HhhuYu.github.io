---
title: MIT-6.824|lab1
mathjax: false
categories: 6.824
tags: []
---



```go
// rpc.go
package mr

//
// RPC definitions.
//
// remember to capitalize all names.
//

import (
	"os"
	"strconv"
)

//
// example to show how to declare the arguments
// and reply for an RPC.
//
type ActionType string

const (
	Reduce             ActionType = "reduce"
	Map                ActionType = "map"
	Finish             ActionType = "finish"
	Wait               ActionType = "wait"
	SuccessMsg         string     = "success"
	ShutdownMsg        string     = "shutdown"
	PingIntervalSecond uint       = 2
	Timtout            uint       = 10
)

type TaskState uint

const (
	Idle      TaskState = 0
	InProcess TaskState = 1
	Completed TaskState = 2
)

// Add your RPC definitions here.
type AskForArgs struct {
	WorkerId string
}

type AskForReply struct {
	Action   ActionType
	FilePath string
	TaskID   string
	NReduce  int
	NMap     int
}

type FinishArgs struct {
	WorkerId string
	TaskID   string
	State    TaskState
}

type FinishReply struct {
	Action ActionType
}

type PingArgs struct {
	WorkerId string
}

type PingReply struct {
	Msg string
}

// Cook up a unique-ish UNIX-domain socket name
// in /var/tmp, for the coordinator.
// Can't use the current directory since
// Athena AFS doesn't support UNIX-domain sockets.
func coordinatorSock() string {
	s := "/var/tmp/824-mr-"
	s += strconv.Itoa(os.Getuid())
	return s
}


```


```go
// coordinator.go
package mr

import (
	"fmt"
	"log"
	"math"
	"net"
	"net/http"
	"net/rpc"
	"os"
	"strconv"
	"strings"
	"sync"
	"time"
)

type MapTask struct {
	FilePath     string
	State        TaskState
	RunStartTime time.Time
	WorlerID     string
}

type ReduceTask struct {
	State        TaskState
	RunStartTime time.Time
	WorlerID     string
}

type Intermediate struct {
	Location string
	Size     uint
}

type WorkerInfo struct {
	WorkerID     string
	LastPingTime time.Time
	TaskID       string
	Avaliable    bool
}

type Coordinator struct {
	// Your definitions here.
	MapList        []*MapTask
	ReduceList     []*ReduceTask
	Worker         map[string]*WorkerInfo
	NReduce        int
	RIntermediate  map[string][]*Intermediate
	MapFinished    bool
	ReduceFinished bool
	mu             sync.Mutex
}

// Your code here -- RPC handlers for the worker to call.
func (c *Coordinator) AskForTask(args *AskForArgs, reply *AskForReply) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	workerid := args.WorkerId
	worker, ok := c.Worker[workerid]
	reply.Action = Wait

	if ok && worker.Avaliable {
		if !c.MapFinished {
			for num, task := range c.MapList {
				if task.State == Idle {
					// reply
					taskID := fmt.Sprintf("%s-%d", Map, num)
					reply.FilePath = task.FilePath
					reply.Action = Map
					reply.TaskID = taskID
					reply.NReduce = c.NReduce
					// task
					task.State = InProcess
					task.RunStartTime = time.Now()
					task.WorlerID = workerid
					// worker
					worker.TaskID = taskID
					worker.Avaliable = false

					break
				}
			}

		} else if !c.ReduceFinished {
			for num, task := range c.ReduceList {
				if task.State == Idle {
					taskID := fmt.Sprintf("%s-%d", Reduce, num)
					// reply
					reply.Action = Reduce
					reply.NMap = len(c.MapList)
					reply.TaskID = taskID
					// task
					task.State = InProcess
					task.RunStartTime = time.Now()
					task.WorlerID = workerid
					//worker
					worker.TaskID = taskID
					worker.Avaliable = false
					break
				}
			}
		} else {
			reply.Action = Finish
		}
	}

	return nil
}

func (c *Coordinator) FinishTask(args *FinishArgs, reply *FinishReply) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	taskID := args.TaskID
	workerID := args.WorkerId
	if args.State == Completed && c.Worker[workerID].TaskID == taskID {
		arr := strings.Split(taskID, "-")
		num, err := strconv.Atoi(arr[1])
		if err != nil {
			return err
		}
		ac := arr[0]
		if ac == string(Map) {
			c.MapList[num].State = Completed
			c.MapFinished = true
			for _, ml := range c.MapList {
				if ml.State != Completed {
					c.MapFinished = false
					break
				}
			}
		} else if ac == string(Reduce) {
			c.ReduceList[num].State = Completed
			c.ReduceFinished = true
			for _, rl := range c.ReduceList {
				if rl.State != Completed {
					c.ReduceFinished = false
					break
				}
			}
		}
	}
	c.Worker[workerID].Avaliable = true
	c.Worker[workerID].TaskID = ""

	if c.MapFinished && c.ReduceFinished {
		reply.Action = Finish
	}
	return nil
}

func (c *Coordinator) Ping(args *PingArgs, reply *PingReply) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	wokerid := args.WorkerId
	val, ok := c.Worker[wokerid]
	if ok {
		val.LastPingTime = time.Now()
	} else {
		wi := WorkerInfo{
			WorkerID:     args.WorkerId,
			LastPingTime: time.Now(),
			Avaliable:    true,
		}
		c.Worker[args.WorkerId] = &wi
	}

	reply.Msg = SuccessMsg
	return nil
}

func (c *Coordinator) CheckAvaiable() {

	ticker := time.NewTicker(time.Duration(PingIntervalSecond * uint(time.Second)))
	defer ticker.Stop()

	for range ticker.C {

		c.mu.Lock()

		// worker
		for _, v := range c.Worker {
			if !v.Avaliable {
				continue
			}
			if uint(math.Abs(time.Until(v.LastPingTime).Seconds())) >= 3*PingIntervalSecond {
				v.Avaliable = false
				if len(v.TaskID) != 0 {
					tm := v.TaskID
					c.ResetTask(tm)
					v.TaskID = ""
				}

			}
		}

		if !c.MapFinished {
			for _, v := range c.MapList {
				if v.State == InProcess {

					if math.Abs(time.Until(v.RunStartTime).Seconds()) >= float64(Timtout) {

						wokerid := v.WorlerID
						v.State = Idle
						v.WorlerID = ""
						v.RunStartTime = time.Time{}
						c.Worker[wokerid].Avaliable = false
						c.Worker[wokerid].TaskID = ""
					}
				}
			}
		}

		if !c.ReduceFinished {
			for _, v := range c.ReduceList {
				if v.State == InProcess {
					if math.Abs(time.Until(v.RunStartTime).Seconds()) >= float64(Timtout) {
						wokerid := v.WorlerID
						v.State = Idle
						v.WorlerID = ""
						v.RunStartTime = time.Time{}
						c.Worker[wokerid].Avaliable = false
						c.Worker[wokerid].TaskID = ""
					}
				}
			}
		}

		c.mu.Unlock()
	}

}

func (c *Coordinator) ResetTask(taskID string) {
	arr := strings.Split(taskID, "-")
	num, err := strconv.Atoi(arr[1])
	if err != nil {
		return
	}
	if arr[0] == string(Map) {
		c.MapList[num].State = Idle

	}
	if arr[0] == string(Reduce) {
		c.ReduceList[num].State = Idle
	}

}

//
// start a thread that listens for RPCs from worker.go
//
func (c *Coordinator) server() {
	rpc.Register(c)
	rpc.HandleHTTP()
	//l, e := net.Listen("tcp", ":1234")
	sockname := coordinatorSock()
	os.Remove(sockname)
	l, e := net.Listen("unix", sockname)
	if e != nil {
		log.Fatal("listen error:", e)
	}
	go c.CheckAvaiable()
	go http.Serve(l, nil)
}

//
// main/mrcoordinator.go calls Done() periodically to find out
// if the entire job has finished.
//
func (c *Coordinator) Done() bool {
	// Your code here.
	c.mu.Lock()
	defer c.mu.Unlock()
	ret := false

	if c.MapFinished && c.ReduceFinished {
		ret = true
	}
	return ret
}

//
// create a Coordinator.
// main/mrcoordinator.go calls this function.
// nReduce is the number of reduce tasks to use.
//
func MakeCoordinator(files []string, nReduce int) *Coordinator {
	workerMap := make(map[string]*WorkerInfo)
	mapList := make([]*MapTask, 0)
	reduceList := make([]*ReduceTask, nReduce)
	intermediate := make(map[string][]*Intermediate)
	// Your code here.

	for _, file := range files {
		mapList = append(mapList, &MapTask{
			FilePath: file,
			State:    Idle,
		})
	}

	for j := 0; j < nReduce; j++ {
		reduceList[j] = &ReduceTask{
			State: Idle,
		}
	}

	c := Coordinator{
		NReduce:        nReduce,
		Worker:         workerMap,
		MapList:        mapList,
		ReduceList:     reduceList,
		RIntermediate:  intermediate,
		mu:             sync.Mutex{},
		MapFinished:    false,
		ReduceFinished: false,
	}

	c.server()
	return &c
}

```


```go
// worker.go

package mr

import (
	"crypto/rand"
	"encoding/json"
	"fmt"
	"hash/fnv"
	"io/ioutil"
	"log"
	"net/rpc"
	"os"
	"sort"
	"strconv"
	"strings"
	"time"
)

var WorkerID = "worker-" + RandString(5)

// var diconnect
//
// Map functions return a slice of KeyValue.
//
type KeyValue struct {
	Key   string
	Value string
}

// for sorting by key.
type ByKey []KeyValue

// for sorting by key.
func (a ByKey) Len() int           { return len(a) }
func (a ByKey) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByKey) Less(i, j int) bool { return a[i].Key < a[j].Key }

//
// use ihash(key) % NReduce to choose the reduce
// task number for each KeyValue emitted by Map.
//
func ihash(key string) int {
	h := fnv.New32a()
	h.Write([]byte(key))
	return int(h.Sum32() & 0x7fffffff)
}

//
// main/mrworker.go calls this function.
//

func Worker(mapf func(string, string) []KeyValue,
	reducef func(string, []string) string) {
	quit := make(chan struct{})
	// Your worker implementation here.
	go HeartBeat(WorkerID, quit)
	go func() {
		for {
			time.Sleep(time.Duration(time.Second * time.Duration(PingIntervalSecond)))
			askForArgs := AskForArgs{WorkerId: WorkerID}
			askForReply := AskForReply{}
			CallAskForTask(&askForArgs, &askForReply)
			taskID := askForReply.TaskID
			finishArgs := FinishArgs{TaskID: taskID, State: InProcess, WorkerId: WorkerID}
			// fmt.Println(askForReply.Action)
			switch askForReply.Action {
			case Map:
				{
					filename := askForReply.FilePath
					file, err := os.Open(filename)
					if err != nil {
						log.Fatalf("cannot open %v", filename)
					}
					content, err := ioutil.ReadAll(file)
					if err != nil {
						log.Fatalf("cannot read %v", filename)
					}
					file.Close()
					kva := mapf(filename, string(content))
					if WriteToIntermediate(kva, askForReply.NReduce, askForReply.TaskID) {
						finishArgs.State = Completed
					}

				}
			case Reduce:
				{
					arr := strings.Split(taskID, "-")
					num, err := strconv.Atoi(arr[1])
					if err != nil {
						log.Fatalf("cannot read intermediate file: %v", taskID)
					}
					intermediate := []KeyValue{}
					for i := 0; i < askForReply.NMap; i++ {
						filename := fmt.Sprintf("mr-%d-%d", i, num)
						file, err := os.OpenFile(filename, os.O_RDONLY, 0666)
						if err != nil {
							log.Fatalf("cannot open %v", filename)
						}
						dec := json.NewDecoder(file)
						for {
							var kv []KeyValue
							if err := dec.Decode(&kv); err != nil {
								break
							}
							intermediate = append(intermediate, kv...)
						}
						file.Close()
					}

					sort.Sort(ByKey(intermediate))

					ofilename := fmt.Sprintf("mr-out-%d", num)

					tmpFile, err := ioutil.TempFile("", "mr-reduce-"+WorkerID+"-"+taskID)
					if err != nil {
						fmt.Println("临时文件创建失败")
					}
					i := 0
					for i < len(intermediate) {
						j := i + 1
						for j < len(intermediate) && intermediate[j].Key == intermediate[i].Key {
							j++
						}
						values := []string{}
						for k := i; k < j; k++ {
							values = append(values, intermediate[k].Value)
						}
						output := reducef(intermediate[i].Key, values)

						// this is the correct format for each line of Reduce output.
						fmt.Fprintf(tmpFile, "%v %v\n", intermediate[i].Key, output)

						i = j
					}
					tmpFile.Close()
					os.Rename(tmpFile.Name(), ofilename)
					finishArgs.State = Completed
				}
			case Wait:
				continue
			case Finish:
				quit <- struct{}{}
				return
			}
			// finishArgs := FinishArgs{TaskID: taskID, State: Completed}
			finishReply := FinishReply{}
			CallFinishTask(&finishArgs, &finishReply)
			if finishReply.Action == Finish {
				quit <- struct{}{}
				return
			}
		}
	}()
	<-quit
	log.Println("Shutdown Server...")

	// uncomment to send the Example RPC to the coordinator.
	// CallExample()

}

func WriteToIntermediate(kva []KeyValue, nreduce int, taskID string) bool {
	intermediate := make([][]KeyValue, nreduce)
	for _, kv := range kva {
		bucket := ihash(kv.Key) % nreduce
		// fmt.Println(kv.Key)
		intermediate[bucket] = append(intermediate[bucket], kv)
	}
	arr := strings.Split(taskID, "-")
	num, err := strconv.Atoi(arr[1])
	if err != nil {
		return false
	}
	for bu, kvs := range intermediate {
		interFileName := fmt.Sprintf("mr-%d-%d", num, bu)
		tmpFile, err := ioutil.TempFile("", "mr-map-"+WorkerID+"-"+taskID)
		if err != nil {
			fmt.Println("临时文件创建失败")
			return false
		}
		enc := json.NewEncoder(tmpFile)
		err = enc.Encode(kvs)
		if err != nil {
			return false
		}
		tmpFile.Close()
		os.Rename(tmpFile.Name(), interFileName)
	}
	return true

}

func RandString(len int) string {
	b := make([]byte, len)
	if _, err := rand.Read(b); err != nil {
		panic(err)
	}
	s := fmt.Sprintf("%X", b)
	// fmt.Println(s)
	return s
}

// heartbeat
func HeartBeat(workerID string, quit chan struct{}) {
	cnt := 0

	ticker := time.NewTicker(time.Duration(PingIntervalSecond) * time.Second)
	defer ticker.Stop()

	for range ticker.C {
		args := PingArgs{
			WorkerId: workerID,
		}
		reply := PingReply{}
		if cnt == 3 {
			quit <- struct{}{}
			return
		}
		if !CallPing(&args, &reply) {
			cnt += 1
			continue
		}

		if reply.Msg == ShutdownMsg {
			quit <- struct{}{}
			return
		} else if reply.Msg == SuccessMsg {
			cnt = 0
		} else {
			cnt = cnt + 1
		}
	}

}

func CallAskForTask(args *AskForArgs, reply *AskForReply) bool {
	return call("Coordinator.AskForTask", &args, &reply)
}

func CallFinishTask(args *FinishArgs, reply *FinishReply) bool {
	return call("Coordinator.FinishTask", &args, &reply)
}

func CallPing(args *PingArgs, reply *PingReply) bool {
	return call("Coordinator.Ping", &args, &reply)
}

//
// send an RPC request to the coordinator, wait for the response.
// usually returns true.
// returns false if something goes wrong.
//
func call(rpcname string, args interface{}, reply interface{}) bool {
	// c, err := rpc.DialHTTP("tcp", "127.0.0.1"+":1234")
	sockname := coordinatorSock()
	c, err := rpc.DialHTTP("unix", sockname)
	if err != nil {
		log.Fatal("dialing:", err)
	}
	defer c.Close()

	err = c.Call(rpcname, args, reply)
	if err == nil {
		return true
	}

	fmt.Println(err)
	return false
}


```