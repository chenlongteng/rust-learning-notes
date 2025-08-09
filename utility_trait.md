# 实用工具特征
熟悉这些常见特征，才能写出符合Rust语言惯例的代码并据此为你的crate设计公共接口，让用户认为这些接口是符合Rust风格的。

特征 | 描述
--- | ---
Drop | 析构器。每当丢弃一个值时，Rust都要自动运行的清理代码
Sized | 具有在编译器已知的固定大小类型的标记特征，与之相对的是动态大小类型(如切片)
Clone | 用来支持克隆值的特征
Copy | 可以简单地通过对包含值内存进行逐字节复制以进行克隆的类型的标记特征
Deref与DerefMut | 只能指针类型的特征
Default | 具有合理“默认值”的类型
AsRef与AsMut | 用于从另一种类型中借入一种引用类型的转换特征
Borrow与BorrowMut | 转换特征，类似AsRef/AsMut，但能额外保证一致的哈希、排序和相等性
From与Into | 用于将一种类型的值转换为另一种类型的转换特征
TryFrom与TryInto | 用于将一种类型的值转换为另一种类型的转换特征，用于可能失败的转换
ToOwned | 用于将引用转换为拥有型值的转换特征

另还有Iterator/IntoIterator，Hash，Send/Sync等特征，其他文件中介绍

## Drop
```rust
trait Drop{
    fn drop(&mut self);
}
```
当一个值的拥有者消失时，Rust会丢弃该值。丢弃一个值就必须释放该值拥有的任何其他值、堆存储好系统资源。  
丢弃可能发生在多种情况下：  
1. 变量超出作用域
2. 在表达式语句的末尾
3. 当截断一个向量时，会从其末尾移除元素  

etc  

大多数情况下，Rust会自动处理丢弃值的工作。  
也可以通过实现std::ops::Drop特征来自定义Rust该如何丢弃此类型的值。  
若定义了std::ops::Drop，那么Rust就会先调用Drop::drop方法，再像往常一样继续丢弃它的字段或元素拥有的任何值。  
由于Drop::drop先调用，所以drop方法里也能随意使用拥有字段。  
一个类型实现了Drop特征，就不能在实现Copy特征了。  

标准库还定义了一个drop函数(非Drop特征中的成员):
```rust
fn drop<T>(_x :T) {}
```
非常简单，传入任意值，然后什么也不做。以为值的所有权移动到drop里，drop函数结束超出值的作用域则触发丢弃  


## Sized
`固定大小类型` 是指其每个值在内存中都有相同大小的类型。
Rust自动为所有适用的类型事项了std::marker::Sized特征。Sized的唯一用途是作为类型变量的限界。

`无固定大小类型` 如`[T]`切片就是不固定大小的，字符串字面量`&str`中的`str`也是不固定大小，但是`&str`对不固定的`str`切片的引用是固定大小的。另一种就是`dyn`类型，它是特征对象的引用目标，对应的用`&dyn trait`和`Box<dyn trait>`指向实现了trait特征的某个值的指针。  

Rust不能将无固定大小的值存储在变量中或将它们作为参数传递。只能通过像&str或Box\<dyn trait>这样的本身是固定大小的指针来处理它们。指向无固定大小的指针始终是一个`胖指针`，宽度为`两个机器字`：指向切片的指针带有切片的长度，特征对象带有指向方法实现的虚表的指针。  

`Sized`已经成为Rust中的隐式默认值，若不需要，需显示指出`T : ?Sized`  

第三种无固定大小类型是结构体的最后一个字段且只能是最后一个可以是无固定大小的，这样结构体也会是无固定大小。如`Rc<T>`引用计数指针的内部实现是指向私有类型`RcBox<T>`:
```rust
struct RcBox<T: ?Sized> {
    ref_count: usize,
    value: T,
}
```
## Clone
`std::clone::Clone`特征适用于可复制自身的类型。Clone定义如下：
```rust
trait Clone: Sized {
    fn clone(&self) -> Self;
    fn clone_from(&mut self, source: &self) {
        *self = source.clone();
    }
}
```
clone在时间消耗和内存占用方面都是相当昂贵。例如，克隆Vec\<String>不仅会复制此向量，还会复制他的每个String元素。这就是Rust不会自动克隆值，而是要求你进行显示方法调用的原因。但是Rc\<T>和Arc\<T>引用计数指针类型例外，克隆其中任何一个都只会增加引用计数并为你返回一个新指针。  

