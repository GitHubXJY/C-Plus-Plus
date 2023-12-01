### AddressSanitizer

#### AddressSanitizer: SEGV on unknown address 0x0000

`SEGV`是分段违规(segmentation violation)的缩写，操作系统中的程序只允许访问分配给它的内存段，分段违规意味着程序试图访问未分配给它的内存段，或试图访问已经释放的内存段，从而导致程序崩溃。

在C中，通常是对包含错误地址的取消引用导致的。代码试图访问内存中一个“无意义”的位置，当程序尝试访问其位置存储在尚未设置或意外覆盖的变量的内存时，可能发生这种情况，通常它会尝试访问0地址位置。

可能导致该问题的原因有：使用错误的指针读取/写入内存分配之前或之后的内存地址；使用指针访问已经释放的内存；使用未初始化的指针访问内存；程序进行无效的内存引用，内存引用可能无效，因为引用的页面不存在。

```C++
// 越界访问
vector<int>v;
v.size()==0;
cout<<v[0]; // error!
```

#### AddressSanitizer: heap-use-after-free on address 0xXXXX

```C++
// 访问已经释放的内存地址
for(std::vector<T>::iterator it=vec_.begin();it!=vec_.end();){
    vec_.earse(it);
    it++;
}
```
