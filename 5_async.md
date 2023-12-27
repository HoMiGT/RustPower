# async 函数和块(代码片段)
> async函数
> ```Rust
> async fn foo(){
> }
> ```
> async块
> ```Rust
> fn foo(){
>     async {
>
>     }
> }
> fn foo1(){
>     async move{
>
>     }
> }
> ```
> 用async定义异步代码，用.await驱动执行
# 异步运行时
> 异步运行时是一个库，包含一个响应器(reactor)和一个或多个执行器(executor)
> 1. 异步代码的执行
> 2. 任务的暂停
> 3. 状态的暂存
> 4. 外部事件的监听注册
> 5. 外部信号来了后，唤醒对应的任务，恢复任务状态
> 6. 多个任务间的调度
# tokio异步编程
> 引入依赖Cargo.toml
> ```Toml
> tokio={version="1",features=["full"]}
> ```
> main函数
> ```Rust
> #[tokio::main]
> async fn main(){
>     println!("Hello world");
> }
> ```
> #[tokio::main]的宏修饰的main展开
> ```Rust
> fn main(){
>     tokio::runtime::Builder::new_multi_thread()
>         .enable_all()
>         .build()
>         .unwrap()
>         .block_on(async {
>             println!("Hello world");
>         })
> }
> ```
> 单线程构建 #[tokio::main(flavor="current_thread")]
## tokio组件
> * Runtime设施组件: 可以自由地配置创建基于系统单线程的Runtime和多线程的Runtime
> * 轻量级任务task: 类似Go的Goroutine轻量级栈
> * 异步输入输出: 网络模块net，文件操作模块fs,signal模块，process模块
> * 时间模块: 定时器Interval等
> * 异步场景下的同步原语: channel，Mutex锁等等
> * 在异步环境下执行计算密集型任务的方案spawn_blocking等等。
## tokio生态
> * Hyper: HTTP协议Server和Client的实现
> * Axum: Web开发框架
> * async-graphql: GraphQL开发框架
> * tonic: gRPC框架的Rust实现
## tokio底层机制
> [async book](https://rust-lang.github.io/async-book/)
## 哪些操作需要加.await?
> **涉及到I/O操作的，都可以加**
> [toiko API](https://docs.rs/tokio/1.32.0/tokio/index.html)
## tokio异步并发的数据安全
> * tokio + Arc + Mutex
> ```Rust
> use std::sync::Arc;
> use tokio::sync::Mutex;
>
> #[tokio::main]
> async fn main() {
>    let db: Vec<u32> = vec![1,2,3,4,5,6,7,8,9,10];
>    let arc_db = Arc::new(Mutex::new(db));  // 加锁
>    let arc_db2 = arc_db.clone();
>    let arc_db3 = arc_db.clone();
>
>    let task_a = tokio::task::spawn(async move {
>        let mut db = arc_db.lock().await;  // 获取锁
>        db[4] = 50;
>        assert_eq!(db[4], 50);             // 校验值
>    });
>    let task_b = tokio::task::spawn(async move {
>        let mut db = arc_db2.lock().await;  // 获取锁
>        db[4] = 100;
>        assert_eq!(db[4], 100);            // 校验值
>    });
>    _ = task_a.await.unwrap();
>    _ = task_b.await.unwrap();
>
>    println!("{:?}", arc_db3.lock().await);  // 获取锁
> }
> // 输出
> // [1, 2, 3, 4, 100, 6, 7, 8, 9, 10]
> ```
> * tokio + RwLock
> ```Rust
> use tokio::sync::RwLock;
> #[tokio::main]
> async fn main(){
>     let lock = RwLock::new(5);
>     {
>         let r1 = lock.read().await;
>         let r2 = lock.read().await;
>         assert_eq!(*r1,5);
>         assert_eq!(*r2,5);
>     }
>
>     {
>         let mut w = lock.write().await;
>         *w +=1;
>         assert_eq!(*w,6);
>     }
> }
> ```
> * tokio + Arc + atomic
> ```Rust
> use std::sync::atomic::AtomicU32;
> fn main(){
>     let atomic_forty_two = AtomicU32::new(42);
>     let arc_data = Arc::new(atomic_forty_two);
>     let mut some_var = AtomicU32::new(10);
>     *some_var.get_mut() = 5;
>     assert_eq!(*some_var.get_mut(),5);
> }
> ```
> [Rust Atomics and Locks](https://marabos.nl/atomics/)
> * tokio + mpsc
> ```Rust
> use tokio::sync::mpsc;
> #[tokio::main]
> async fn main(){
>     let mut db:Vec<u32> = vec![1,2,3,4,5,6,7,8,9,10];
>     let (tx,mut rx) = mpsc::channel::<u32>(100);
>     let tx1 = tx.clone();
>     let tx2 = tx.clone();
>     let task_a = tokio::task::spawn(async move {
>         if let Err(_) = tx1.send(50).await{
>             println!("receiver dropped");
>             return;
>         }
>     });
>     let task_b = tokio::task::spawn(async move {
>         if let Err(_) = tx2.send(100).await{
>             println!("receiver dropped");
>             return;
>         }
>     });
>     let task_c = tokio::task::spawn(async move{
>         while let Some(i) = rx.recv().await{
>             println!("got = {}",i);
>             db[4] = i;
>             println!("{:?}",db);
>         }
>     });
>     _ = task_a.await.unwrap();
>     _ = task_b.await.unwrap();
>     _ = task_c.await.unwrap();
> // 输出
> // Got = 50
> // [1, 2, 3, 4, 50, 6, 7, 8, 9, 10]
> // got = 100
> // [1, 2, 3, 4, 100, 6, 7, 8, 9, 10]
> // ^C
> }
> ```
> * tokio + Oneshot + mpsc
> ```Rust
> //! Oneshot 通道只能用一次，只发送一条数据，发送完就关闭了，tx和rx就无法再次使用了，很适合等待计算结果返回的场景
> use std::time::Duration;
> use tokio::sync::{mpsc,oneshot};
> use tokio::task;
> use tokio::time;
> #[tokio::main]
> async fn mian(){
>     let mut db:Vec<u32> = vec![1,2,3,4,5,6,7,8,9,10];
>     let (tx,mut rx) = mpsc::channel::<(u32,oneshot::Sender<bool>)>(100);
>     let tx1 = tx.clone();
>     let tx2 = tx.clone();
>     let task_a = task::spawn(async move {
>         time::sleep(Duration::from_secs(3)).await;
>         let (resp_tx,resp_rx) = oneshot.channel();
>         if let Err(_) = tx1.send((50,resp_tx)).await{
>             println!("receiver dropped");
>             return;
>         }
>         if let Ok(ret) = resp_rx.await{
>             if ret{
>                 println!("task_a finished with success.");
>             }else{
>                 println!("task_a finished with failure");
>             }
>         }else{
>             println!("oneshot sender dropped");
>             return;
>         }
>     });
>     let task_b = task::spawn(async move {
>         let (resp_tx,resp_rx) = oneshot.channel();
>         if let Err(_) = tx2.send((100,resp_tx)).await{
>             println!("receiver dropped");
>             return;
>         }
>         if let Ok(ret) = resp_rx.await{
>             if ret{
>                 println!("task_b finished with success.");
>             }else{
>                 println!("task_b finished with failure");
>             }
>         }else{
>             println!("oneshot sender dropped");
>             return;
>         }
>     });
>     let task_c = task::spawn(async move{
>         while let Some((i,resp_tx)) = rx.recv().await{
>             println!("got = {}",i);
>             db[4] = i;
>             println!("{:?}",db);
>             resp_tx.send(true).unwrap();
>         }
>     });
>     _ = task_a.await.unwrap();
>     _ = task_b.await.unwrap();
>     _ = task_c.await.unwrap();
> // 输出
> // got = 100
> // [1, 2, 3, 4, 100, 6, 7, 8, 9, 10]
> // task_b finished with success.
> // got = 50
> // [1, 2, 3, 4, 50, 6, 7, 8, 9, 10]
> // task_a finished with success.
> // ^C
> } 
> ```
> * tokio + broadcast channel(MPMC)
> 广播模式，实现了MPMC，即多生产者多消费者，可以用来实现发布-订阅模式。每个消费者都会收到每个生产者发出的同样的消息副本。
> [tokio 链接](https://docs.rs/crate/tokio/1.33.0)
> * tokio + watch channel
> 特定化的broadcast通道，只有一个生产者，多个消费者。只关心最后一个值。
> [tokio 链接](https://docs.rs/crate/tokio/1.33.0)
> * tokio获取任务执行的结果
>   * 阻塞,花费时间=任务执行花费最长的时间
> ```Rust
> use tokio::task;
> async fn my_bg_op(id:i32) -> String{
>     let s = format!("String backgroud task {}.",id);
>     println!("{}",s);
>     s
> }
> #[tokio::main]
> async fn main(){
>     let ops = vec![1,2,3];
>     let mut tasks = Vec::with_capacity(ops.len());
>     for op in ops{
>         tasks.push(tokio::spawn(my_bg_op(op)));
>     }
>     let mut outputs = Vec::with_capacity(tasks.len());
>     for task in tasks{
>         outputs.push(task.await.unwrap());
>     }
> }
> ```
> ```Rust
> /// tokio::join!()等待所有任务完成，一起返回结果
> use std::time::Duration;
> use tokio::task;
> use tokio::time;
> #[tokio::main]
> async fn main(){
>     let task_a = task::spawn(async move{
>         println!("in task_a");
>         time::sleep(Duration::from_secs(3)).await; // 等待3s
>         1
>     });
>     let task_b = task::spawn(async move{
>         println!("in task_b");
>         2
>     });
>     let task_c = task::spawn(async move{
>         println!("in task_c");
>         3
>     });
>     let (r1,r2,r3) = tokio::join!(task_a,task_b,task_c);
>     println!("{}, {}, {}",r1.unwrap(),r2.unwrap(),r3.unwrap());
> // 输出
> // in task_a
> // in task_b
> // in task_c
> // 1,2,3
> }
> ```
     * 优先一个任务先返回
> ```Rust
> /// 在一批任务中，哪个任务先执行完，就马上返回那个任务的结果，剩下的任务，要么不关心它们的执行结果，要么直接取消它们执行， 使用tokio::select!() 
> use std::time::Duration;
> use tokio::task;
> use tokio::time;
> #[tokio::main]
> async fn main(){
>     let task_a = task::spawn(async move{
>         println!("in task_a")
>         time::sleep(Duration::from_secs(3)).await;
>         1
>     });
>     let task_b = task::spawn(async move{
>         println!("in task_b");
>         2
>     });
>     let task_c = task::spawn(async move{
>         println!("in task_c");
>         3
>     });
>     let ret = tokio::select!{
>         r = task_a => r.unwrap(),
>         r = task_b => r.unwrap(),
>         r = task_c => r.unwrap(),
>     }
>     println!("{}",ret);
> // 输出
> // 第一次
> // in task_b
> // in task_a
> // 2
> // in task_c
> // 第二次
> // in task_a
> // in task_c
> // in task_b
> // 2
> // 第n次
> // in task_a
> // in task_c
> // in task_b
> // 3
> }
> ```