clone_from方法会把self修改成source的副本。clone_from的默认定义只是克隆source，然后转移给*self。但是对于某些类型，有更快的方法。  
假设s和t都是String。s = t.clone(); 语句必然会克隆t，丢弃s的旧值，然后将克隆后的值转移给s，其中会进行一次堆分配和一次堆释放。  
但是如果属于原始s的堆缓冲区有足够的容量来保存t的内容，就不需要分配或释放：可以简单地将t的文本复制到s的缓冲区并调整长度。在泛型代码中，应该优先使用clone_from。

## Copy
`std::marker::Copy`  
对于大多数类型，赋值时会移动值，而不是赋值它们。不拥有任何资源的简单类型可以使Copy类型，对这些简单类型赋值会创建值的副本，而不会移动值使源回到未初始化状态。  
只有当类型需要一个浅层的逐字节复制时，Rust才允许它实现Copy。  
拥有任何其他资源（比如堆缓冲区或操作系统句柄）的类型都无法实现Copy。
任何实现Drop特征的类型都不能是Copy。Rust认为如果一个类型需要特殊的清理代码，那么就必然需要特殊的复制代码。


## Deref与DerefMut
实现`std::ops::Deref`和`std::ops::DerefMut`,可以指定像*和.这样的解引用运算符在你的类型上的行为。  
如果上下文对引用目标进行了赋值或借用了可变引用，那么Rust就会使用DerefMut(解可变引用)特征。
```rust
trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}

trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```
`隐式解引用`，由于deref会接受&self引用并返回&self::target引用，因此Rust会利用这一点自动将前一种类型的引用转换为后一种类型的引用。换句话说，如果只要插入一个deref调用就能解决类型不匹配问题，Rust就会插入它。实现DerefMut也可以为可变引用启用相应的转换。  

必要情况下，Rust会连续应用多个隐式解引用。如str切片类型的split_at方法，可以直接应用于Rc\<String>，因为Rc\<T>实现Deref\<Target=T>所以&Rc\<String>解引用成了&String，后者又因为String实现了Deref\<Target=str>所以也可以解引用成&str，而&str具有split_at方法。  

## Default
显然，某些类型具有合理的默认值：向量或字面量默认为空、数值默认为0、Option默认为None，等等。这样的类型都可以实现`std::default::Default`特征：
```rust
trait Default {
    fn default() -> self;
}
```

Rust的所有集合类型（Vec、HashMap、BinaryHeap等）都实现了Default，其Default方法会返回一个空集合。当你需要构建一些值的集合但又想让调用者来决定具体构建何种集合时，很有用。  
Default的另一个常见用途是为表示大量参数集合的结构体生成默认值，其中大部分参数通常不用更改。例如`glium::DrawParameters`结构体包含24个字段，由于实现了Default，因此值需要提及想要更改的字段即可创建一个结构体：
```rust
let params = glium::DrawParameters {
    line_width : Some(0.02),
    point_size : Some(0.02),
    .. Default::default()
};

target.draw(..., &params).unwrap();
```
**三种自动实现Default**  
1. 如果类型T实现了Default，那么标准库就会自动为Rc\<T>、Arc\<T>、Box\<T>、Cell\<T>、RefCell\<T>、Cow\<T>、Mutex\<T>和RwLock\<T>实现Default。例如，类型Rc\<T>的默认值就是一个指向类型T默认值的Rc。
2. 如果一个元组的所有元素类型都实现了Default，那么该元组类型也同样会实现Default，这个元组的默认值包含每个元素的默认值。
3. Rust不会为结构体类型隐式实现Default，但如果结构体的所有字段都实现了Default，则可以使用#[derive(default)]为结构体自动实现default。

## AsRef与AsMut
如果一个类型实现了AsRef\<T>，那么就意味着你可以高效地从中借入&T。表示一种类型可以廉价地转换为对类型 T 的引用。
```rust
trait AsRef<T: ?Sized> {
    fn as_ref(&self) -> &T;
}

trait AsMut<T: ?Sized> {
    fn as_mut(&mut self) -> &mut T;
}
```
例如，Vec\<T>实现了AsRef<[T]>，String实现了AsRef\<str>和AsRef\<[u8]>。说明Vec\<T>可以零成本转换成&[T]。实现了AsRef特征的函数参数能够接受多种类型，如T:AsRef[T]，说明参数可以数组切片&[T]，也可以是向量Vec\<T>。  
还有对于任意类型T和U，只要满足T:AsRef\<U>，就必然满足&T:AsRef\<U>，所以字符串字面量&str才能转换成&Path，因为str实现了AsRef\<Path>。  

