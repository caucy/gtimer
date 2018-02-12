# timer
用golang实现的定时器，基于delayqueue

# 实现
实现受到了Java DelayQueue.java的启发
源码地址
[DelayQueue.java](http://www.docjar.com/html/api/java/util/concurrent/DelayQueue.java.html)

依赖的几个结构依次为为
timer -> delayqueue -> priorityqueue -> heap

由于golang的Condition不支持wait一段时间，所以使用golang原生的Timer来替代了Condition在delayqueue中的作用

# Installation
## Install:
```
go get -u github.com/vearne/gtimer
```
## Import:
```
import "github.com/vearne/gtimer"
```


## Quick Start
```
package main


import (
	"fmt"
	"time"
	"github.com/vearne/gtimer"
	"strconv"
	"math/rand"
	"sync"
	log "github.com/sirupsen/logrus"
)



func main(){
	wg := sync.WaitGroup{}
	timer := gtimer.NewSuperTimer(5)
	// concurrent push task
	for i:=0;i<3;i++{
		wg.Add(1)
		go push(timer, "worker" + strconv.Itoa(i))
		wg.Done()
	}
	wg.Wait()
	log.Infof("[producer]------push ok-------")
	go func(){
		log.Infof("[start]try to stop")
		time.Sleep(5 * time.Second)
		timer.Stop()
		log.Infof("[end]try to stop")
	}()
	// wait until stop
	timer.Wait()

}

func DefaultAction(t time.Time, value string){
	fmt.Printf("trigger_time:%v, value:%v\n", t, value)
}

func push(timer *gtimer.SuperTimer, name string){
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	for i:=0;i< 10;i++{
		now := time.Now()
		t := now.Add(time.Second * time.Duration(r.Int63n(5)) + 3)
		log.Infof("[produce] now:%v, target:%v\n", now.UnixNano(), t.UnixNano())
		value := fmt.Sprintf("%v:value:%v", name, strconv.Itoa(i))
		// create a delayed task
		item := gtimer.NewDelayedItemFunc(t, value, DefaultAction)
		log.Infof("%v", item.OnTrigger)
		timer.Add(item)
	}
}
```

use NewDelayedItemFunc, we can create a task
```
// triggerTime is time of the task should be execute
func NewDelayedItemFunc(triggerTime time.Time, value string, f func(time.Time, string)) *Item
```
task struct like 
```
type Item struct {
	value    string // The value of the item; arbitrary.
	priority int64    // The priority of the item in the queue.
	// The index is needed by update and is maintained by the heap.Interface methods.
	index int // The index of the item in the heap.
	// when task is ready, execute OnTrigger function
	OnTrigger func(time.Time, string)
}
```