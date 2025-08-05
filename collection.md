# 集合  

## Vec\<T\>  可增长数组、普通向量
可增长的、分配在堆上的T类型值数组。  
向量拥有3个字段：长度、容量和指向用于存储元素的堆分配内存的指针
### 创建向量
1. vec![]
2. Vec::new()
3. iter.collect::<Vec\<T>>(), 实现了std::iter::FromIterator

### 访问向量中的元素
1. .first()/.first_mut()   
   -> Option<&T>/Option<&mut T>  
   返回对第一个元素的引用/可变引用，为空时返回None
2. .last()/.last_mut()   
   -> Option<&T>/Option<&mut T>   
   返回对最后一个元素的引用/可变引用，为空时返回None
3. .get(index)/.get_mut(index) 
   -> Option<&T>/Option<&mut T>   
   若存在返回对应下表元素的Some值，个数少于index+1时返回None

### 扩大与收缩
```rust
// 返回向量长度 
len() -> usize     

// 向量未包含任何元素（len() == 0）则为真
is_empty() -> bool 

// 将给定value添加到vec的末尾
push(value)   

// 移除并返回最后一个元素，返回Some(T)，为空则None
pop() -> Option(T) 

// 在index处插入元素value  
// vec[index..]中所有的值都往右移动一个位置  
// 若index > vec.len()，则panic
insert(index, value) 

// 移除并返回vec[index]
// vec[index+1..]中所有的值都往左移动一个位置
// index >= vec.len()，则panic
// 向量越长，速度越慢，少用
remove(index) -> T 

// 移除vec中所有元素
clear() 

// 截断，将vec的长度减少到new_len
// 丢弃vec[new_len..]范围内的任何元素
truncate(new_len)

// 类似truncate，但是返回一个Vec<T>
// 包含vec[index..]里的值
split_off(index) -> Vec<T>

// 按顺序在vec的末尾添加给定iterable值的所有条目
extend(iterable)

// 追加，将vec2的所有元素移动到vec中，并清空vec2
append(&mut vec2)

// 抽取，从vec中移除range范围内的切片vec[range]
// 并返回对移除元素的迭代器
drain(range) ->Iterator<Item=T>
```
### 联结
```rust
// 下面两方法针对数组的数组，类似二维数组，如：
// [[1, 2], [3, 4], [5, 6]]
// 即一个数组、切片或向量的元素也是数组、切片或向量

// 串联
// [[1, 2], [3, 4], [5, 6]] ->
// [1, 2, 3, 4, 5, 6]
concat() 

// 联结
// 如join(&0), [[1, 2], [3, 4], [5, 6]] ->
// [1, 2, 0, 3, 4, 0, 5, 6]
join(&separator)
```
### 拆分
```rust
// 同时获得多个对向量中元素的不可变引用是比较容易的
// 但是获得多个可变引用就不容易了，因为不可存在对同一向量v的多个可变引用
// 提供了不少split开头的函数用来拆分，可以存在多个可变借用
// 因为它们把数据拆分成了几个不重叠的区域
// 但是这类函数还没见过使用，暂不记录
// TODO


// 滑动窗口
// 返回一个行为类似于slice中数据的滑动窗口的迭代器
// 第一个值&slice[0..n]，第二个值&slice[1..n+1]，以此类推
// 如果n大于len()，则不会生成任何切片。若n=0，则panic
windows(n)
```
### 交换
```rust
// 交换vec[i]和vec[j]
swap(i, j)

// 交换vec和vec2所有内容，且vec和vec2长度必须相同
swap_with_slice(vec2)

// 移除并返回vec[i], 与remove(i)类似。
// 但是不会左移，而是将最后一个元素移动到空缺位置
// 性能更高，如果不关心顺序。
swap_remove(i) -> T
```
### 排序和搜索
```rust
// 升序排列，T:Ord
sort()

// 就地逆转切片
reverse()

// 二分搜索
// 排序完毕后再使用
// 若找到value，则返回Ok(index)，若找不到返回Err(insertion_point)
binary_search(&value) -> Result<usize, usize>

// 未排序时用contain搜索
// 若存在和value相等的元素，返回true
contain(&value) -> bool
```

## VecDeque\<T\> 双端队列  
适合用作先入先出队列，支持在队列前后高效地添加和移除。代价是其他操作会慢一些。
```rust
// 队首推入
push_front(value)

// 队尾推入
push_back(value)

// 队首弹出，移除并返回队列的首端值
pop_front()  -> Option<T>

// 队尾弹出，移除并返回队列的尾端值
pop_back()   -> Option<T>

// 返回队首/队尾元素的引用 
front()/back()  -> Option<&T>

// 可变引用
front_mut()/back_mut()  -> Option<&mut T>

// 将双端队列变成向量
// 因Vec<T> 实现了From<VecDeque<T>>
// 时间复杂度为O(n)，因为可能要重新排列元素
Vec::from(deque)

// 将向量变成双端队列
// 因VecDeque<T>也实现了Vec<T>
// 时间复杂度O(1)，因为Rust直接把向量缓冲区转移给VecDeque，而不会重新分配
VecDeque::from(vec)
```

