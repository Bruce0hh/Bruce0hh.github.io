# 常用关键字

## for 和 range

### 现象

1. “循环永动机”

   ```go
   func main() {
       arr := []int{1,2,3}
       for _, v := range arr {
           arr = append(arr, v)
       }
       fmt.Println(arr)
   }
   // 结果
   1 2 3 1 2 3
   ```

   - 上述代码并不会无限输出 `1 2 3`

2. 神奇的指针

   ```go
   func main() {
       arr := []int{1,2,3}
       newArr := []*int{}
       for _, v := range arr {
           newArr = append(newArr, &v)
       }
       for _, v := range newArr {
           fmt.Println(*v)
       }
   }
   // 结果
   3 3 3
   ```

   - 上述代码的正确做法是使用`&arr[i]` 替代 `&v`

3. 遍历清空数组

   ```go
   func main() {
       arr := []int{1,2,3}
       for i,_ := range arr {
           arr[i] = 0
       }
   }
   ```

4. 随机遍历

   ```go
   func main() {
       hash := map[string]int {
           "1" : 1,
           "2" : 2,
           "3" : 3,
       }
       for k, v := range hash {
           println(k, v)
       }
   }
   ```

   - 上述遍历顺序是不确定的

### 经典循环

组成：

1. 初始化循环的 `Ninit`；
2. 循环的继续条件`Left`；
3. 循环体结束时执行的`Right`；
4. 循环体`NBody`。

### 范围循环（使用 for 和 range）

1. 数组和切片
   - **清空数组**：相比于一次清除数组或者切片中的数据，Go 语言会直接使用 `runtie.memclrNoHeapPointers` 或者`runtime.memclrHasPointers`清除目标数组内存空间中的全部数据，并在执行完成后更新遍历数组的索引。
   - **`for range a {}` **：代码不需要获取数组的索引和元素，只需要使用数组或者切片的数量执行对应的循环，所以会生成一个最简单的 for 循环。
   - **`for i := range a{}`**：会传递遍历数组使用的索引。
   - **`for i，v := range a{}`**：当代码同时关心索引和切片的情况，它不仅会在循环体中插入更新索引的语句，还会插入赋值操作让循环体内部的代码能够访问数组中的元素。
   - 对于**所有 range 循环**，Golang 都会在编译期将原切片或者数组赋值给一个新变量，在复制过程中就发生了复制，而我们又通过 len 关键字预先获取了切片的长度，所以在循环中追加新元素不会改变循环执行的次数。
   - 当遇到同时遍历索引和元素的 range 循环时，Golang 会额外创建一个新的变量存储切片中的元素，循环中使用的这个变量会在每一次迭代被重新赋值而覆盖，赋值时也会触发复制。
   - 因为在循环中获取返回变量的地址都完全相同，所以会发生神奇的指针现象。所以当我们想访问数组中元素所在的地址时，不应该直接获取 range 返回的变量地址 &v ，而应该使用 &a[index] 这种形式。

2. 哈希表

   **桶的选择**：

   - 当待遍历的桶为空时，选择需要遍历的新桶；
   - 当不存在待遍历的桶时，返回`(nil, nil)`键值对并中止遍历。

   **桶内元素的遍历**：

   - 从桶中找到下一个遍历的元素，在大多数的情况下会直接操作内存获取目标键值的内存地址。

   **遍历顺序**：首先选出一个绿色的正常桶开始遍历，随后遍历所有黄色的溢出桶，最后按照索引顺序遍历哈希表中其他的桶，直到遍历完所有桶。

3. 字符串

   使用下标访问字符串中的元素时得到的就是字节，在遍历时会获取字符串。其余遍历过程与上述几个很相像。

4. Channel

   `for v := range ch {}`

   ```go
   ha := a
   hv1, hb := <-ha
   for ; hb != false; hb1, hb = <-ha {
       v1 := hv1
       hv1 = nil
       ...
   }
   ```

   

   该循环会使用`<-ch`从 Channel 中取出待处理的值，这个操作会调用`runtime.chanrecv2` 并阻塞当前协程，当`runtime.chanrecv2`返回时会根据布尔值 hb 判断当前值是否存在：

   - 如果当前值不存在，意味着当前 Channel 已经被关闭；
   - 如果当前值存在，会为 v1 赋值并清除 hv1 变量中的数据，然后重新陷入阻塞等待新数据。

