
db.rs中介绍了Cache trait，这里讲的是cache_manager，看的时候需要将这2个文件结合起来看。

1. 
    CacheManager 是个结构体，里面有3个配置：pref_cache_size、max_cache_size和bytes_per_cache_entry ，和1个cache ：HashSet的VecDeque。  
    也就是里面可以放置很多个cache，每一个cache是1个hashset，然后多个cache以VecDeque的方式组织。  
    有一个全部变量：COLLECTION_QUEUE_SIZE用来定义这个VecDeque的大小。  

2. 
    CacheManager 定义了4个方法：new、note_used、collect_garbage、retate_cache_if_needed。  

    new方法就是根据入参中的3个配置进行设置，并新建 COLLECTION_QUEUE_SIZE 个HashSet构成1个VecDeque。  
    
    note_used：如果在第0个HashSet中，直接返回；如果不在，在后面的几个HashSet中查找，找到的话就删除掉，然后插入到第0个HashSet中  
                这样的话，就是使用第0个HashSet来保存used的数据。
                
    
    collect_garbage：清理那些不用的数据。  
            首先判断current_size是否小于 pref_cache_size，如果是，就调用 rotate_cache_if_needed。 ？  
            VecDeque总共有 COLLECTION_QUEUE_SIZE 个HashSet，依次弹出最后1个，使用提供的方法进行处理，并在VecDeque的开头放置1个新的HashSet。  
            如果current_size小于max_cache_size，就返回了。  
            
    rotate_cache_if_needed：如果条件满足就进行滚动。  
            如果VecDeque是空的，直接返回；  
            如果第0个HashSet的长度乘以 bytes_per_cache_entry 大于 pref_cache_size除以COLLECTION_QUEUE_SIZE 的值，就进行滚动。  
            滚动方法就是弹出最后一个HashSet，放入队列最前面。  
    
3. 通过上面的分析，可以看出 3条配置控制着VecDeque的运作。 max_cache_size是整个cache的最大值，pref_cache_size是整个cache需要滚动的值，
    bytes_per_cache_entry 是每个HashSet的字节值。  
    我们的代码中pref_cache_size是16K，max_cache_size是1M，bytes_per_cache_entry是400B。  