## BinaryHeap\<T\> 二叉堆
优先级队列，高效地查找，以及移除最大值。
T:Ord就是最大堆。

```rust 
// 向堆中添加一个值
push()

// 从堆中移除并返回最大值，堆为空则返回None
pop() -> Option<T>

// 返回对堆中最大值的引用
peek() -> Option<&T>

// 支持Vec的部分方法
BinaryHeap::new()
len()/is_empty()/capacity()/clear()/append(&mut heap2)
```

## HashMap\<K, V> 
键-值对构成的表，由键查找值很快。条目Entry存储顺序任意。

```rust
// 创建新的空Map
HashMap::new()

// 从键-值对创建和填充新的HashMap 
iter.collect()

// 返回条目数/没有条目返回true
len()/is_empty()

// 具有给定key的条目，则返回true
contains_key(&key)

// 搜索给定key的条目，若找到匹配条目
// 返回Some(r)，r是对应值value的引用，反之返回None
get(&key)/get_mut(&key) -> Option<&V>/Option<&mut V>

// 插入(key, value)条目并返回旧值old_value(如果有的话)。
insert(key,value) -> Option<V>

// 清空
clear()
```
### 条目

```rust
// Entry条作用旨在消除冗余的Map查找

// 返回给定key的条目Entry。
// 如果map没有这样的键，返回一个空条目
map.entry(key)

// or_insert用来处理空条目
// 插入具有给定value的新条目
// 返回对新值或现有值的可变引用
map.entry(key).or_insert(value) -> &mut V

// 修改现有字段的便捷方法
// 若存在具有key的条目，则调用closure，并传入对该值value的可变引用
// 返回Entry，因此可以与其他方法做链式调用
map.entry(key).and_modify(closure) -> Entry
```
### 对map进行迭代

1. 按值迭代(`for (k, v) in map`)以生成`(k, v)`对，消耗map
2. 按不可变引用迭代`for (k, v) in &map`以生成`(&k, &v)`对
3. 按可变引用迭代`for (k, v) in &mut map`以生成`(&k, &mut v)`对。(无法对存储在Map中的键进行可变访问，因为这些条目是通过键进行组织的)

```rust
// 返回只有“键引用”的迭代器
map.keys()

// 返回只有“值引用”的迭代器
map.values()

// 返回只有“值可变引用”的迭代器
map.values_mut()

// 消耗此Map，分别返回遍历键-值对(K, V)、键或值的迭代器
map.into_iter()/into_keys()/into_values()
```

## HashSet\<T\> 
由T类型的值组成的set。很快的添加值和移除值，以及查询给定值是否包含。

```rust 
// 创建新set
HashSet::new()

// 从任意迭代器创建新set。如iter多次生成同一个值，重复值丢弃
iter.collect()

// 返回set值的数量/若set不包含任何元素，返回true
len()/empty()

// 包含，若set包含给定value，返回true
contain(&value)

// 插入value，若value是新增的返回true，set中已有value则false
insert(value) -> bool

// 移除value，若移除移除了一个值则返回true，set中没有value则false
remove(&value) -> bool
```
### 对set进行迭代

1. 按值迭代`for v in set`会生成Set的成员并消耗此Set
2. 按不可变迭代`for v in &set`会生成Set成员的不可变引用  
3. set.iter()  返回set中成员引用的迭代器

`不支持通过可变引用迭代Set`  

### 针对整个Set的运算
```rust 
// 返回同时出现在set1和set2中的值的迭代器
set1.intersection(&set2)   <==> &set1 & &set2

// 返回存在于set1或set2中或者同时存在于两者中的值的迭代器
set1.union(&set2)      <==>  &set1 | &set2

// 返回存在于set1但不在于set2中的值的迭代器
set1.difference(&set2)    <==> &set1 - &set2

// 返回存在于set1或set2中但不同时存在于两者中的迭代器
set1.symmetric_difference(&set2)   <==> &set1 ^ &set2

// 如果set1和set2没有共同的值，返回true，无交集
set1.is_disjoint(set2)

// set1是set2的子集
set1.is_subset(set2)

// set1是set2的超集
set1.is_superset(set2)
```
`还支持==和!=进行相等性测试`