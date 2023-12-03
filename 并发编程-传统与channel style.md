### 术语

1. 关键段（临界区，Critical Section）：涉及共享资源的程序代码。
2. 竞态条件（Race Condition）：多个线程或进程同时访问共享资源导致操作执行顺序不确定，结果不确定
3. 不确定性程序（Indeterminate Program）：多个并发执行的线程或进程之间对共享资源的访问没有得到适当的同步或互斥保护，因此无法确保执行顺序的确定性。
4. 互斥：即保证只有一个线程/进程能访问CS。它有三点要求：
	1. 正确：符合预期
	2. 效率：不浪费资源
	3. 公平：获取资源的机会均等

### 传统方法 

#### 互斥量（Mutex）

互斥量（Mutex），又叫锁，是一种常见实现互斥的机制。基于**CAS**可以实现Mutex。

Mutex可执行的操作为`Lock`与`Unlock`，前者上锁，本质是等待获取资源；后者解锁，本质是释放获取资源的信号。

```go
func main() {
	threadsNum := 100
	loopNum := 500
    share := 0
    finished := make(chan bool)
    race := func() {
        for i := 0; i < loopNum; i++ {
            share++
        }
        finished <- true
    }
	for i := 0; i < threadsNum; i++ {
		go race()
	}
	for i := 0; i < threadsNum; i++ {
		<-finished
	}
    fmt.Println(share)
}
```

这是一个存在RC的程序，因此它的结果一般小于50000。为CS加锁即可解决这一问题。

```go
func main() {
	threadsNum := 100
	loopNum := 500
    share := 0
    finished := make(chan bool)
    mutex := Mutex{} // 初始化锁
    race := func() {
        m.Lock() //上锁
        for i := 0; i < loopNum; i++ {
            share++
        }
        m.Unlock() // 解锁
        finished <- true
    }
	for i := 0; i < threadsNum; i++ {
		go race()
	}
	for i := 0; i < threadsNum; i++ {
		<-finished
	}
    fmt.Println(share)
}
```


锁或互斥量的实质是，标记自身被使用无法获取，直到本线程释放锁之后自身才能被其他线程获取。借助CAS可以轻易编写Mutex。CAS包含三个参数：`addr, old, new`，可以用来实现`Lock`。Mutex包含一个变量，变量`0`表示无人使用资源。`Lock`使用`CAS`原子地检测变量值是否为`0`，若为`0`则返回`true`并赋值为`1`表示开始使用资源，此时其他人调用`Lock`时则无法获取资源。`Unlock`置变量为`1`即可。这种锁称为 **自旋锁**。

```go
type Mutex struct {
	value atomic.Int32
}

func (m *Mutex) Lock() {
	for !m.value.CompareAndSwap(0, 1) { // 自旋
	}
}

func (m *Mutex) Unlock() {
	m.value.Store(0)
}
```

#### 信号量

信号量（Semaphore）是一个计数器，表示可用资源的数量。它提供两种操作：
1. wait（P操作）：当线程希望访问资源时，首先使用P操作。如果信号量大于1，表示有剩余资源，将计数器减一并继续执行；否则线程阻塞。
2. signal（V操作）：当线程使用完资源后，用V操作表示释放了一个资源，使得信号量加1。

这两种操作都是原子操作。

二值信号量只有两个状态，用0与1表示，与互斥量没有实质上的区别。计数信号量可以有多个状态，通常用于控制并发访问的数量。

```go
type Semaphore struct {
	m     sync.Mutex
	state int32
}

type Semaphore struct {
	m     sync.Mutex
	state atomic.Int32
}

func (s *Semaphore) Wait() {
	s.m.Lock() // 借助互斥量实现锁
	defer s.m.Unlock()
	for s.state.Load() == 0 {
	}
	s.state.Add(-1)
}

func (s *Semaphore) Signal() {
	s.state.Add(1)
}

func main() {
	threadsNum := 100
	loopNum := 500
    share := 0
    finished := make(chan bool)
    semaphore := Semaphore{state: atomic.Int32{}}
	semaphore.state.Store(1)
    race := func() {
        semaphore.Wait()
        for i := 0; i < loopNum; i++ {
            share++
        }
        semaphore.Signal()
        finished <- true
    }
	for i := 0; i < threadsNum; i++ {
		go race()
	}
	for i := 0; i < threadsNum; i++ {
		<-finished
	}
    fmt.Println(share)
}
```

