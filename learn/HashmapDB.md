# HashmapDB

[TOC]

仿照其他 DB 或者 map 结构的评测，来评估 hashmapDB。罗列一些指标信息



## Chunk_file

**constructor 说明：**

  Chunk_file( const std::string& prefix, int arr_size, int block_num, int chunk_idx)

arr_size: 每个 block 中最多存储的数据条目

block_num：每个 chunkfile 中最多容纳的 block 个数

chunk_idx: 同一个库可能由多个 chunkfile 组成，标示 chunkfile 的索引



**通过链表的模式维护数据节点**



### 关键数据结构

```c++
    // nested structs
    struct FileHead
    {
        int empty_;     // 存储此文件中当前可用的 block索引信息
        char block_[0];   
    };
    struct BlockHead
    {
        int prev_;
        int next_;
        int tail_;
        size_t length_;
        int cur_num_;
        char content_[0];  // 定义长度为0的数组，创建结构体变量时不分配空间，但是可以通过content变量寻址到 cur_num_之后的地址，事先分配好内存空间，直接通过 memcpy 即可拷贝内容到 content_的位置
    };

/**
 * TermKey_t 存储key签名 
 */
struct TermKey_t {
    unsigned int sign1;     	/**<  term sign1      */
    unsigned int sign2;     	/**<  term sign2      */
};
// 创建 TermKey_t 的算法
// 依赖ul_sign.h
creat_sign_f64(const_cast<char*>(ak),
                strlen(ak), &(common_key.sign1), &(common_key.sign2));
// 依赖 cloud_index_def.h 
template<>
inline TermKey_t to_termkey_t<uint64_t>(const uint64_t &key) 
{
    TermKey_t term;
    term.sign1 = 0;
    term.sign2 = 0;
   
    uint64_t * value = (uint64_t *)&term;
    *value = key;   //  ？？？？？？
    return term;
}

//mmap start pos getter ？？？作用是什么
    FileHead * GetFileHead()
    {
        return (FileHead *) mmap_ptr_;
    }

```

block_size_ = sizeof(BlockHead) + arr_size * sizeof(Value_Type);

block 中的 tail 是如何维护的？？？？未看到有 set

chunk_file 中chunk_idx_是如何维护的？？？？

​      hashmapdb 创建 chunk_file 时，通过构造函数传递。由 hashmapdb 维护



### 关键函数

```c++
// 添加1个或多个元素到 block
int Chunk_file<Key_Type,Value_Type>::AppendData(int block_id, const Value_Type * arr, size_t arr_len){
  	// 检验当前 block_id可容纳的元素数量
  	// 填充当天的 block_id
    // 如果 arr_len 大于 block 能容纳的元素数量，创建新的 block
    // memcpy函数进行数据拷贝
}
// 
BlockHead* _getLocalBlock( int local_block_id) {
        size_t offset = (size_t)local_block_id ;
        offset *= block_size_;

        void * ret = GetFileHead()->block_ + offset;    // 内存怎么分配的
        return (BlockHead*)ret;
}
// 获取指定 item 的数据
Value_Type* Chunk_file<Key_Type,Value_Type>::FetchData( int blk, int item, int* shortcut_len_buf){
  BlockHead* pb = GetBlock(blk);
  return (Value_Type*) (pb->content_ + sizeof(Value_Type) * item);
}
// 新建 chunkfile 本地文件，并通过 mmap 方式映射到内存，添加数据时不再涉及到 new 分配内存，直接通过指针偏移即可
int Chunk_file<Key_Type,Value_Type>::open( open_type_t type) {
 	int fd = ::open( filename_.c_str(), O_RDWR | O_CREAT | O_TRUNC , S_IRUSR | S_IWUSR);
  size_t chunk_size = sizeof(FileHead) + (size_t)block_num_ * (size_t)block_size_;
  if ( 0 != ftruncate( fd, chunk_size) ) {
            //UB_LOG_WARNING("ftruncate failed errno[%d] chunk_size[%zu]", errno, chunk_size);
            return -1;
  }
  void * p = mmap( NULL, chunk_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
}

int Chunk_file<Key_Type,Value_Type>::GetNewBlock(){
  int res = (*p_empty_)++;   // 标示当前chunk可用 block 的索引
  return _getGlobalBlockId(res);
}
int _getGlobalBlockId(int local_block_id) {
        return local_block_id + ( block_num_ * chunk_idx_);
    }
```

chunk_file 的内存是如何分配的???



## HashmapDB

### 关键数据结构

```c++
    struct MetaHead
    {   
        // size of key and value , in bytes.
        size_t key_size_; 
        size_t value_size_;
        // number of data in a block
        int arr_size_;
        // number of block in a chunkfile
        int block_num_;

        char content_[0];
    };  
    struct MetaItem
    {   
        Key_Type key_;
        int head_block_;
    };

// hashmapdb 中含有多个 chunkfile
std::vector<chunk_t_*> chunks_;
// ????
std::map<Key_Type,int> kb_table_;

```

blockID是如何生成的？？

### 关键函数

```c++
// 为 block 分配内存
int HashMapDB<Key_Type,Value_Type>::_getNewBlock() {
  chunk_t_* pc = new (std::nothrow) chunk_t_(data_dir_ + map_name_, arr_size_, block_num_, chunk_cnt) ;
  // 创建 t
  pc->open( FORCE_CREAT);
  chunks_.push_back( pc );
  return chunks_[chunk_cnt]->GetNewBlock();
}

// 添加元素
int HashMapDB<Key_Type,Value_Type>::add( const Key_Type * key, const Value_Type *arr, size_t arr_len){
  
}

```









