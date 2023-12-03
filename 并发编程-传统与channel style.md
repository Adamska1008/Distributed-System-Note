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

互斥量（Mutex），又叫锁，是一种常见实现互斥的机制。基于CAS可以实现Mutex。

Mutex可执行的操作为`Lock`与`Unlock`，前者上锁，本质是等待获取资源；后者解锁，本质是释放获取资源的信号。

```go
func main() {
    share := 0
    finished := make(chan bool)
    race := func() {
        for i := 1; i <= 100000; i++ {
            share++
        }
        finished <- true
    }

    go race()
    go race()
    for i := 0; i < 2; i++ {
        <-finished
    }

    fmt.Println(share)
}
```

这是一个存在RC的程序，因此它的结果一般小于`200000`。为CS加锁即可解决这一问题。

```go
func main() {
    share := 0
    finished := make(chan bool)
    mutex := Mutex{}
    race := func() {
        mutex.Lock()
        for i := 1; i <= 100000; i++ {
            share++
        }
        mutex.Unlock()
        finished <- true
    }

    go race()
    go race()
    for i := 0; i < 2; i++ {
        <-finished
    }

    fmt.Println(share)
}

```


锁或互斥量的实质是，标记自身被使用无法获取，直到本线程释放锁之后自身才能被其他线程获取。借助CAS可以轻易编写Mutex。CAS包含三个参数：`addr, old, new`，可以用来实现`Lock`。Mutex内涵一个变量，变量`0`表示无人使用资源。`Lock`使用`CAS`原子地检测变量值是否为`0`，若为`0`则返回`true`并赋值为`1`表示开始使用资源，此时其他人调用`Lock`时则无法获取资源。`Unlock`置变量为`1`即可。

```go
type Mutex struct {
    value int32
}

func (m *Mutex) Lock() {
    for !atomic.CompareAndSwapInt32(&m.value, 0, 1) {
    }
}

func (m *Mutex) Unlock() {
    atomic.StoreInt32(&m.value, 0)
}
```

#### 信号量

信号量（Semaphore）是一个计数器，表示可用资源的数量。它提供两种操作：
1. wait（P操作）：当线程希望访问资源时，首先使用P操作。如果信号量大于1，表示有剩余资源，将计数器减一并继续执行；否则线程阻塞。
2. signal（V操作）：当线程使用完资源后，用V操作表示释放了一个资源，使得信号量加1。

二值信号量只有两个状态，用0与1表示，与互斥量没有实质上的区别。计数信号量可以有多个状态，通常用于控制并发访问的数量。

```go
type Semaphore struct {
    value int32
}

func (s *Semaphore) Wait() {
    for atomic.LoadInt32(&s.value) == 0 {
    }
    atomic.AddInt32(&s.value, -1)
}

func (s *Semaphore) Signal() {
    atomic.AddInt32(&s.value, 1)
}

func main() {
    share := 0
    finished := make(chan bool)
    semaphore := Semaphore{value: 1}
    race := func() {
        semaphore.Wait()
        for i := 1; i <= 100000; i++ {
            share++
        }
        semaphore.Signal()
        finished <- true
    }

    go race()
    go race()
    for i := 0; i < 2; i++ {
        <-finished
    }

    fmt.Println(share)
}
```

上文代码中使用了二值信号量来代替互斥量，此时二者的使用并无差别，只不过`value`的涵义不同（恰恰相反）。

计数信号量则有更加丰富的应用场景。

#### 线程安全的队列

下面实现一个线程安全的队列。