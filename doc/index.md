---
title: "Synchronized Queues in Golang"
date: 2021-03-12
slug: "/syncqueue"
---

## Synchronized Queue in Golang

All code files are in Github at [dmh2000/golang-bounded-queue](https://github.com/dmh2000/golang-bounded-queue)

### Queue

A queue is a data structure with first-in/first-out (FIFO) semantics.
A generic queue typically has more or less the following methods:

- init : initialize or otherwise create a queue
- put : adds an element to the tail of the queue.
- get : takes the element from the head of the queue.
- length : how many elements are in the queue
- capacity : how many elements the queue can hold

A bounded queue is one that also has a 'capacity'. In that case, the queue is set up to contain a fixed number of elements.

Queues are used in various algorithms, such as breadth first search. Another common use of a queue is to provide a means of communication between threads. If a queue is used for that purpose, then it may have blocking semantics. The 'get' method may block if the queue is empty. The 'put' method may block if the queue is full. If that is how it works, then there can be a TryPut method that adds an element if there is room in the queue or returns an indication that the queue is full, without blocking. Likewise there can be a TryGet method that returns an element or an indication the queue is empty. Of course in Go you have buffered channels, which are effectively queues with blocking and non-blocking support.

Typically, if a queue is used for communication between threads it is expected to live for the life of the program. It gets tricky if interthread queues are to be created and discarded over the life of execution. This can lead to memory leaks and threads hung on blocking Get's or Put's if care is not taken to make sure all goroutines are released from the queue before sending it go garbage collection. The code here doesn't detect or handle that case. It is up to the application to clean up.

#### Why have a bound?

Its possible to have an unbounded queue that never gets full. Using dynamic allocation you could implement an unbounded queue. In C++ a list or vector can grow until there is no available memroy. In Go, the container/list object has no bound on its number of elements. These are unbounded until you run out of memory. There is always an implicit bound. You might want to bound a queue as a type of throttling. Say a system has processing power that can only support N things going on a time. An input queue of capacity N would give it a way to stop accepting things until less than N are working. Or, maybe the design wants to know it has enough memory to support all its functions. In many real time systems there is a practice that calls for allocating all resources during initialization so that all the constraints are known and can be analyzed.

### Queue API

Here is an interface definition for a simple FIFO queue with a set bound. This interface does not support blocking semantics. Its intended to specify the methods needed to encapsulate an underlying data structure.

```go
// Queue - interface for a simple, non-thread-safe queue with a set capacity
type Queue interface {
	// current number of elements in the queue
	Len() int

	// maximum number of elements allowed in queue
	Cap() int

	// enqueue a value on the tail of the queue
	Push(value interface{})  error

	// dequeue and return a value from the head of the queue
	Pop() (interface{},error)
}

```

#### Implementations

I have several implementations of the Queue interface.

- ListQ
  - queue using a container/list
  - **queue_list.go**
- RingQ
  - queue using a container/ring
  - **queue_ring.go**
- HeapQ
  - queue using a container/heap
  - **queue_heap.go**
  - in this case, its implemented as a priority queue rather than FIFO
- CircularQ
  - queue using a circular buffer
  - **queue_circular.go**

The data elements are interface{} so any type can be used. This matches some of the approaches in the standard library for certain data structures. These implementations can be passed to any function needed a Queue.

There is one additional version that works like the Queue interface, where the data type is a native 'int' instead of interface{}. It uses the circular buffer approach but since its not using interface{} it might have better performance. The analysis will tell us that.

- NativeIntQ
  - queue using a circular buffer with mutex/condition variable
  - queue_native.go
  - typesafe for 'int'

At this point I'm not sure about the performance of any of these. We need to measure that.approaches.

### SynchronizedQueue

The Queue interface doesn't support some of the methods that are convenient for using in a threaded environment. The interface doesn't guarantee thread safety, nor does it specify blocking or non-blocking semantics. For that purpose, here is an extended interface that supports thread-safety and both blocking and non-blocking semantics. I provide an implementation of this interface that wraps a Queue with a SynchronizedQueue interface.

```go

// SynchronizedQueue is a queue with a bound on the number of elements in the queue
// Any implementation of this SHOULD promise thread-safety and the proper blocking semantics.
type  SynchronizedQueue interface {

	// add an element onto the tail queue
	// if the queue is full, the caller blocks
 	Put(value interface{})

	// add an element onto the tail queue
	// if the queue is full an error is returned
	TryPut(value interface{}) error

	// get an element from the head of the queue
	// if the queue is empty the caller blocks
	Get() interface{}

	// try to get an element from the head of the queue
	// if the queue is empty an error is returned
	TryGet() (interface{}, error)

	// current number of elements in the queue
 	Len() int

	// capacity maximum number of elements the queue can hold
	Cap() int

	// close any resources (required for channel version)
	Close()

	// string representation
	fmt.Stringer
}
```

#### Synchronized Queue Using Channels

Of course, in the Go language, there are buffered channels, which literally are bounded queues. If you aren't familiar with Go channels, search for 'golang buffered channel' and there is lots of imformation. The official [Go Tour](https://tour.golang.org/concurrency/2) has a basic explanation. There are tons of references online about how to use buffered channels.

For channels there is an implementation of SyncrhonizedQueue without requiring the wrapper. There is no need for the Mutex/Cond support when you use channels.

See file [queue_channel.go](https://github.com/dmh2000/golang-bounded-queue/blob/main/queue_channel.go).

One other note about the channel version. If the queue is to be discarded at some point in execution but the program continues, the channel must be closed and all remaining data must be received so it doesn't leak. The interface has a Close() method that can be used by the application to close the channel when the queue is no longer needed. Only a producer calling Put may call Close(). A closed channel will panic if a Put is attempted. Once a channel is closed, any remaining data in the channel may still be accessible to Get's.

#### SynchronizedQueue Using Mutex/Condition Variable

There is an implementation of the SynchronizedQueue interface in the file **queue_sync.go**. This implementation takes a Queue and wraps it with the proper sync methods. In this case it uses the common Mutex/Condition variable approach. A function to create an instance of this interface is provided.

```go
// NewSynchronizedQueue is a factory for creating synchronized queues
// it takes an instance of the Queue interface and wraps it using
// a mutex and condition variable
// returns an instance of SynchronizedQueue
func NewSynchronizedQueue(q Queue) SynchronizedQueue
```

##### Mutex/Condition Variable

If you are familiar with the Mutex/Condition Variable paradigm, you can skip this section.

A Condition Variable is a synchronization object that allow threads to wait until a condition occurs. It does this in conjunction with a Mutex to provide mutual exclusion. The implemenation of a interthread queue is a typical usage of a Mutex/Condition variable pair.

In the Go language, the standard libary package 'sync' provides both _Mutex_ and _Cond_. It works like this:

```go

// declare a Mutex and A Cond
var mtx sync.Mutex
var cvr *sync.Cond

// initialize them
mtx = sync.Mutex{}
cvr = sync.NewCond(&mtx)
// the mutex is attached to the condition variable and after that
// it is accessed through the L.Lock() and L.Unlock() functions

// a function that responds if a condition is fulfilled
func f(cvr *sync.Cond) {
	// lock the mutex
	cvr.L.Lock()

	// make sure the mutex is unlocked when the function returns
	defer cvr.L.Unlock()

	// the mutex is locked at this point so its thread safe
	// to manipulate the data structure or resource

	// while some condition is not true, execute the loop
	for condition == false {
		// this function unlocks the associated mutex
		// and blocks the goroutine on the condition variable
		// It will wake up if some other goroutine calls Signal
		// on the condition variable
		cvr.Wait()
	}

	// the mutex is locked at this point so its thread safe
	// to manipulate the data structure or resource

	// the condition is now true, so do whatever is needed

	// signal a waiter if any
	// this wakes up one other goroutine that is blocked on the
	// condition variable. There is also Broadcast which will
	// wake up all goroutines waiting on the condition variable
	cvr.Signal()

	// the defer unlock releases the mutex
}
```

#### SyncrhonizedQueue Implementation

The file [queue_sync.go](https://github.com/dmh2000/golang-bounded-queue/blob/main/queue_sync.go) has the wrapper for a Queue that provide thread-safety using Mutex/Condition Variable support.

```go
// SynchronizedQueueImpl is an implementation of the SynchronizedQueue interface
// using a Mutex and 2 condition variables.
type SynchronizedQueueImpl struct {
	queue Queue	    // some data structure for backing the queue
	mtx sync.Mutex      // a mutex for mutual exclusion
	putcv *sync.Cond    // a condition variable for controlling Puts
	getcv *sync.Cond    // a condition variable for controlling Gets
}
```

Note that there are two condition variables band a single mutex. That is to support blocking on both ends, Put and Get. The single mutex protects the data structure in both the Put and Get calls. Put operations block on the **putcv** condition variable and signals the **getcv** condition variable when the Put adds an element to the queue. A Get operation works in the opposite direction, blocking on the **getcv** condition variable and signalling the **putcv** condition variable when an element is removed from the queue.

The non-blocking TryPut and TryGet operations still need to signal their opposite condition variable in case the other end is using the blocking version.

#### Queue Using container/list with interface{}

In this implementation the container/list data structure is used. In hindsight using a List is probably not the best approach since the queue is bounded so it doesn't need the flexibility of a List to shrink and grow. We'll see in the analysis.

See file [queue_list.go](https://github.com/dmh2000/golang-bounded-queue/blob/main/queue_list.go).

```go
// ListQueue
// the container/list data structure supports the semantics and methods
// needed for the Queue interface, with the exception of a capacity.
type ListQueue struct {
	list *list.List	 // contains the elements currently in the queue
	capacity int	     // maximum number of elements the queue can hold
}

// ...

// Synchronized List Queue Factory
func NewSyncList(cap int) SynchronizedQueue {
	var lq Queue
	var bq SynchronizedQueue

	// create a ListQueue
	lq = NewListQueue(cap)

	// wrap it with a SynchronizedQueue
	bq = NewSynchronizedQueue(lq)

	return bq
}
```

#### Queue Using container/ring with interface{}

In this implementation the container/ring data structure is used.

See file [queue_list.go](https://github.com/dmh2000/golang-bounded-queue/blob/main/queue_list.go).

```go
// RingQueue - a Queue backed by a container/ring
type RingQueue struct {
	ring *ring.Ring	 // preallocated ring for all slots in the queue
	head *ring.Ring  // head of the queue
	tail *ring.Ring  // tail of the queue
	capacity int     // maximum number of elements the ring can hold
	length int       // current number of element in the ring
}

// ...

// Synchronized RingQueue factory
func NewSyncRing(cap int) SynchronizedQueue {
	var rq Queue
	var bq SynchronizedQueue
	// create a ring queue
	rq = NewRingQueue(cap)

	// wrap it with a SynchronizedQueue
	bq = NewSynchronizedQueue(rq)

	return bq
}
```

#### Queue Using circular buffer with interface{}

This version uses a homegrown circular buffer as the queue data structure. Just guessing it should have better performance than the list. However it still uses interface{} for data elements so it might have some overhead for that vs a native data type.

See file [queue_circular.go](https://github.com/dmh2000/golang-bounded-queue/blob/main/queue_circular.go).

```go
// CircularQueue
type CircularQueue struct {
	queue []interface{}	// data
	head     int		// items are pulled from the head
	tail     int		// items are pushed to the tail
	length   int		// current number of elements in the queue
	capacity int	    // maximum allowed elements total
}

// ...

// Synchronized CircularQueue Factory
func NewSyncCircular(cap int) SynchronizedQueue {
	var cq Queue
	var bq SynchronizedQueue

	// create a circular queue
	cq = NewCircularQueue(cap)

	// wrap it with a synchronized queue
	bq = NewSynchronizedQueue(cq)

	return bq
}
```

#### SynchronizedQueue Using circular with native ints

This version uses a circular buffer as the queue data structure. It is almost identical to the previous circular buffer version with the exception it only supports 'int' elements. I'm guessing that this may be a bit faster than the empty interface version. This version is not compatible with the SynchronizedQueue interface so it has its own mutual exclusion support.

See file [queue_native.go](https://github.com/dmh2000/golang-bounded-queue/blob/main/queue_native.go).

```go
// NativeIntQ iis a type specific implementation
type NativeIntQ struct {
	queue []int			// data
	head     int		// items are pulled from the head
	tail     int		// items are pushed to the tail
	length   int		// current number of elements in the queue
	capacity int	    // maximum allowed elements total
	mtx sync.Mutex      // a mutex for mutual exclusion
	putcv *sync.Cond    // a condition variable for controlling Puts
	getcv *sync.Cond    // a condition variable for controlling Gets
}

// NewNativeQueue is a factory for creating queues
// that use a condition variable and circular buffer
// for the specific type. In this case 'int'.
func NewNativeQueue(size int) *NativeIntQ {
	var nvq NativeIntQ

	// allocate the whole slice during init
	nvq.queue = make([]int,size,size)
	nvq.head = 0
	nvq.tail = 0
	nvq.length = 0
	nvq.capacity = size
	nvq.mtx = sync.Mutex{}
	nvq.putcv = sync.NewCond(&nvq.mtx)
	nvq.getcv = sync.NewCond(&nvq.mtx)

	return &nvq
}

```

#### Queue Using container/heap with interface{} (PriorityQueue)

In this implementation the container/heap structure is used. This is a special case. The implementation
creates a priority list. It is modeled after the [PriorityQueue example](https://golang.org/pkg/container/heap/#example__priorityQueue) from the standard library.

This implementation requires a separate set of tests because the other ones use plain old ints for their data but this one requires a PriorityItem with both a **value interface{}** and **priority int**. It could be implemented with an int that represents both the value and priority but then its value won't be type agnostic.

See file [queue_priority.go](https://github.com/dmh2000/golang-bounded-queue/blob/main/queue_priority.go).

```go
// one item in the priority queue
type PriorityItem struct {
	value interface{}
	priority int
}

// ...

// Synchronized PriorityQueue factory
func NewSyncPriorityQueue(cap int) SynchronizedQueue {
	var pq Queue
	var bq SynchronizedQueue

	// create the int heap
	pq = NewPriorityQueue(cap)

	// wrap it in the syncrhonized bounded queue
	bq = NewSynchronizedQueue(pq)

	return bq
}
```

### Testing

Three files have test code using the Go native test framework.

#### Synchronous Tests

The file [sync_test.go](https://github.com/dmh2000/golang-bounded-queue/blob/main/sync_test.go) contains tests where non-blocking TryPut's and TryGet's are performed in a single goroutine environment. These provide a test that the basic Try operations seem to work. They also test the capacity and length limits of the queues. There is no blocking in these tests.

As of this writing, all the synchronous tests pass.

```bash
# -v : verbose output
# -run <regex> : run only test functions match the regex
# . build with all files in current directory
$ go test -v -run Test.*Sync .
=== RUN   TestChannelSync
--- PASS: TestChannelSync (0.00s)
=== RUN   TestListSync
--- PASS: TestListSync (0.00s)
=== RUN   TestCircularSync
--- PASS: TestCircularSync (0.00s)
=== RUN   TestQueueNativeSync
--- PASS: TestQueueNativeSync (0.00s)
PASS
ok      dmh2000.xyz/queue       0.002s
```

#### Asynchronous Tests

The file [async_test.go](https://github.com/dmh2000/golang-bounded-queue/blob/main/async_test.go) contains tests where blocking Put's and Get's are performed in separate goroutines, one as a producer, one as a consumer. There are two versions of the tests, one where the Put and Get loops have no delays in them. The second is similar but with a random delay before each Put and Get. Intended to check that the blocking and wakeups are working properly.

As of this writing, all the synchronous tests pass.

```bash
# -v : verbose output
# -run <regex> : run only test functions match the regex
# . build with all files in current directory
$ go test -v -run Test.*Async .
=== RUN   TestChannelAsync
--- PASS: TestChannelAsync (0.95s)
=== RUN   TestListAsync
--- PASS: TestListAsync (1.29s)
=== RUN   TestCircularAsync
--- PASS: TestCircularAsync (1.44s)
=== RUN   TestNativeAsync
--- PASS: TestNativeAsync (0.00s)
PASS
ok      dmh2000.xyz/queue       3.676s
```

#### Benchmarks

The file [benchmark_test.go](https://github.com/dmh2000/golang-bounded-queue/blob/main/benchmark_test.go) contains tests both synchronous and asynchronous version similar to the Async tests, but intended to be used as benchmarks for timing and memory usage.

```bash
# -v : verbose output
# -run <regex> : run only test functions match the regex
# -bench : run benchmarks
# . : build with all files in current directory
# -benchmem : measure memory usage
# -memprofile <file> : write memory usage information to <file>
# -cpuprofile <file> : write cpu usage information to <file>
$ go test -v -run Benchmark.* -bench . -benchmem -memprofile mem.out -cpuprofile cpu.out
goos: linux
goarch: amd64
pkg: dmh2000.xyz/queue
cpu: Intel(R) Core(TM) i5-3470 CPU @ 3.20GHz
BenchmarkQueueChannelSync
BenchmarkQueueChannelSync-4       660957              1941 ns/op             424 B/op          3 allocs/op
BenchmarkListSync
BenchmarkListSync-4               366070              3276 ns/op            1184 B/op         24 allocs/op
BenchmarkCircularSync
BenchmarkCircularSync-4           506072              2120 ns/op             528 B/op          4 allocs/op
BenchmarkQueueNativeSync
BenchmarkQueueNativeSync-4        692731              1779 ns/op             368 B/op          4 allocs/op
BenchmarkQueueChannelAsync
BenchmarkQueueChannelAsync-4      220192              5881 ns/op             440 B/op          4 allocs/op
BenchmarkListAsync
BenchmarkListAsync-4              153816              7382 ns/op            1200 B/op         25 allocs/op
BenchmarkCircularAsync
BenchmarkCircularAsync-4          209679              6056 ns/op             544 B/op          5 allocs/op
BenchmarkQueueNativeAsync
BenchmarkQueueNativeAsync-4       215972              5138 ns/op             384 B/op          5 allocs/op
PASS
ok      dmh2000.xyz/queue       11.146s
```

#### Analysis

##### Memory

After running the benchmark tests above, the file mem.out contains information about the memory usage of the benchmarks. This file can be analyzed using the pprof go tool.

```bash
# run the pprof tool with input file mem.out
# (pprof) top10 : shows top 10 locations for memory allocation
$ go tool pprof mem.out
File: queue.test
Type: alloc_space
Time: Mar 17, 2021 at 11:56am (PDT)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top10
Showing nodes accounting for 1998.75MB, 99.24% of 2013.99MB total
Dropped 22 nodes (cum <= 10.07MB)
Showing top 10 nodes out of 21
      flat  flat%   sum%        cum   cum%
  752.53MB 37.37% 37.37%   752.53MB 37.37%  container/list.(*List).insertValue (inline)
  373.59MB 18.55% 55.92%   373.59MB 18.55%  dmh2000.xyz/queue.NewChannelQueue
  314.52MB 15.62% 71.53%   314.52MB 15.62%  sync.NewCond (inline)
  265.57MB 13.19% 84.72%   361.58MB 17.95%  dmh2000.xyz/queue.NewCircularQueue (inline)
  209.53MB 10.40% 95.12%   326.53MB 16.21%  dmh2000.xyz/queue.NewNativeQueue
      45MB  2.23% 97.36%       45MB  2.23%  container/list.New (inline)
      38MB  1.89% 99.24%   184.51MB  9.16%  dmh2000.xyz/queue.NewListQueue
         0     0% 99.24%   752.53MB 37.37%  container/list.(*List).PushBack (inline)
         0     0% 99.24%   752.53MB 37.37%  dmh2000.xyz/queue.(*ListQ).Put
         0     0% 99.24%   106.52MB  5.29%  dmh2000.xyz/queue.BenchmarkCircularAsync
```

##### CPU

```bash
# run the pprof tool with input file cpu.out
# (pprof) top10 : shows top 10 locations for cpu usage
$ go tool pprof cpu.out
File: queue.test
Type: cpu
Time: Mar 17, 2021 at 11:55am (PDT)
Duration: 11.13s, Total samples = 12.39s (111.30%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top10
Showing nodes accounting for 5870ms, 47.38% of 12390ms total
Dropped 138 nodes (cum <= 61.95ms)
Showing top 10 nodes out of 133
      flat  flat%   sum%        cum   cum%
    1380ms 11.14% 11.14%     1380ms 11.14%  sync.(*Mutex).Lock
    1230ms  9.93% 21.07%     1230ms  9.93%  sync.(*Mutex).Unlock
     560ms  4.52% 25.59%      560ms  4.52%  runtime.futex
     510ms  4.12% 29.70%      630ms  5.08%  runtime.heapBitsSetType
     500ms  4.04% 33.74%      500ms  4.04%  runtime.lock2
     460ms  3.71% 37.45%     1900ms 15.33%  runtime.mallocgc
     400ms  3.23% 40.68%      400ms  3.23%  runtime.unlock2
     340ms  2.74% 43.42%      800ms  6.46%  dmh2000.xyz/queue.(*NativeIntQ).Get
     260ms  2.10% 45.52%     4050ms 32.69%  dmh2000.xyz/queue.b1
     230ms  1.86% 47.38%      620ms  5.00%  dmh2000.xyz/queue.(*CircularQ).Get
```