# 字符串
> 1. String,&String
> 2. str,&str,&'static str
> 3. [u8],&[u8],&[u8;N],Vec<u8>
> 4. OsStr,OsString
> 5. Path,PathBuf
> 6. CStr,CString

## String,&String
> * String是字符串的所有权形式，常常在堆中分配。String字符串内容大小是可以变动的
## str,&str,&'static str
> * str 是字符串的切片，通常以切片引用&str的形式出现，是字符串视图的借用形式。
> * &'static str 在静态区的字符串的切片引用
>```Rust
>let s1:&'static str = "I am a superman";  // 在静态区
>let s2:String = s1.to_string();  // s2在栈空间，值在堆空间，同时值从静态区拷贝到了堆空间
>let s3:&String = &s2;  // s3在栈空间，是对s2的引用，指向s2
>let s4:&str = &s2[..];  // 在栈空间 对s2视图的引用
>let s5:&str = &s2[..3];  // 在栈空间 对s2视图的引用
>// ***************** &str 与 String的相互转换  *************************
>let s:String = "I am a superman".to_string();   // &str --> String
>let a_slice:&str = &s[..];  // String --> &Str
>let another_string:String = a_slice.to_string();  // &str --> String
>```
>
> 函数参数为 **&str** 时，可以同时接受 **String** 类型和 **&str**
> 
## [u8],&[u8],&[u8;N],Vec<u8>
> | 类型 | 字符串 | 字节串 | 
> | :--:|:--:|:--:| 
> |具有所有权|String|Vec<u8>| 
> |切片|str|[u8]| 
> |切片引用|&str|&[u8]| 
> |所有权到切片引用|&a_string[..]|&a_vec[..]| 
> |切片到所有权|a_slice.to_string()<br>a_slice.to_owned()|a_slice.to_vec()<br>a_slice.to_owned()|
>
> Vec类型的as_slice()的语法等价于&a_vec[..] 
> let a_vec = vec![1,2,3,4,5]; 
> assert_eq!(&[1,2,3,4,5],a_vec.as_slice()); 
>
> 函数参数为 **&[T]** 时, 可以同时接受 **&Vec<T>** 类型和 **&[T]**
>
> 字节串转换字符串
> ```Rust
> /// String::from_utf8()   Vec<u8> --> String
> /// String::from_utf8_unchecked()  Vec<u8> --> String unsafe方法 一般不推荐用
> /// str::from_utf8()  &[u8] --> &str
> /// str::from_utf8_unchecked()  &[u8] --> &str  unsafe方法 一般不推荐用
> let s: String = String::from("中国No.1");
> let char_vec: Vec<char> = s.chars().collect();
> ```
