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
> 
