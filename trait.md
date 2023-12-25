# trait
> **trait是对类型参数的约束，引导具体类型实现具体功能**    
> 具体类型：内置的数据类型，用户的自定义枚举和结构体类型，函数类型    
> 类型参数：泛型T
> **trait是约束和能力的体现**
## trait里可以包含***关联函数***，***关联类型***，***关联常量***
> ```Rust
> pub trait A{
> const AConst:i32=3;
>     type Output;
>     fn Afn(&self);
> }
> ```
>
> ```mermaid
> graph LR
> A(变量) --> B(类型<值空间太过宽泛,添加约束>)
> B --> C(泛型-类型参数<约束过死,放开约束>)
> C-->D(trait)  
> ```
### 关联函数
> ```Rust
> trait Sport{
>     fn play(&self) {};  // 花括号表示trait的关联函数的默认实现
>     fn play_mut(&mut self);  
>     fn play_own(self);  
>     fn play_some() -> Self;
> }
> ```
### 关联类型
> * 在T上使用关联类型
> ```Rust
> pub trait Iterator{
>     type Item;
>     fn next(&mut self)->Option<Self::Item>;  // Self::Item 实际是 <Self as Iterator>::Item 
> }
> ```
> * 在约束中具化关联类型
> ```Rust
> trait A{
>     type Item;
> }
> struct Foo<T:A<Item=String>> {x:T}
> struct B;
> impl A for B{
>     type Item=String;
> }
> fn main(){
>     let a = Foo{x:B};
> }
> ```
>
### 关联常量
> ```Rust
> trait A{    
>     const LEN: u32=10;  // 相当于是一个默认值，可以实现，也可以不实现   
> }   
> struct B;   
> impl A for B{   
>     const LEN:u32=14;   
> }    
> fn main(){    
>     println!("{:?}",A::LEN);   
>     println!("{:?}",<B as A>::LEN);  
> }    
> // 输出 14  14
> ```
> <b><B as A>::Len 叫做完成限定语法</b>    

## 约束依赖
> trait 提供了一种语法表示约束间的依赖，**约束之间是完成平等**    
> trait A: B {} // 如果要实现A，那么同时也要实现B。 A为subtrait B为supertrait，在B的约束上，又增加A的约束    

## 一个trait在一个类型上只能被实现一次
> ```Rust
> trait A{}
> struct AType{};
> impl A for AType{}
> impl A for AType{}
> // 编译输出: error[E0119]: conflicting implementations of trait `A` for type `AType`
> ```
>
## trait的孤儿规则
> Rust要求在一个模块中，如果要对一个类型实现某个trait,这个类型和这个trait，其中必须有一个是在当前模块中定义的
> ```Rust
> use std::fmt::Display;
> struct A;
> impl Display for A{}
> // 或者
> trait B{}
> impl B for u32{}
> ```
> 如下情况是不被允许的
> ```Rust
> use std::fmt::Dispaly;
> impl Display for u32{}
> ```
> 如果想实现u32的，可以通过Newtype模式
> ```Rust
> use std::fmt::Display
> struct MyU32(u32);
> impl Display for MyU32{}
> impl MyU32{
>     fn get(&self)->u32{self.0}
> }
> ```
>

## Blanket Implementation 又叫统一实现
> 统一实现后，就不要对具体的类型再实现一次了
> ```Rust
> trait A{}
> trait B{}
> impl<T:B> A for T{} 
> ```

## trait类型参数的默认实现
> trait A<T=u64> {} 使用时 impl A for SomeType {} 等价于 impl A<u64> for SomeType {}
> ```Rust
> // Self可以用在默认类型上
> trait A<T=Self>{
>     fn func(t:T){}
> }
> trait B<T=i32>{
>     fn func2(t:T){}
> }
> struct SomeType;
> impl A for SomeType{
>     fn func(t:SomeType) {}
> }
> impl B for SomeType{
>     fn func2(t:i32) {}
> }
> impl A<String> for SomeType{
>     fn func(t:String) {}
> }
> impl B<String> for SomeType{
>     fn func2(t:String) {}
> }
> ```
## trait类型参数更加通用，关联类型更加具体
## trait object
> 利用trait object 传参
> * impl trait 编译器静态展开，单态化
> ```Rust
> struct A;
> struct B;
> struct C;
> trait Trait{}
> impl Trait for A{}
> impl Trait for B{}
> impl Trait for C{}
> fn doit(x:impl Trait){}
> fn main(){
>     let a = A;
>     doit(a);
>     let b = B;
>     doit(b);
>     let c = C;
>     doit(c);
> }
> ```
> * dyn trait
> ```Rust
> struct A;
> struct B;
> struct C;
> trait Trait{}
> impl Trait for A{}
> impl Trait for B{}
> impl Trait for C{}
> fn doit(x:&dyn Trait){} // 引用形式
> fn main(){
>     let a = A;
>     doit(&a);
>     let b = B;
>     doit(&b);
>     let c = C;
>     doit(&c);
> }
> ```
> **impl trait和dyn trait都是消除类型参数的办法**
> trait object 与 enum 的区别在于 一个是开放型类型集，另一个是封闭型类型集
>
> * 利用trait object将不同的数据集装进集合里
> ```Rust
> struct A;
> struct B;
> struct C;
> trait Trait{}
> impl Trait for A{}
> impl Trait for B{}
> impl Trait for C{}
> fn main(){
>     let a = A;
>     let b = B;
>     let c = C;
>     let v:Vec<&dyn Trait> = vec![&a,&b,&c];
> }
> ```

## 哪些trait能用作trait object
> 只有满足对象安全的trait才可用作trait object
> * 安全的trait object
> ```Rust
> trait ObjectSafe{
>     fn foo(&self) {}
>     fn foo_mut(&mut self) {}
>     fn foo_box(self:Box<Self>) {}
> }
> ```
> * 不安全的trait object
> ```Rust
> trait UnObjectSafe{
>     const Const:i32 = 1; //不能包含关联常量
>     fn foo() {}  // 不能包含这样的关联函数
>     fn selfin(self);  // 不能将self的所有权传入
>     fn return(&self) ->Self;  // 不能返回self
>     fn typed<T>(&self,x:T) {} // 方法中不能有类型参数
> }
> ```
> 场景：1. 不要在trait里面定义构造函数；2. trait里面尽量定义传引用&self或&mut self的方法，不要定义传值self的方法

# 标准库的trait