上文代码中使用了二值信号量来代替互斥量，此时二者的使用并无差别，只不过`state`的涵义不同（恰恰相反）。

计数信号量则有更加丰富的应用场景。

#### 线程安全的队列

下面实现一个线程安全的队列。

```go
type Queue interface {
	Push() int
	Pop() int
}
```

所谓线程安全的容器是指在多线程环境中依然能安全进行并发读写操作的数据结构。对于一个线程不安全的队列，操作时必须要进行加锁等处理。

```go
type UnconcurrentQueue struct {
	inner []int
}

func (q *UnconcurrentQueue) Push(val int) {
	q.inner = append(q.inner, val)
}

func (q *UnconcurrentQueue) Pop() (int, bool) {
	if len(q.inner) == 0 {
		return 0, false
	}
	val := q.inner[0]
	q.inner = q.inner[1:]
	return val, true
}
```

对不安全的容器直接进行多线程读写可能导致多种异常，例如数据丢失或覆盖，这是由于多个线程同时写入导致的。在以下代码中可以看出这一问题。

```go
func produce(q Queue, num int, finished chan bool) {
	for i := 0; i < num; i++ {
		q.Push(i)
	}
	finished <- true
}

// several producer
func main() {
	q := UnconcurrentQueue{inner: make([]int, 0)}
	finished := make(chan bool)
	for i := 0; i < producerNum; i++ {
		go produce(&q, singleProduce, finished)
	}
	for i := 0; i < producerNum; i++ {
		<-finished
	}
	fmt.Printf("Elements in queue: %v\n", len(q.inner))
}
```

运行代码可以看到队列中的元素小于总共生产的元素。

要获得一个线程安全的队列，最简单的方法就是在方法中上锁。

```go
type ConcurrentQueue struct {
	m     sync.Mutex
	inner []int
}

func (q *ConcurrentQueue) Push(val int) {
	q.m.Lock()
	q.inner = append(q.inner, val)
	q.m.Unlock()
}
```

此时重新运行上文中的代码，即可看到队列中的元素数量等于所有协程生成的元素数量。

但是，跨线程的队列常常需要在出队时满足若队伍中无元素则阻塞。假如此时仍仅在方法中上锁就会如下所示。

```go
func (q *ConcurrentQueue) Pop() int {
	q.m.Lock()
	defer q.m.Unlock()
	for len(q.inner) == 0 {
	}
	val := q.inner[0]
	q.inner = q.inner[1:]
	return val
}
```

由于已经对`q`上锁，故队列为空时必进入死循环。或者，也可以将上锁与解锁步骤写在循环内部，但这对资源的消耗是非常大的。

此时需要使用信号量。信号量本身是一个计数器，可以限制只允许在信号量大于0时出队。此时与前文的区别是，即使`Pop`方法停留在P操作出，`Push`方法仍然可以正常获取锁并写入数据。

```go
type ConcurrentQueue struct {
	m     sync.Mutex
	items Semaphore
	inner []int
}

func (q *ConcurrentQueue) Push(val int) {
	q.m.Lock()
	defer q.m.Unlock()
	q.inner = append(q.inner, val)
	q.items.Signal()
}

func (q *ConcurrentQueue) Pop() int {
	q.items.Wait()
	q.m.Lock()
	defer q.m.Unlock()
	val := q.inner[0]
	q.inner = q.inner[1:]
	return val
}
```

但是信号量仍有缺点。假如这个队列具备一些其他复杂的操作，如`Flush`，清除队列中所有内容。

```go
func (q *ConcurrentQueue) Flush() {
	q.m.Lock()
	defer q.m.Unlock()
	q.inner = make([]int, 0)
	q.items.Flush()
}
```

此时可能导致其他问题：当`Pop`运行完`Wait`，未执行`Lock`时，可能调用`Flush`，接下来`Pop`再获取锁并允许，便会导致访问空数组。
### 条件变量（Condition Variable）