AsMut\<T>只有修改给定的T肯定不会违反此类型的不变性约束时，实现AsMut\<T>的类型才有意义。如Vec\<T> 实现 AsMut<[T]>，修改切片元素不会破坏 Vec 的结构（容量、长度等保持不变）。完全安全，因为切片是 Vec 数据的直接视图。
## Borrow与BorrowMut
`std::borrow::Borrow` 特征类似AsRef：如果一个类型实现了Borrow\<T>，那么它的borrow方法就能高效地从**自身**借入一个&T。  
但是Borrow施加了更多限制：只有当&T能够通过与它借来的值相同的方式进行哈希和比较时，此类型才应实现Borrow\<T>。这使得Borrow在处理哈希表和树中的键或者处理因为某些原因要进行哈希或比较的值时非常有用。  

如String实现了AsRef\<str>、AsRef\<u8>和AsRef\<Path>，但这3种目标类型通常具有不一样的哈希值。只有&str切片才能保证像其等效的String一样进行哈希，因此String只实现了Borrow\<str>。  

Borrow旨在解决具有泛型哈希表和其他关联集合类型的特定情况。

## From与Into
`std::convert::From`和`std::convert::Into`特征表示类型转换，这种转换会接受一种类型的值并返回另一种类型的值，也就是会获取其参数的`所有权`。
```rust
trait Into<T>: Sized {
    fn into(self) -> T;
}

trait From<T>: Sized {
    fn from(other :T) -> Self;
}
```
标准库自动实现了从每种类型到自身的简单转换：每种类型T都实现了From\<T>和Into\<T>。看上去很怪。   
Into\<T>作为参数不仅可以接受参数T，还能接受所有能够转换成T的参数也就是实现了Into\<T>特征。  
From的from方法充当泛型构造函数，用于从另一个值生成本类型的实例。给定适当的From实现，标准库会自动实现相应的Into特征。例如定义自己的类型时，遇到单参数构造函数，可以写成适当类型的From\<T>的实现，这样就会自动获得相应的Into实现。  
因为会获取参数的所有权，所以此转换可以复用原始值的资源来构造出转换后的值。如String的Into\<Vec\<u8>>的实现只是获取String的堆缓冲区，并不进行任何更改的情况下将其重新用作所返回向量的元素缓冲区。此转换既不需要分配内存，也不需要复制文本，是一个通过移动进行高性能实现的例子。

## TryFrom与TryInto
由于转换的行为方式不够清晰，因此Rust没有为i32实现From\<i64>，也没有实现任何其他可能丢失信息的数值类型之间的转换，而是为i32实现了TryFrom\<i64>。
```rust
pub trait TryFrom<T>: Sized {
    type Error;
    fn try_from(value: T) -> Result<Self, Self::Error>;
}

pub trait TryInto<T>: Sized {
    type Error;
    fn try_into(self) -> Result<T, Self::Error>;
}
```
返回一个Result，因此我们可以选择在异常情况下该怎么做，为自己的类型实现容错的转换也很容易。

## ToOwned
`std::borrow::ToOwned`特征提供了一种稍微宽松的方式来将引用转换为拥有型的值
```rust
trait ToOwned {
    type Owned: Borrow<Self>;
    fn to_owned(&self) -> Self::Owned;
}
```
to_owned可以返回任何能让你从中借入&self的类型：Owned类型必须实现Borrow\<Self>。所以Vec\<T>转换成&[T]，只要T再实现了Clone，[T]就能实现ToOwned\<Owned=Vec\<T>>，这样就能将切片元素复制到向量中了。

## Cow
`std::borrow::Cow`类型用于写入时克隆。Cow\<B>要么借入对B的不可变引用，要么拥有可供借入此类引用的值。如果它是Owned，就会借入对拥有值的不可变引用，如果它是Borrowed，就会转让自己持有的引用。
```rust
pub enum Cow<'a, B> 
where
    B: 'a + ToOwned + ?Sized,
{
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}
```
Cow\<T>：根据需要在借用和所有权之间灵活切换