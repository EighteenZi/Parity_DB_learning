
db.rs里面是DB相关的一些内容：列有哪些、对Batch、KVDB的扩展。

1. 首先是DB的列的名称定义，总共7个列。

2. 剩余的全部内容都是对Batch的扩展，首先的扩展是Batch进行Cache，然后是Batch实现Writable，KVDB实现Readable。
    2.1
    Cache是个trait，里面有insert、remove、get等3个方法。
    CacheUpdatePolicy 定义了更新Cache的策略：Overwrite和Remove。
    然后对HashMap实现了Cache trait，这样就可以使用HashMap来作为Batch的Cache。
    
    2.2 
    Writable是个trait，里面有write、delete、write_with_cache、extend_with_cache、extend_with_option_cache等5个方法，
    其中write和delete是本体要实现的；
    write_with_cache、extend_with_cache以及extend_with_option_cache是在写本体的时候还要更新cache，写本体的方法就是调用write。
    区别是：write_with_cache是写单个value，extend_with_cache是写多个value，extend_with_option_cache是判断value的值来抉择对本体写还是删除。
    
    Readable是个trait，里面有read、read_with_cache、exists、exists_with_cache等4个方法。
    其中read和exists是本体要实现的，read是根据key从本体获取value，exists是根据key判断value是否存在。
    read_with_cache是先看看cache中有没有，有就直接返回，没有的话从本体中获取，并更新cache。
    exists_with_cache是先看看cache中有没有，有就返回true，没有的话就根据本体进行判断。
    
    最终Batch实现了Writable，满足KeyValueDB trait bound的KVDB实现了Readable。
    
    
3. 对Rust语言掌握的不深，上面的理解可能后续会更新。
    