## select

### 现象

- **select 能在 Channel 上进行非阻塞的收发操作；**

  通常情况下，select 语句会阻塞当前 Goroutine，并等待多个 Channel 中的一个达到可以收发的状态；但是如果 select 控制结构中包含 default 语句，那么该 select 语句在执行时会遇到以下两种情况：

  1. 当存在可以收发的 Channel 时，直接处理该 Channel 对应的 case；
  2. 当不存在可以收发的 Channel 时，执行 default 中的语句。

- **select 在遇到多个 Channel 同时响应时，会随机执行一种情况。**

  select 在遇到多个 `<-ch`同时满足可读或可写条件时会随机选择一个 case 执行其中的代码。如果两个 case 同时满足执行条件，我们按照顺序依次执行判断的话，那么后面的条件永远都得不到执行，引入随机性就是为了避免饥饿问题的发生。

### 小结

编译期间，Golang 会对 select 语句进行优化，它会根据 select 中 case 的不同选择不同的优化路径。

- 空的 select 语句会被转换成调用`runtime.block`直接挂起当前 Goroutine。
- 如果 select 语句中只包含一个 case，编译器会将其转换成 `if ch == nil {block};`表达式；
  - 首先判断操作的 Channel 是否为空；
  - 然后执行 case 结构中的内容。
- 如果 select 语句中只包含两个 case 并且其中一个是 default，那么会使用`runtime.selectnbrecv`和`runtime.selectnbsend`非阻塞地执行收发操作。
- 默认情况下会通过`runtime.selectgo`获取执行 case  的索引，并通过多条 if 语句执行对应 case 中的代码

编译优化之后，Golang 会在运行时执行编译期间展开的 runtime，selectgo 函数，该函数会按照以下流程执行：

1. 随机生成一个遍历的轮询顺序 pollOrder 并根据 Channel 地址生成加锁顺序 lockOrder。
2. 根据 pollOrder 遍历所有 case，查看是否有可以立刻处理的 Channel：
   - 如果存在，直接获取 case 对应的索引并返回；
   - 如果不存在，创建`runtime.sudog`结构体，将当前 Goroutine 加入所有相关 Channel 的收发队列，并调用`runtime.gopark`挂起当前 Goroutine 等待调度器唤醒。
3. 当调度器唤醒当前 Goroutine 时，会再次按照 lockOrder 遍历所有 case，从中查找需要被处理的`runtime.sudog`对应的索引。

## defer

**defer 机制——开放编码：**

- 编译期间判断 defer 关键字、return 语句的个数确定是否开启开放编码优化；
- 通过 deferBits 和 `gc.openDeferInfo`存储 defer 关键字相关信息；
- 如果 defer 关键字的执行可以在编译期间确定，会在函数返回前直接插入响应代码，否则会由运行时的`runtime.deferreturn`处理。

**两个现象：**

- 后调用的 defer 函数会先执行：
  - 后调用的 defer 会被追加到 Goroutine `_defer`链表的最前面；
  - 运行`runtime.-defer`时是从前到后依次执行的。
- 会预先计算函数的参数：
  - 如果调用`runtime.deferproc`函数创建新的延迟调用，就会立刻复制函数的参数，函数的参数不会等到真正执行时计算。

## panic 和 recover

- panic 能够改变程序中的控制流，调用 panic 后会立刻停止执行当前函数的剩余代码，并在当前 Goroutine 中递归执行调用方的 defer。
- recover 可以中止 panic 造成的程序崩溃。该函数只能在 defer 中发挥作用，在其他作用域中调用不会发挥作用。

### 现象

- panic 只会触发**当前 Goroutine 的 defer** （跨协程失效）；
- recover 只有在 defer 中调用才会生效 （失效的崩溃恢复）；
  - recover 只有在发生 panic 之后调用才会生效。但如果 recover 是在 panic 之前调用的，自然就不满足生效条件，所以需要在 defer 中使用 recover 关键字。
- panic 允许在 defer 中嵌套多次调用（嵌套崩溃）。

## make 和 new

- make 的作用是初始化内置的数据结构，也就是切片、哈希表、Channel。
- new 的作用是根据传入的类型分配一块内存空间，并返回指向这块内存空间的指针。

