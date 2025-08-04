# Iterotor trait
## 什么是迭代器？  
迭代器是Rust中一种提供**序列化**访问集合元素的 trait，它允许你依次处理集合中的每个元素而不需要显式地管理索引。

## 为什么要使用迭代器？
1. 更简洁的代码  
迭代器允许你用声明式的方式表达数据处理逻辑，避免了显式的循环和索引管理
2. 更安全的代码
迭代器消除了许多常见的错误来源：  
没有越界访问风险（因为不需要手动管理索引）  
不会意外引入无限循环  
所有权系统确保内存安全
3. 更好的性能  
迭代器经常比手写循环更快，因为：   
Rust编译器可以更好地优化迭代器链  
消除了边界检查（迭代器内部知道何时停止）  
提供了零成本抽象（没有运行时开销）
4. 惰性求值  
迭代器是惰性的，只有在消费者请求时才计算值，这可以：  
避免不必要的计算  
处理无限序列  
提高内存效率（不需要存储中间结果）  
5. 组合性  
迭代器可以轻松组合成复杂的数据处理管道：  
每个适配器返回一个新迭代器  
可以表达复杂的数据转换  
易于维护和修改  
etc....

## 什么时候用到迭代器？

## 迭代器三要素
迭代器(Iterator): 负责产生值的类型

消费者(Consumer): 使用迭代器产生的值

适配器(Adapter): 转换迭代器为另一种迭代器

## 核心trait：Iterator
Rust迭代器的核心是std::iter::Iterator trait：

```rust
pub trait Iterator {
    type Item;  // 迭代器产生的值的类型
    
    fn next(&mut self) -> Option<Self::Item>;
    
    // 提供了许多默认方法...
}
```

## 迭代器创建
iter(): 产生不可变引用

iter_mut(): 产生可变引用  

into_iter(): 获取所有权并产生值

**其他创建方式**  
a..b: 范围迭代器  

some_vec.into_iter(): 转换为迭代器

## 适配器
### map
**定义**  
接收一个闭包（closure），将这个闭包应用到迭代器的每个元素上，产生一个新的迭代器。
**签名**  
```rust
fn map<B, F>(self, f: F) -> Map<Self, F>
where
    Self: Sized,
    F: FnMut(Self::Item) -> B,
```
1. 大多数适配器会按值接受self，这就要求Self必须是固定的Sized(所有常见的迭代器都是这样)   
2. 签名中函数实际返回虽然是Map<Self, F>（std::iter::Map），也就一个（隐藏了实现细节的）struct类型。但是我们只关心他最终返回了能生成给定类型条目的Iterator，因为传入的闭包返回B，则函数返回条目类型为B的迭代器，即-> impl Iterator<Item=B> 。
3. 闭包参数为按值传递，因为是函数核心是转换，通常需要获取所有权。

**基本用法**
```rust
// 简单转换
let numbers = vec![1, 2, 3];
let squares: Vec<_> = numbers.iter().map(|x| x * x).collect();
// squares = [1, 4, 9]

// 类型转换
let strings = vec!["1", "2", "3"];
let numbers: Vec<i32> = strings.iter().map(|s| s.parse::<i32>().unwrap()).collect();
// numbers = [1, 2, 3]
```
### filter
**定义**  
filter是Rust迭代器中用于筛选元素的重要方法，它允许你根据条件选择保留或丢弃迭代器中的元素。  
**签名** 
```rust
fn filter<P>(self, predicate: P) -> Filter<Self, P>
    where
        Self: Sized,
        P: FnMut(&Self::Item) -> bool,
```
1. 类似map
2. 闭包返回值为bool，true保留，false丢弃
3. 函数返回值则返回条目类型与原来一样Self::Item的迭代器，-> impl Iterator<Iter=Self::Item>
4. 闭包参数为不变引用，是为了保留所有权，允许过滤后根据选定条目继续使用原集合

**基本用法**  
```rust
let numbers = vec![1, 2, 3, 4, 5];
let evens: Vec<_> = numbers.iter().filter(|&x| x % 2 == 0).collect();
// evens = [2, 4]
```
### flatten
**定义**   
flatten是Rust迭代器API中用于处理嵌套迭代器结构的重要方法，它可以将多层嵌套的迭代器"展平"为单层迭代器。  
**签名** 
```rust
fn flatten(self) -> Flatten<Self>
    where
        Self: Sized,
        Self::Item: IntoIterator,
```  
1. 当前迭代器self的条目必须实现IntoIterator特征，才能展平。  
2. -> impl Iterator<Item=Self::Item::Item>

**基本用法**   
```rust  
// 展平Vec的Vec
let nested = vec![vec![1, 2], vec![3, 4]];
let flat: Vec<_> = nested.into_iter().flatten().collect();
// flat = [1, 2, 3, 4]

// 展平Option
let options = vec![Some(1), None, Some(3)];
let values: Vec<_> = options.into_iter().flatten().collect();
// values = [1, 3]

// 展平Result
let results = vec![Ok(1), Err("error"), Ok(3)];
let ok_values: Vec<_> = results.into_iter().flatten().collect();
// ok_values = [1, 3]
```
### flat_map
**定义**   
flat_map是Rust迭代器中一个功能强大的方法，它结合了map和flatten两种操作，用于处理嵌套的迭代器结构。先映射map后展平flat。  

