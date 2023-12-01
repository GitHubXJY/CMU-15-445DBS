### B+ Tree

本项目是在数据库系统中实现**索引**，用于进行快速数据检索，而无需搜索数据库表中的每一行，为快速随机查找和有序记录的高效访问提供基础。

#### Task1 B+ Tree Pages

##### 1.1 three page class

实现三个Page类来存储B+ Tree的数据：B+ Tree Parent Page 父页面、B+ Tree Internal Page 内部页面、B+ Tree Leaf Page 叶子页面。

1. B+ Tree Parent Page

B+ Tree Parent Page是B+ Tree Internal Page和B+ Tree Leaf Page的父类。父页面只包含两个子类共享的信息。父页面分为如下几个字段：

|变量名|大小|描述|
|:-:|:-:|:-:|
|page_type_|4|页面类型（内部或叶子）|
|size_|4|页面目前存储的键值对的数量|
|max_size_|4|页面能存储的键值对的最大数量|
|parent_page_id_|4|父页面ID|
|page_id_|4|页面ID|
|lsn_|4|日志序列号（项目4中使用）|

2. B+ Tree Internal Page 内部页面

内部页面不存储任何实际数据，而是存储`m个=key`和`m+1个value`，其中`key`是有序的，`value`是指向子页的指针，即子页的`page_id`。指针`PAGE_ID(i)`指向的子树中的所有键`k`都满足：`K(i) <= k < K(i+1)`。由于指针的数量不等于键的数量，第一个键被设置为无效，任何查找都应该忽略第一个键。（m+1 个 k&v 对，第一个 key 无效，第 2 到 m+1 共 m 个有效 key）

内部页面的设计是：`| HEADER | KEY(1)+PAGE_ID(1) | KEY(2)+PAGE_ID(2) | ... | KEY(n)+PAGE_ID(n) |`，其中键以递增的顺序存储。（内部页面的理论结构是 **<指针、键、指针、......、键、指针>**。但为了方便存储，会在头部多补一个无效的键，从而可以用一个pair的数组存储。因此实际上是 **<~~键(exist but invalid)~~、指针、键、指针、......、键、指针>**）

类中用于页面数据的数组成员`Mapping array_[1]`是`flexible array`，在为类分配空间时，除了已经使用的空间，`flexible array`会自动填充剩下的空间。因此它只能是类的最后一个数据成员。

3. B+ Tree Leaf Page

叶子页面存储`m个key`和`m个value`，在实际实现中，`value`只能是64位的`RID (record_id)`，用于定位实际元组的存储位置。叶子页面的设计是：`| HEADER | KEY(1) + RID(1) | KEY(2) + RID(2) | ... | KEY(n) + RID(n)`。`RID`通过两个数据成员页面ID`page_id_`和逻辑偏移量`slot_num_`来定位数据实际存放的位置。

Note：尽管叶子页面和内部页面包含相同种类的`key`，但它们可能有不同类型的`value`，因此叶子页面和内部页面的`max_size_`可能不同。

B+树所有的叶子页面和内部页面相当于（或者说对应）从缓冲池中获取的内存页面的内容（即`data`部分）。因此每次尝试读取或写入叶子页面/内部页面时都需要先使用其唯一的`page_id`来从缓冲池中获取页面，然后`reinterpret_cast`强制转换成叶子页面或内部页面，并且在任何写入或读取操作之后`unpin`页面。

**上面一段的意思是：叶子页面和内部页面都不是直接创建出来的，而是由 Buffer Pool 管理的 Page 的`data`部分经过类型转换而来的。B+树的节点使用预先分配好的固定空间，`array_`可以控制从该位置开始到 Page的`data`部分结束为止的这一段空间。因此节点对象的生命周期不由`new`和`delete`控制，而由项目一中实现的BufferPoolManager管理：取页面用`FetchPage()`，使用结束归还页面用`UnpinPage()`。`BPlusTreePage`中的数据成员`page_id_`不仅是B+树中节点的编号，同时也是这个节点使用的Page在BufferPool中的编号。**

##### 1.2 任务实现

#### Task2 B+ Tree Data Structure

##### 2.1 BPlusTree

[B树和B+树的插入与删除](https://www.cnblogs.com/nullzx/p/8729425.html)

`BPlusTree`类代表整个B+树，其数据成员有：`buffer_pool_manager_`，由外部传入；`root_page_id_`，表示根节点ID；`comparator_`，`KeyComparator`类型的对象，用于键的大小比较；`leaf_max_size_`和`internal_max_size_`，表示叶子节点和内部节点的最大容量。

B+树索引的键是唯一的。如果插入触发了拆分条件（对于叶子节点，插入后的键值对数量等于`max_size`，对于内部节点，插入之前的子节点数量等于`max_size`），插入后应该执行拆分。如果删除触发了合并条件（），删除后应该执行合并。

由于任何写操作都可能导致B+树索引中的`root_page_id`被更改，因此需要更新标头页面(`~/src/include/storage/page/header_page.h`)中的`root_page_id`，这是为了确保索引在磁盘上是持久的。在`BPlusTree`类中，已经实现了一个名为`UpdateRootPageId()`的函数，每当B+树索引的`root_page_id`发生变化时调用次函数。

此外，还有已经实现的类及其描述：

* `KeyType`：索引中键的类型。这只是`GenericKey`，它的实际大小是使用模板参数指定和实例化，并且取决于索引属性的数据类型。
* `ValueType`：索引中`value`的类型。它只是64位的`RID`。
* `KeyComparator`：用于比较两个`KeyType`实例大小的类。

##### 2.2 任务实现

#### Task3 Index Iterator

#### Task4 Concuurent Index
