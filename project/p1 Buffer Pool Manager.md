### Buffer Pool Manager 缓冲池管理器

在该项目中，为`BusTub DBMS`构建一个新的面向磁盘的存储管理器。这样的存储管理器假设数据库的主要存储位置在磁盘上。

该项目是在存储管理器中实现缓冲池。缓冲池负责在主内存和磁盘之间来回移动物理页，它允许DBMS支持大于系统可用内存量的数据库。缓冲池的操作对系统中的其它部分是透明的。系统通过页面的唯一标识符(`page_id`)向缓冲池请求页面，缓冲池从内存或磁盘中检索该页面后返回给系统。

需要在存储管理器中实现三个组件：

* Extendible Hash Table 可扩展哈希表
* LRU-K Replacement Policy LRU-K置换算法
* Buffer Pool Manager Instance 缓冲池管理器实例

#### Task1 Extendible Hash Table

##### 1.1 可扩展哈希

[Extendible Hashing](https://www.geeksforgeeks.org/extendible-hashing-dynamic-approach-to-dbms/)
[2022 p1 Buffer Pool](https://blog.csdn.net/q2453303961/article/details/128153709?spm=1001.2014.3001.5502)

静态哈希（静态散列）要求桶的数目始终固定，那么在确定桶的数目和选择哈希函数时，如果桶的数目过小，随着数据量的增加，性能会降低；如果留有一定余量，又会带来空间的浪费；或者定期重组哈希索引结构，但开销大且耗时。为此，提出了几种动态哈希(dynamic hashing)技术，可扩展哈希(extendible hashing)便是其一。

可扩展哈希中的一些术语：

* **目录(Directories)**：目录存储指向桶的指针，每个目录都有一个唯一的`ID`，每次扩展时该ID都可能会发生变化。哈希函数返回目录ID，用于定位到合适的桶。目录数=2^全局深度。
* **桶(Buckets)**：桶用来存放数据。目录指向桶。如果桶的局部深度小于全局深度，则可能包含多个指向该桶的指针。注意区分桶的大小和局部深度的区别。
* **全局深度(Global Depth)**：对`key`哈希后得到二进制的数据，选取二进制数据的`D`个最低有效位作为作为目录ID。`D`就是全局深度，即全局深度=目录ID的位数。
* **局部深度(Loacl Depth)**：每个桶都有独立的局部深度（记作`L`），局部深度表示该桶中所有元素的`key`的哈希值的后`L`位是相等的。局部深度根据全局深度来决定发生溢出时要执行的操作。局部深度始终小于等于全局深度。
* **目录扩展(Directory Expansion)**：目录扩展发生在桶溢出时。当溢出桶的局部深度等于全局深度时，进行目录扩展。目录扩展将使哈希结构中的目录数加倍。
* **桶分裂(Bucket Splitting)**：当桶中的元素数量超过设定大小时，桶被分为两部分。

##### 1.2 任务实现

构建一个可扩展哈希表，该表使用无序存储桶来存储唯一的键值对。实现的哈希表必须支持插入和删除键/值的功能，而无需指定表的最大大小。实现的表需要根据需要逐渐增大，但不需要缩小它。

1. Insert()

* s1 获取插入元素的`hash()`，取得到的二进制数的`目录的全局深度`个最低有效位，并将其与目录`dir_`进行匹配。得到匹配的目录所指向的桶的指针，即`dir_[i]`。
* s2 如果`dir_[i]`指向的桶已经存在待插入元素，则用新值覆盖旧值，并转到s5。
* s3 尝试插入元素，如果插入成功则转到s5。否则桶已满，进行桶溢出处理：
* * 桶溢出处理：判断当前桶的局部深度`local_depth_`和全局深度`golbal_depth_`的大小关系：
* * * c1：如果`local_depth_`等于`global_depth_`，则先进行目录扩展，再进行桶分裂。
* * * c2：如果`local_depth_`小于`global_depth_`，则只进行桶分裂。
* s4 桶溢出处理完成，转到s3 再次尝试插入元素。
* s5 插入元素完成。

**目录扩展**：将当前目录`dir_`的**目录项扩容一倍**，然后将**全局深度`global_depth_`的值加1**。假设扩容前`global_depth_=2`，则`size(dir_)=4`，扩容后`global_depth_=3`，`size(dir_)=8`。扩容后的新增的目录项`dir_[x] (x=4,5,6,7)`和目录项`dir_[x-4] (x-4=0,1,2,3)`指向同一个桶。因为(假设`x=5`)：`dir_[5]`对应的`key`的哈希值的低3位为`101`，低2位为`01`，`dir_[1]`对应的`key`的哈希值的低3位为`001`，低2位为`01`，也就是说，扩容后本该存储在`dir_[5]`中的元素在扩容前存储在`dir_[1]`中，所以这两个目录指向同一个桶。

任何时候任一桶的局部深度`local_depth_`都小于等于全局深度`global_depth_`。任一局部深度小于全局深度的桶都会被多个目录所指向，桶中存储的元素的`local_depth_`个最低有效位都相同；这多个目录的`local_depth_`个最低有效位都相同，但`global_depth_`个最低有效位都不相同。

**桶分裂**：**将要分裂的桶的局部深度`local_depth_`的值加1**，然后**新建一个新桶**，原先的桶称为旧桶。以`global_depth_=3`，`x=1`，分裂前`local_depth_=2`，分裂后`local_depth_=3`为例：之前`dir_[1]`指向的桶已满，新建一个新桶，**根据新的局部深度，判断哪些目录指针应该指向新桶并修改指向**（如`dir_[5]`之前指向的桶与`dir_[1]`相同，但现在应该指向新桶）。旧桶中的元素的`2`个最低有效位都相同，但`3`个最低有效位不一定相同，**桶分裂后对旧桶中的元素根据新的局部深度重新进行判断，分配到新桶(`101`)和旧桶(`001`)中**。

2. **确保哈希表中的Insert()、Remove()、Find()都是线程安全的。**

#### Task2 LRU-K Replacement Policy

##### 2.1 LRU-K

`缓存`是一种常见的空间换时间的做法，但缓存的大小不是无限制的，需要一种算法淘汰超出大小限制的缓存。LRU（Least Recently Used，最近最少使用）算法根据数据的历史访问记录来淘汰数据，其核心思想是“最近被访问过的数据，其在将来被访问的几率更大”。其实现原理是：维护一个`历史记录队列`（链表）保存每次访问的数据。新的访问数据插入到队列的头部（最近使用）；每当缓存命中（即缓存数据被访问）时将数据移动到队列的头部；当队列满时将淘汰队尾（最久使用）的数据。

LRU-K算法的主要目的是为了解决LRU算法可能会出现的`缓存污染`（和`预读失效`）问题，因此将“最近访问过一次”的判断标准扩展为“最近访问过K次”。只有访问次数达到`K`次的数据才会被缓存，因此需要对缓存数据的访问次数进行计数，并且访问记录不能无限记录，也需要使用替换算法进行替换，当需要淘汰数据时，LRU-K会淘汰第K次访问时间与当前时间间隔最大的数据。相比LRU，LRU-K需要多维护一个`缓存队列`，当数据的访问次数达到K次的时候，才将数据索引从历史记录队列移动到缓存队列。缓存队列中的数据被访问后进行重新排序。

后向k距离的计算方式是当前时间戳与第k次访问时的时间戳之间的时间差。历史访问次数少于k次的帧被赋予`+inf`(正无穷大)作为其后向k距离。LRU-K算法将替换器的所有帧中后向k距离最大的那个帧驱逐。当多个帧具有+inf后向k距离时，替换器将驱逐具有最早时间戳的那个帧。

实际实现上：历史记录队列采用FIFO，驱逐时选择队尾的元素（最久访问），新元素的访问记录插入到队头，再次访问队列中的元素时不移动其位置。缓存队列采用LRU，再次访问队列中的元素时将其移动到队头，从历史记录队列移动到缓存队列的元素也插入到队头，驱逐时选择队尾的元素。

`LRUKReplacer`的最大`size`与缓冲池的`size`相同，因为它包含`BufferPoolManager`中所有页面的帧占位符。但在任何给定时刻，替换器中并非所有的帧都是可驱逐的。`LRUKReplacer`的`size`由可驱逐的帧的数量表示，`LRUKReplacer`被初始化为其中没有帧，只有当一个帧被标记为可驱逐时，替换器的`size`才会增加。

##### 数据成员

每个元素有一个是否可驱逐标记`evictable`，如果为`false`则无论在哪个队列中都不能被驱逐。

类`LRUKReplacer`(~/src/buffer/lru_k_replacer.h)数据成员：

`size_t curr_size_{0}`：替换器中可驱逐的帧的数量。不可驱逐的帧不计入其中。
`size_t replacer_size_`：限定`frame_id`的取值范围（大小不能超过该值），用于判断入参是否合法，并不是表示cache的容量。

```C++
// additional members
struct FrameInfo{               // 帧的信息
  bool is_evictable_{false};    // 是否可驱逐
  size_t access_count_{0};      // 被访问次数
  std::list<frame_id_t> pos_;   // 帧在队列中的位置（无论在历史记录队列还是
    // 缓存队列中，因为二者的类型相同，且list的迭代器不会因为插入删除操作失效）
}
```

`std::list<frame_id_t> history_list_`：历史记录队列
`std::list<frame_id_t> cache_list_`：缓存队列
`std::unordered_map<frame_id_t,FrameInfo> frame_info_map_`：关联二者信息，为了在`O(1)`的时间内完成帧信息的查找。

##### 2.2 任务实现

1. `auto Evict(frame_id_t *frame_id) -> bool`：与其它所有被`Replacer`跟踪的可驱逐的帧相比，驱逐具有最大后向k距离的那个，将帧ID存储在输出参数`frame_id`中并返回true，如果没有可驱逐的帧则返回false。

* 先从历史队列中尝试驱逐满足条件的帧，否则从缓存队列中尝试驱逐满足条件的帧。
* 根据驱逐的结果更新替换器中可驱逐帧的数量。
* 注意：无论在哪个队列，被标记为不可驱逐的帧都不能被驱逐。

2. `void RecordAccess(frame_id_t frame_id)`：记录给定ID的帧在当前时间戳的访问。当页面固定在(is pinned in)`BufferPoolManager`后调用此方法。

* 更新该帧的访问次数（值加1），根据访问次数决定之后的操作。
* * 等于`1`次：是一个新帧，放到历史记录队列的头部（最新访问位置）。
* * 大于`1`次小于`k`次：不会改变位置，即不进行任何操作。
* * 等于`k`次：将该帧从历史记录队列移出，放入缓存队列的最新访问位置（队列头部或尾部，取决于具体实现：哪边是最久访问，哪边是最新访问）。
* * 大于`k`次：已经在缓存队列中，移动其位置，放在最新访问的位置。

3. `void Remove(frame_id_t frame_id)`：清除和帧关联的所有访问记录。仅当在`BufferPoolManager`中删除页面时才调用此方法。

* 判断给定的frame_id是否合法或存在。
* 判断该帧是否是可驱逐的，为不可驱逐的则不能删除。否则从历史记录队列或缓存队列中删除该帧的信息。
* 替换器中更新可驱逐帧的数量（`curr_size_`变量的值）。

4. `void SetEvictable(frame_id_t frame_id, bool set_evictable)`：此方法决定一个帧是否可驱逐。

* 判断给定的frame_id是否合法或存在。
* 根据该帧当前的是否可驱逐标志的值和给定的`set_evictable`的值，更新替换器中当前可驱逐帧的数量。
* 根据上一步的结果更新该帧的是否可驱逐标志的值。

5. `Size()`：返回当前位于`LRUKReplacer`中可驱逐的帧的数量。

#### Task3

##### 3.1 Buffer Pool Manager Instance

1. page && frame

DBMS启动时会从OS申请一片内存区域，即`Buffer Pool`，并将这块区域划分为大小相同的内存页面，用于存储`disk page`的具体内容，每个内存页面(`in-memory page`)由一个`Page`对象表示。为了与`disk page`区分，通常称之为`frame`。`frame`不等同于`Page`，而是指示一个`disk page`对应BP中的一个`in-memory page`，即位置。当DBMS请求一个`disk page`时，它首先需要被复制到`Buffer Pool`的一个`frame`（指示的某个内存页面）中。

简单来说BP是充当数据库上层设施和磁盘文件之间的缓冲区，`Page`是承载4K大小数据的类(**`class`**)，可以通过`DiskManager`从磁盘文件读写，用`page_id`标识。`Frame`不是具体的类(**`int32_t`**)，BP中有容纳`page`的槽位，BPM中有一个`Page`数组，`frame_id`就是某个`disk page`在该`Page`数组中的下标。

`page table`：维护disk中的`page`和buffer pool中的`frame`的映射关系。
`page directory`：维护`page id`和`page`在磁盘上位置的映射关系。该映射关系必须记录到磁盘，这样DBMS重启时不会丢失映射关系。

1. Buffer Pool Manager Instance

实现缓存池管理器(`BufferPoolManagerInstance`)。`BPMI`负责从`DiskManager`（磁盘管理器）获取数据库的页面，并把它们存储在内存中。当需要为新页面腾出空间而驱逐页面时或者明确指出需要这样做时，`BPMI`负责将脏页(dirty page)写入到磁盘里。（磁盘管理器类`(src/include/storage/disk/disk_manager.h)`负责从磁盘来回的读取和写入页面数据。当需要将页面提取到缓冲池和将页面刷新到磁盘时，缓冲池管理器使用`DiskManager::ReadPage()`和`DiskManager::WritePage()`来实现。）

系统中所有的`in-memory page`均由`Page`对象表示。`BPMI`不需要了解这些内存页面的具体内容。但是注意：`Page`对象只是缓冲池中存储`disk page`的容器，因此它并不特定于唯一的页面。也就是说：`Page`对象包含一个内存块，`DiskManager`将使用该内存块来复制从磁盘里读取的物理页（`disk page`）的内容。`BPMI`将重用相同的`Page`对象来存储来回移动到磁盘的数据。这意味着在系统的整个生命周期中，相同的`Page`对象可能包含不同的物理页。`Page`对象的标识符`page_id`用于跟踪它包含的物理页，如果`Page`对象不包含物理页，则其`page_id`必须设置为`INVALID_PAGE_ID`。

外界只知道页面的`page_id`，向BPM查询时，BPM首先要确定该`disk page`是否存在（是否在BP的某个内存页面槽位中）以及其位置（frame_id（在的话）），所以要维护一个从`page_id`到`frame_id`的映射。为了区分空闲的和占用的`Page`（`disk page`的内容被移动到BP的内存页面上存放，这部分内存页面是占用的，其它的内存页面是空闲的），需要维护一个`free_list_`，保存目前空闲的内存页面的`frame_id`，初始时所有的内存页面都是空闲的。

每个`Page`对象还维护一个计数器(`pin_count`)，用于记录固定("pin")/引用该页面的线程数。`BPMI`不允许释放被固定（被引用）的`Page`，因为它正在被使用，即使空间不够时也不能被移除。每个`Page`对象还跟踪它是否脏(`dirty`)。需要记录页面在被取消固定之前是否被修改（被修改过的页就是脏页）。`BPMI`必须将脏页(`Page`)的内容写回磁盘（因为已经被修改了），然后才能重用该对象。

`BufferPoolManagerInstance`将使用`ExtendibleHashTable`作为将`page_id`映射到`frame_id`的表。它还使用`LRUKReplacer`来跟踪`Page`对象何时被访问，以便为了从磁盘上复制一个新的物理页，而必须释放一个帧来腾出空间时，来决定驱逐哪个对象。

##### 3.2 类的数据成员

`~/src/include/buffer/buffer_pool_manager_instance.h`

* `Page* pages_`：缓冲池的页面数组。
* `ExtendibleHashTable<page_id_t, frame_id_t> *page_table_`：维护`page_id`到`frame_id`的映射关系。
* `std::list<frame_id_t> free_list_`：保存空闲的内存页面的`frame_id`。初始时，每个内存页面都在该空闲列表中。

`~src/include/storage/page/page.h`

* `page_id_t page_it_=INVALID_PAGE_ID;`：页面ID。不包含物理页时值为`INVALID_PAGE_ID`。
* `char data_[BUSTUB_PAGE_SIZE]{};`：存储在页面中的实际数据。
* `int pin_count_=0;`：固定/引用页面的计数。
* `bool is_dirty_=false;`：脏页为true。即与磁盘上对应的页不同。

##### 3.2 任务实现

1. `auto NewPgImp(page_id_t *page_id) -> Page*`：在缓冲池中创建一个新页面，将`page_id`设置为新页面的ID。如果无法创建新页面则返回`nullptr`，否则返回指向新页面的指针。

* 首先从空闲列表中寻找是否有可用的`frame`，如果没有则在替换器中尝试驱逐，从而得到一个可用的替换`frame`。二者都没有则返回`nullptr`。
* 得到空闲帧或替换帧之后调用`AllocatePage()`获取新页面的ID。
* * 如果得到的是替换帧，且有脏页标记，则要先将其写回磁盘，然后重置这块内存，为新页面腾出空间。
* 调用`disk_manager->ReadPage()`从磁盘将新页面的内容读取进来。
* 同时更新相关信息：更改对应的`Page`对象的成员信息，记录`page_id`和`frame_id`的映射关系，将该帧固定并设置为不可驱逐，并为该帧记录一次历史访问记录。

2. `auto FetchPgImp(page_id_t page_id) -> Page *`：从缓冲池中请求获取页面ID为`page_id`的页面。如果

* 如果缓冲池中有该页面，返回指向请求页面的指针。
* 如果没有该页面，则从缓冲池中找一个空闲帧/替换帧：
* * 有空闲帧/替换帧，则调用`disk_manager->ReadPage()`从磁盘将ID为`page_id`的页面读取进来。
* * 找不到则返回`nullptr`。
* * 如果找到的是替换帧，则需要和`NewPgImp()`中一样进行脏页的处理。
* 同时更新相关信息：更改对应的`Page`对象的成员信息，记录`page_id`和`frame_id`的映射关系，将该帧固定并设置为不可驱逐，并为该帧记录一次历史访问记录。

3. `auto UnpinPgImp(page_id_t page_id, bool is_dirty) -> bool`：将页面ID为`page_id`的页面从缓冲池中取消固定。

* 如果该页面不在缓冲池中，或者在缓冲池中但其引用计数(`pin_count_`)已经为0，则返回false。
* 如果该页面在缓冲池中且引用计数不为0，递减一次引用计数(`pin_count_--`)，如果递减后为0，将该帧设置为可驱逐的。返回true。
* 参数`is_dirty`指示页面在固定时是否被修改，true则更新`is_dirty_`的值为true，false则维持`is_dirty_`的值不变。

4. `auto FlushPgImp(page_id_t page_id) -> bool`：将页面ID为`page_id`的页面刷新到磁盘，不管页面是否为脏页和是否正在被引用。

* 如果`page_id`是`INVALID_PAGE_ID`，或者该页面不在缓冲池中，返回false，否则返回true。

5. `auto DeletePgImp(page_id_t page_id) -> bool`：从缓冲池中删除页面ID为`page_id`的页面。

* 如果该页面不在缓冲池中，什么都不做，返回true。如果该页面存在但被固定且不能删除（即被引用），立即返回false。
* 从缓冲池中删除该页面：重置页面的`Page`信息，删除在哈希表中的映射记录，删除在替换器中的记录，将该`frame`放到`free_list_`中。最后调用`DeallocatePage()`模仿在磁盘上释放页面。删除页面后返回true。