**签名**   
```rust
fn flat_map<U, F>(self, f: F) -> FlatMap<Self, U, F>
    where
        Self: Sized,
        U: IntoIterator,
        F: FnMut(Self::Item) -> U,
```
1. -> impl Iterator<Item=U::Item>
2. 最终会生成闭包返回的序列U串联的结果  

**基本用法**   
```rust
// 简单展平
let words = ["hello", "world"]; 
let letters: Vec<_> = words.iter()               //-->iter("hello"), iter("world")
                        .flat_map(|s| s.chars()) //-->['h', 'e', 'l', 'l', 'o'], ['w', 'o', 'r', 'l', 'd'] 
                                                 //-->['h', 'e', 'l', 'l', 'o', 'w', 'o', 'r', 'l', 'd']
                        .collect();
// letters = ['h', 'e', 'l', 'l', 'o', 'w', 'o', 'r', 'l', 'd']

// 处理嵌套结构
let nested = vec![vec![1, 2], vec![3, 4]];
let flat: Vec<_> = nested.into_iter().flat_map(|inner| inner).collect();
// flat = [1, 2, 3, 4]
```
### enumerate
**定义** 
enumerate是Rust迭代器中一个非常有用的方法，它为迭代器的每个元素添加索引，形成一个(index, value)的元组序列。  

**签名**   
```rust
fn enumerate(self) -> Enumerate<Self>
    where
        Self: Sized,
```
**基本用法**
```rust
let fruits = ["apple", "banana", "cherry"];
for (index, fruit) in fruits.iter().enumerate() {
    println!("Fruit {}: {}", index, fruit);
}
// 输出:
// Fruit 0: apple
// Fruit 1: banana
// Fruit 2: cherry
```

### zip
**定义**   
zip是Rust迭代器中用于将两个迭代器"压缩"在一起的重要方法，它会将两个迭代器的元素一一配对，生成一个新的元组迭代器。  
**签名**   
```rust
fn zip<U>(self, other: U) -> Zip<Self, U::IntoIter>
    where
        Self: Sized,
        U: IntoIterator,
```
1. 一一对应，生成一个元组。两个迭代器中任何一个结束时，整个过程就结束了，多余的舍弃

**基本用法**    
```rust
// 简单配对
let names = ["Alice", "Bob"];
let ages = [30, 25];
let zipped: Vec<_> = names.iter().zip(ages.iter()).collect();
// zipped = [(&"Alice", &30), (&"Bob", &25)]

// 不同类型配对
let letters = 'a'..='c';
let numbers = 1..;
let zipped: Vec<_> = letters.zip(numbers).collect();
// zipped = [('a', 1), ('b', 2), ('c', 3)]
```

## 消费者
通过带有for循环的迭代器或者显示调用next，可以消耗迭代器。但是一些常见任务不必一遍又一遍重复写出来，下面提供了一些可选方法来涵盖一些常见任务。  
### count/sum/product
1. count方法会从迭代器中提取条目，直到迭代器返回None，并报告提取的条目数。
2. sum方法计算迭代器条目之和。
3. product方法计算迭代器条目的乘积。
 
### min/max
使用前，必须实现std::cmp::Ord，这样条目之间才能相互比较。  
如f32和f64仅实现std::cmp::PartialOrd，所以不能使用min和max。
1. min返回迭代器生成的最小条目
2. max返回迭代器生成的最大条目

返回值为Option<Self::Item>，以便迭代器不再生成任何条目时能返回None。

### fold
**定义**   
fold方法接收一个初始值（accumulator）和一个闭包，闭包会对每个元素执行操作并更新累加器，最终返回最终的累加值。  
**签名**   
```rust
fn fold<Acc, G>(self, init: Acc, g: G) -> Acc
    where
        G: FnMut(Acc, Self::Item) -> Acc,
```
**基本用法** 
```rust
// 求和
let numbers = [1, 2, 3, 4];
let sum = numbers.iter().fold(0, |acc, &x| acc + x);
// sum = 10

// 字符串连接
let words = ["hello", "world", "rust"];
let sentence = words.iter().fold(String::new(), |mut acc, &s| {
    if !acc.is_empty() {
        acc.push(' ');
    }
    acc.push_str(s);
    acc
});
// sentence = "hello world rust" 
```
### collect
**定义**   
collect方法消费整个迭代器，将其元素收集到指定的集合类型中。可以收集到多种实现了FromIterator trait的类型。  
**签名**   
```rust
fn collect<B: FromIterator<Self::Item>>(self) -> B
    where
        Self: Sized,
```

**基本用法**   
```rust
// 收集到Vec
let doubled: Vec<_> = (1..5).map(|x| x * 2).collect();
// doubled = [2, 4, 6, 8]

// 收集到HashSet
use std::collections::HashSet;

let unique: HashSet<_> = vec![1, 2, 2, 3].into_iter().collect();
// unique = {1, 2, 3}
```