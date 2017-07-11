# TBB— concurrent_hash_map

## 源码解析

### 关键数据结构

struct hash_map_node_base;

struct bucket;

//! Segment pointer

typedef bucket *segment_ptr_t;

static size_type const pointers_per_table = sizeof(segment_index_t) * 8; // one segment per bit

//! Segment pointers table type

typedef segment_ptr_t segments_table_t[pointers_per_table];

class hash_map_base;

