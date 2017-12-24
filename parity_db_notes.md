# 对Parity DB的调研
在Parity的代码中可以看到有多个DB，都在**util**目录下，都以lib的形式存在，它们是：**kvdb，hashdb， memorydb，kv-rocksdb，kv-memorydb，journaldb**。它们之间关系紧密，我们要调研的JournalDB也在其中，下面按顺序介绍一下。 
> 备注：  
Parity代码版本是：  
commit d1bf0e0e62587980856e5f7a7fd1f5bba8a19c1b  
Date:   Thu Nov 30 10:35:49 2017 +0000


##  一. KeyValueDB 
**util/kvdb**中定义了KeyValueDB相关的内容：**trait KeyValueDB，struct DBTransaction，enum DBOp**。 DBOp定义了单条操作内容；DBTransaction定义了批量操作内容；KeyValueDB定义了KV数据库的操作方法。  

KeyValueDB可以简单的理解为是面向DBTransaction的，收到DBTransaction后解析并执行其中的操作内容。
1. DBOp  
DBOp是枚举类型，代表一条基本的KV数据库操作：Insert(插入)、InsertCompressed(插入压缩数据)、Delete(删除)。Insert和InsertCompressed都需要3项参数：column、key、value。 
```
/// Database operation.
#[derive(Clone, PartialEq)]
pub enum DBOp {
	Insert {
		col: Option<u32>,
		key: ElasticArray32<u8>,
		value: DBValue,
	},
	InsertCompressed {
		col: Option<u32>,
		key: ElasticArray32<u8>,
		value: DBValue,
	},
	Delete {
		col: Option<u32>,
		key: ElasticArray32<u8>,
	}
}
```

2. DBTransaction  
DBTransaction是结构体类型，代表着对KV数据库的批量操作，里面的内容是DBOp的Vector。好处是提升效率。  
```
/// Write transaction. Batches a sequence of put/delete operations for efficiency.
#[derive(Default, Clone, PartialEq)]
pub struct DBTransaction {
	/// Database operations.
	pub ops: Vec<DBOp>,
}
```

3. KeyValueDB  
KeyValueDB是一个trait，定义了通用的KV数据库模型。可以看出，它定义了内部有缓存的数据库，可以通过flush()方法将数据从buffer写入backing storage中。另外，KeyValueDB支持column families(列簇)。  
由于具有缓存，KeyValueDB一般可以简单理解为由硬盘（disk）数据库来实现，当然在内存中也可以进行实现，而且util/kv-memorydb确实也impl了这个trait，但是为了方便起见，我们一般理解为这个trait面向硬盘数据库，如rocksdb。  
KeyValueDB中定义的写操作都是基于DBTransaction的，也就是接收单元是DBTransaction，收到后进行解析，然后操作硬盘数据库。
```
impl DBTransaction {
	/// Create new transaction.
	pub fn new() -> DBTransaction {}

	/// Create new transaction with capacity.
	pub fn with_capacity(cap: usize) -> DBTransaction {}

	/// Insert a key-value pair in the transaction. Any existing value will be overwritten upon write.
	pub fn put(&mut self, col: Option<u32>, key: &[u8], value: &[u8]) {}

	/// Insert a key-value pair in the transaction. Any existing value will be overwritten upon write.
	pub fn put_vec(&mut self, col: Option<u32>, key: &[u8], value: Bytes){}

	/// Insert a key-value pair in the transaction. Any existing value will be overwritten upon write.
	/// Value will be RLP-compressed on flush
	pub fn put_compressed(&mut self, col: Option<u32>, key: &[u8], value: Bytes) {}

	/// Delete value by key.
	pub fn delete(&mut self, col: Option<u32>, key: &[u8]) {}
}

```
方法解释如下：  
* transaction:  新建一个空的DBTransaction
* get:          获取指定key的数据项
* get_by_prfix: 根据前缀找到数据项
* write_buffered: 将DBTransaction写入KV数据库的buffer，这里的buffer不是硬盘数据库的buffer
* write       : 将DBTransaction写入硬盘。实现过程是先调用write_buffered，然后调用flush。
* flush:         刷buffer入硬盘
* iter:         循环取出指定column的数据
* iter_from_prefix: 根据指定前缀找到column中的数据
* restore: 	     使用新的db来替换掉本来的db

## 二. HashDB
**util/hashdb**定义了Hash DB相关的内容：**trait HashDB， trait AsHashDB， type DBValue**。  
HashDB定义了基本的数据存储，Value是128字节长的数据，Key是Value的Hash值。
HashDB可以简单的理解为是面向DBValue的。  
1. DBValue  
DBValue是一种重命名类型，是128Bytes长的弹性数组。
```
/// `HashDB` value type.
pub type DBValue = ElasticArray128<u8>;
```

2. HashDB  
HashDB是一个trait，定义了基本的HashDB的操作。HashDB也是一种KV DB，Value是字节数据，Key是Value的32Bytes长的Keccak Hash。
```
/// Trait modelling datastore keyed by a 32-byte Keccak hash.
pub trait HashDB: AsHashDB + Send + Sync {
	/// Get the keys in the database together with number of underlying references.
	fn keys(&self) -> HashMap<H256, i32>;

	/// Look up a given hash into the bytes that hash to it, returning None if the
	/// hash is not known.
	fn get(&self, key: &H256) -> Option<DBValue>;

	/// Check for the existance of a hash-key.
	fn contains(&self, key: &H256) -> bool;

	/// Insert a datum item into the DB and return the datum's hash for a later lookup. Insertions
	/// are counted and the equivalent number of `remove()`s must be performed before the data
	/// is considered dead.
	fn insert(&mut self, value: &[u8]) -> H256;

	/// Like `insert()` , except you provide the key and the data is all moved.
	fn emplace(&mut self, key: H256, value: DBValue);

	/// Remove a datum previously inserted. Insertions can be "owed" such that the same number of `insert()`s may
	/// happen without the data being eventually being inserted into the DB. It can be "owed" more than once.
	fn remove(&mut self, key: &H256);
}
```
方法解释如下：
* keys:    获取所有的key
* get:     根据key获取value
* contains: 判断是否包含key
* insert:   插入数据
* emplace: 插入key和value
* remove: 删除指定数据项

3. HashDB与KeyValueDB的关系  
- 相同点：  
都是trait，都是Key-Value类型的数据库。  
- 区别：  
KeyValueDB是定义了具有缓存的数据库，更像*是对硬盘数据库的抽象；
HashDB只是简单的Key-Value，更像是对内存数据库的抽象。

>备注：  
*: 这里“更像”是指更符合一般的数据库实现方法，当然可以有其他的实现方法，就像kvdb-memorydb是内存中的数据库，但也实现了trait KeyValueDB。

## 三. MemoryDB
**util/memorydb**中定义了一种具体的内存数据库MemoryDB，里面的内容有：struct MemoryDB，MemoryDB自身的方法，MemoryDB对HashDB trait的实现。  

MemoryDB是一种基于内存的、可引用计数的HashDB的具体实现。它的本质是一种HashMap， Value的类型是元组(DBValue,  i32)，Key是Value的Hash值，Hash 函数是自定义的， 可以看出Value中不仅存储了数据，还存储了引用计数。
1. MemoryDB
MemoryDB本质上是一种自定义的HashMap。
```
#[derive(Default, Clone, PartialEq)]
pub struct MemoryDB {
	data: H256FastMap<(DBValue, i32)>,
}
```
2. MemoryDB的方法
```
impl MemoryDB {
	/// Create a new instance of the memory DB.
	pub fn new() -> MemoryDB {}

	/// Clear all data from the database.
	///
	/// # Examples
	/// ```rust
	/// extern crate hashdb;
	/// extern crate memorydb;
	/// use hashdb::*;
	/// use memorydb::*;
	/// fn main() {
	///   let mut m = MemoryDB::new();
	///   let hello_bytes = "Hello world!".as_bytes();
	///   let hash = m.insert(hello_bytes);
	///   assert!(m.contains(&hash));
	///   m.clear();
	///   assert!(!m.contains(&hash));
	/// }
	/// ```
	pub fn clear(&mut self) {}

	/// Purge all zero-referenced data from the database.
	pub fn purge(&mut self) {}

	/// Return the internal map of hashes to data, clearing the current state.
	pub fn drain(&mut self) -> H256FastMap<(DBValue, i32)> {}

	/// Grab the raw information associated with a key. Returns None if the key
	/// doesn't exist.
	///
	/// Even when Some is returned, the data is only guaranteed to be useful
	/// when the refs > 0.
	pub fn raw(&self, key: &H256) -> Option<(DBValue, i32)> {}

	/// Returns the size of allocated heap memory
	pub fn mem_used(&self) -> usize {}

	/// Remove an element and delete it from storage if reference count reaches zero.
	/// If the value was purged, return the old value.
	pub fn remove_and_purge(&mut self, key: &H256) -> Option<DBValue> {}

	/// Consolidate all the entries of `other` into `self`.
	pub fn consolidate(&mut self, mut other: Self) {}
}
```
代码注释中包含一个小例子，可以了解MemoryDB的基本使用方法。  
方法解释如下：
* new: 新建DB
* clear: 清空DB
* purge: 净化DB，清除引用计数为0的数据项
* drain:  取出内部的hashmap，并清除所在内存的状态
* raw:   取出里面原始的数据，即元组数据
* mem_used: 查看内存堆占用大小
* remove_purge: 清除数据项，如果引用为0，就彻底删除
* consolidate:  合并其他数据库

3. MemoryDB对trait HashDB的实现
```
impl HashDB for MemoryDB {
	fn get(&self, key: &H256) -> Option<DBValue> {}

	fn keys(&self) -> HashMap<H256, i32> {}

	fn contains(&self, key: &H256) -> bool {}

	fn insert(&mut self, value: &[u8]) -> H256 {}

	fn emplace(&mut self, key: H256, value: DBValue) {}

	fn remove(&mut self, key: &H256) {}
}
```
这是对trait HashDB的实现，功能介绍参见HashDB。

## 四. kvdb-rocksdb
### RocksDB
先简单介绍下RocksDB。RocksDB是Facebook开发的一种高效数据库软件，是建立在Google的LevelDB的基础上的，可以发挥快速存储（闪存）的潜力。它是一个C++库，用于存储KV，可以是任意大小的字节流。  
> RocksDB：A library that provides an embeddable, persistent key-value store for fast storage.

这句话是github 中RocksDB[主页](https://github.com/facebook/rocksdb)上的说明，翻译过来是：它是一个库，能够在快速存储设备上进行嵌入式的、持久化的存储key-value。  
这里有几个关键字：
+ **快速存储设备**：像flash（闪存）、RAM（内存）。RocksDB针对快速存储设备做了优化，使得存储性能更好。
+ **持久化存储**：有硬盘的支持。
+ **Key-Value**：存储的内容是Key和Value的键值对。
+ **库**：代码以C++库的形式体现。
+ **可嵌入**：与库的概念有些重叠，可以灵活地集成在软件中。

rocksdb上有wiki，如果要研究清楚rocksdb有哪些功能特性、如何使用等，需要花费较多时间，这个留待以后再深入学习。

### rust-rocksdb
由于RocksDB是C++库，Parity代码无法直接使用，于是做了一层封装，也就有了rust-rocksdb。   
crates.io上有rust-rocksdb的crate，但是parity使用的代码应该是在此基础上做了修改，github上代码路径在[这里](https://github.com/paritytech/rust-rocksdb#getting-an-iterator)。  
基本操作方法如下：
* open_default：以默认配置打开数据库。
* put：插入数据。
* get：获取数据。
* delete：删除数据。
* batch：批量插入，先通过put放入batch，然后使用write写入数据库。
* Iterator：迭代器可以实现批量获取，可以设置迭代开始的位置：Start、End、From（forward、reverse）。From类似于文件操作中的seek。
* snapshot：快照功能，可以保存时间点视图。

## 五. kvdb-memorydb
**util/kvdb-memorydb**中定义了一种在内存中实现了trait KeyValueDB的数据库：**struct InMemory**。这里的代码是实验性质的，并没有实际使用，参见下面注释中的说明。
```
/// A key-value database fulfilling the `KeyValueDB` trait, living in memory.
/// This is generally intended for tests and is not particularly optimized.
#[derive(Default)]
pub struct InMemory {
	columns: RwLock<HashMap<Option<u32>, BTreeMap<Vec<u8>, DBValue>>>,
}
```

## 六. journaldb

**util/journaldb**中定义了trait JournalDB以及4种具体实现：**struct ArchiveDB、struct EarlyMergeDB、struct OverlayRecentDB、struct RefCountedDB**。这4种实现都用到了Overlay和Backing。Overlay对应内存中的数据库，具体在这里是MemoryDB；Backing是实现了KeyValueDB  trait的数据库，跟之前描述的一样，可以简单理解为硬盘数据库，在代码中是kvdb-rocksdb。  

JournalDB是我们调研的对象，也是几乎上面所有DB的集成者。   从抽象的角度来看，JournalDB相当于一种更大的数据库，将Overlay和Backing都包含进来，统一管理。从名字上来看，JournalDB是一种“日志数据库”，会将针对数据库的所有操作先记录下来，然后在适当的时候将所有操作汇总成batch发送给Backing，由Backing执行这些操作。这样的好处是批量操作，效率高。JournalDB可以有不同的实现，Overlay和Backing各自存储什么内容、彼此之间如何交互，可以根据需求有各种各样的实现方式，所以才会有ArchiveDB、EarlyMergeDB等，不止这4种实现，可以存在很多种方式。  
缓存是个很普遍的概念。总体上来说，Overlay相当于Backing的缓存，两者之间存在同步的问题。每一层上还可能有自己的缓存，比如说，rocksdb是最底层的数据库，它维护着自己的缓存；kvdb-rocksdb是对rocksdb的封装，它也有自己的缓存，write_buffered就是写入自己的缓存中，而不是写入rocksdb的缓存中。所以说，整个的存储结构是比较复杂的。  

### JournalDB
JournalDB是一个trait，它的实现依赖于trait HashDB的实现，它本身也可以认为是一种HashDB。  
```
/// A `HashDB` which can manage a short-term journal potentially containing many forks of mutually
/// exclusive actions.
pub trait JournalDB: HashDB {
	/// Return a copy of ourself, in a box.
	fn boxed_clone(&self) -> Box<JournalDB>;

	/// Returns heap memory size used
	fn mem_used(&self) -> usize;

	/// Returns the size of journalled state in memory.
	/// This function has a considerable speed requirement --
	/// it must be fast enough to call several times per block imported.
	fn journal_size(&self) -> usize { 0 }

	/// Check if this database has any commits
	fn is_empty(&self) -> bool;

	/// Get the earliest era in the DB. None if there isn't yet any data in there.
	fn earliest_era(&self) -> Option<u64> { None }

	/// Get the latest era in the DB. None if there isn't yet any data in there.
	fn latest_era(&self) -> Option<u64>;

	/// Journal recent database operations as being associated with a given era and id.
	// TODO: give the overlay to this function so journaldbs don't manage the overlays themeselves.
	fn journal_under(&mut self, batch: &mut DBTransaction, now: u64, id: &H256) -> Result<u32, UtilError>;

	/// Mark a given block as canonical, indicating that competing blocks' states may be pruned out.
	fn mark_canonical(&mut self, batch: &mut DBTransaction, era: u64, id: &H256) -> Result<u32, UtilError>;

	/// Commit all queued insert and delete operations without affecting any journalling -- this requires that all insertions
	/// and deletions are indeed canonical and will likely lead to an invalid database if that assumption is violated.
	///
	/// Any keys or values inserted or deleted must be completely independent of those affected
	/// by any previous `commit` operations. Essentially, this means that `inject` can be used
	/// either to restore a state to a fresh database, or to insert data which may only be journalled
	/// from this point onwards.
	fn inject(&mut self, batch: &mut DBTransaction) -> Result<u32, UtilError>;

	/// State data query
	fn state(&self, _id: &H256) -> Option<Bytes>;

	/// Whether this database is pruned.
	fn is_pruned(&self) -> bool { true }

	/// Get backing database.
	fn backing(&self) -> &Arc<kvdb::KeyValueDB>;

	/// Clear internal strucutres. This should called after changes have been written
	/// to the backing strage
	fn flush(&self) {}

	/// Consolidate all the insertions and deletions in the given memory overlay.
	fn consolidate(&mut self, overlay: ::memorydb::MemoryDB);

	/// Commit all changes in a single batch
	#[cfg(test)]
	fn commit_batch(&mut self, now: u64, id: &H256, end: Option<(u64, H256)>) -> Result<u32, UtilError> {
		let mut batch = self.backing().transaction();
		let mut ops = self.journal_under(&mut batch, now, id)?;

		if let Some((end_era, canon_id)) = end {
			ops += self.mark_canonical(&mut batch, end_era, &canon_id)?;
		}

		let result = self.backing().write(batch).map(|_| ops).map_err(Into::into);
		self.flush();
		result
	}

	/// Inject all changes in a single batch.
	#[cfg(test)]
	fn inject_batch(&mut self) -> Result<u32, UtilError> {
		let mut batch = self.backing().transaction();
		let res = self.inject(&mut batch)?;
		self.backing().write(batch).map(|_| res).map_err(Into::into)
	}
}
```
> 备注：  
	一般来说，数据库是通用的，所以era和id可以是任意数据，只要有era是随着时间增长的，id可以唯一的标识数据项。具体到代码的使用中，era是块的高度，id使用的是块 header的hash。

### OverlayDB
OverlayDB是最基础的DB，有着原始的存储模型：1个Overlay，1个Backing。在Parity代码中它并没有被使用，似乎是为其他DB打基础。
1. 结构体
```
/// Implementation of the `HashDB` trait for a disk-backed database with a memory overlay.
///
/// The operations `insert()` and `remove()` take place on the memory overlay; batches of
/// such operations may be flushed to the disk-backed DB with `commit()` or discarded with
/// `revert()`.
///
/// `lookup()` and `contains()` maintain normal behaviour - all `insert()` and `remove()`
/// queries have an immediate effect in terms of these functions.
#[derive(Clone)]
pub struct OverlayDB {
	overlay: MemoryDB,
	backing: Arc<KeyValueDB>,
	column: Option<u32>,
}
```
从上面可以看出：OverlayDB包含了MemoryDB和KeyValueDB，当然还有KeyValueDB的column，对应DB中的列。


2. 方法
```
impl OverlayDB {
	/// Create a new instance of OverlayDB given a `backing` database.
	pub fn new(backing: Arc<KeyValueDB>, col: Option<u32>) -> OverlayDB {}

	/// Create a new instance of OverlayDB with an anonymous temporary database.
	#[cfg(test)]
	pub fn new_temp() -> OverlayDB {}

	/// Commit all operations in a single batch.
	#[cfg(test)]
	pub fn commit(&mut self) -> Result<u32> {}

	/// Commit all operations to given batch.
	pub fn commit_to_batch(&mut self, batch: &mut DBTransaction) -> Result<u32> {}

	/// Revert all operations on this object (i.e. `insert()`s and `remove()`s) since the
	/// last `commit()`.
	pub fn revert(&mut self) { self.overlay.clear(); }

	/// Get the number of references that would be committed.
	pub fn commit_refs(&self, key: &H256) -> i32 { self.overlay.raw(key).map_or(0, |(_, refs)| refs) }

	/// Get the refs and value of the given key.
	fn payload(&self, key: &H256) -> Option<(DBValue, u32)> {}

	/// Put the refs and value of the given key, possibly deleting it from the db.
	fn put_payload_in_batch(&self, batch: &mut DBTransaction, key: &H256, payload: (DBValue, u32)) -> bool {}
}
```
方法解释如下：
* new             :创建一个新的实例
* new_temp        :使用kvdb_memorydb创建一个新的实例
* commit          :打包成batch，然后写入backing数据库中
* commit_to_batch :取出overlay中的数据打包batch（取出之后数据不再存在），返回batch中操作的个数。
* revert		   : 清理overlay
* commit_refs	   : 获取指定key的引用次数
* payload          : 去硬盘数据库中取出指定值
* put_payload_in_batch: 将数据放入batch中，如果ref大于1，就put，如果为0，就delete
	
从payload和put_payload_in_batch的实现中可以看出，存入batch的value是rlp编码，长度为2, 0号位是引用计数rc，1号位是data。

3. HashDB方法
```
impl HashDB for OverlayDB {
	fn keys(&self) -> HashMap<H256, i32> {}

	fn get(&self, key: &H256) -> Option<DBValue> {}

	fn contains(&self, key: &H256) -> bool {
		// return ok if positive; if negative, check backing - might be enough references there to make
		// it positive again.
		let k = self.overlay.raw(key);
		match k {
			Some((_, rc)) if rc > 0 => true,
			_ => {
				let memrc = k.map_or(0, |(_, rc)| rc);
				match self.payload(key) {
					Some(x) => {
						let (_, rc) = x;
						rc as i32 + memrc > 0
					}
					// Replace above match arm with this once https://github.com/rust-lang/rust/issues/15287 is done.
					//Some((d, rc)) if rc + memrc > 0 => true,
					_ => false,
				}
			}
		}
	}

	fn insert(&mut self, value: &[u8]) -> H256 { self.overlay.insert(value) }
	fn emplace(&mut self, key: H256, value: DBValue) { self.overlay.emplace(key, value); }
	fn remove(&mut self, key: &H256) { self.overlay.remove(key); }
}
```
方法解释如下：
* keys:    获取所有的key和引用计数rc  
实现：先取出backing数据库中的值和rc，然后在此基础上插入overlay中的值和rc
* get:     获取指定key的数据  
		    实现：先在overlay中查找，如果rc>0即可返回，否则再去backing中查找
* contains: 判断是否包含指定key的数据  
		    实现：如果overlay中rc>0，即可返回true，否则再去backing中查找	
* insert:   插入数据
* emplace: 插入Value和key
* remove：删除指定key。

插入和删除操作，insert、emplace、remove，都是在overlay上直接操作。

4. 总结  
OverLayDB的操作是比较简洁的，存储模型总结如下：
* OverlayDB包含了Overlay(MemoryDB)和Backing(KeyValueDB)，Overlay充当Backing的缓冲。
* MemoryDB和KeyValueDB都是K-V型数据库，MemoryDB中存的是128字节长的数据和引用计数；KeyValueDB一般是具体的硬盘数据库,里面的K和V可以是任意的字节流。
* 每一项操作都会先放入MemoryDB，然后由MemoryDB打包成batch，即DBTRansaction结构，传递给KeyValueDB的实现(这里是kvdb-rocksdb)，然后由它解析batch对硬盘数据库进行插入删除操作。
* insert和remove发生在memory overlay上，所以会有即时有效的效果；commit会将缓冲打包成batch写入硬盘数据库，revert会将这些缓冲删除；另外还有lookup、contains等查找功能，查询的时候先查看overlay，如果没有再去backing中找。

从名字上来看，“OverlayDB”，是指基本操作都是在Overlay上进行操作的DB，只有调用commit之后才会写入硬盘数据库。这种操作模式有些类似于Git，先在本地(MemoryDB)进行修改，然后再提交到仓库(KeyValueDB)中去。其实所有的JournalDB应该都是这种操作模式，只是具体操作细节不太相同。  


### ArchiveDB
ArchiveDB是在OverlayDB的基础上做了些改进，是个实际可用的JournalDB。

1. 结构体
```
/// Implementation of the `HashDB` trait for a disk-backed database with a memory overlay
/// and latent-removal semantics.
///
/// Like `OverlayDB`, there is a memory overlay; `commit()` must be called in order to
/// write operations out to disk. Unlike `OverlayDB`, `remove()` operations do not take effect
/// immediately. As this is an "archive" database, nothing is ever removed. This means
/// that the states of any block the node has ever processed will be accessible.
pub struct ArchiveDB {
    overlay: MemoryDB,
    backing: Arc<KeyValueDB>,
    latest_era: Option<u64>,
    column: Option<u32>,
}
```
可以看出，ArchiveDB包含MemoryDB、KeyValueDB、latest_era、column。  

2. 方法
```
impl ArchiveDB {
	/// Create a new instance from a key-value db.
	pub fn new(backing: Arc<KeyValueDB>, col: Option<u32>) -> ArchiveDB {}

	fn payload(&self, key: &H256) -> Option<DBValue> {}
}
```
方法解释如下：
* new: 新建一个ArchiveDB实例  
	实现：从backing中取出latest_era
* payload: 根据指定key从backing中获取数据项

3. HashDB方法
```
impl HashDB for ArchiveDB {
	fn keys(&self) -> HashMap<H256, i32> {}

	fn get(&self, key: &H256) -> Option<DBValue> {}

	fn contains(&self, key: &H256) -> bool {}

	fn insert(&mut self, value: &[u8]) -> H256 {}

	fn emplace(&mut self, key: H256, value: DBValue) {}

	fn remove(&mut self, key: &H256) {}
}
```
方法解释如下：
* keys:     获取所有的key和引用计数rc  
            与OverlayDB的实现有些类似，只是从backing中取出的rc固定为1
* get:      获取指定key的数据项  
            先从overlay中获取，没有的话再调用payload中找
* contains: 判断是否包含指定key
			直接调用get方法
* insert:   插入数据
* emplace:  插入key和value
* remove:   删除指定key的数据项

插入和删除操作，insert、emplace、remove，都是在overlay上直接操作。

4. JournalDB方法
```

impl JournalDB for ArchiveDB {
	fn boxed_clone(&self) -> Box<JournalDB> {}

	fn mem_used(&self) -> usize {}

	fn is_empty(&self) -> bool {}

	fn journal_under(&mut self, batch: &mut DBTransaction, now: u64, _id: &H256) -> Result<u32, UtilError> {}

	fn mark_canonical(&mut self, _batch: &mut DBTransaction, _end_era: u64, _canon_id: &H256) -> Result<u32, UtilError> {}

	fn inject(&mut self, batch: &mut DBTransaction) -> Result<u32, UtilError> {}

	fn latest_era(&self) -> Option<u64> { self.latest_era }

	fn state(&self, id: &H256) -> Option<Bytes> {}

	fn is_pruned(&self) -> bool { false }

	fn backing(&self) -> &Arc<KeyValueDB> {}

	fn consolidate(&mut self, with: MemoryDB) {}
}
```
方法解释如下：
* boxed_clone: 返回一份拷贝
* mem_used:    返回Overlay占用内存大小
* is_empty:    判断数据库是否为空  
			  实现：使用latest_era进行判断
* journal_under: 将Overlay中的数据打包进batch，即DBTransaction结构  
                 实现：  
                     - 将Overlay中的数据取出，rc>0添加put操作；rc<0添加delete操作；  
                     - 如果需要更新latest_era的话，同时在batch中插入LATEST_ERA_KEY.
* mark_canonical: 标记canonical。  
                实现：由于不会进行删除操作，所以直接返回OK(0)
* inject: 		  也是将Overlay中的数据打包成batch，只是会去硬盘上查看一下
* latest_era:     获取latest_era
* state:		  根据Patricia key从硬盘上获取数据
* is_pruned:	  判断是否支持删除操作，这里直接返回false
* consolidate:	  合并Overlay

5. 总结  
ArchiveDB与OverlayDB比较一下，可以看出一些相似的地方，ArchiveDB是在OverlayDB的基础上做了修改。存储模型总结如下：
* ArchiveDB包含overlay、backing、column、latest_era选项
* HashDB的insert、emplace、remove操作是在Overlay上操作的，所以会有即时的效果；也需要调用commit才能将操作写入backing（*这里的commit，应该是需要调用backing的commit，因为ArchiveDB的代码中没有commit函数*）。
* 不支持prune功能。remove操作是在Overlay上进行的，因为ArchiveDB会保存所有的state，所以并不会真正的删除，任何之前的状态都可以被访问到。journal_under中碰到rc< 0的情况，并不会在batch中添加delete操作，而inject会进行delete。

backing中存储的内容，需要注意是：
* OverlayDB的backing中存储的Value是rlp编码值，0号位是引用计数rc，1号位是数据；ArchiveDB的backing中存储的Value就是Value，不包含引用计数（所以在new的时候rc默认设置为1）.
* ArchiveDB会存储latest_era，key是固定的LATEST_ERA_KEY，Value是latest_era的rlp编码值。

顾名思义，ArchiveDB，是会将所有数据都进行存档的数据库。操作模式也是类似于Git，先在Overlay上进行修改，然后写入backing中由backing去处理。

### OverlayRecentDB
OverlayRecentDB相对于前两个DB，是复杂了很多，我们先分析文件的注释，然后分析代码。引用格式的文字是对源文件的翻译，紧接着的文字是分析。

#### 注释说明
1. 简介
> OverlayRecentDB: 'JournalDB' over in-memory overlay  

OverlayRecentDB在overlay中有一个DB用来存储Journal。这里的Journal的内容类似于快照，详细解释参见下面的分析。  
> OverlayRecentDB跟OverlayDB一样，有一个memory overlay；需要主动调用commit才会将数据写入disk；OverlayDB的Overlay和Backing的交互模型比较简单，所有操作都先在Overlay上进行，remove等操作是直接在Overlay上操作的，所以会立即生效，这里remove不会立即生效，但是有些年代（基本是线性的，但是没有确定的规律，比较随机）的数据可能在remove之前就已经被删除了。

2. 总体结构  
> OverlayRecentDB有2个memory overlay：transaction overlay和history overlay。Transaction overlay包含了当前的transaction data，会在commit的时候合并入history overlay；History overlay包含了hisory period的所有数据，在commit的时候，如果其中的node已经变成ancient，会被写入disk。  

可以看出这是2层overlay，上层是transaction overlay，缓存了transaction；下层是history overlay。在commit的时候transaction overlay会先merge到history overlay，然后history overlay中ancient的node会被写入disk。可以参见下图。  
```
graph TB
A(Transaction Overlay).->B[Transactions]
B-->C[History]
D(History Overlay).->C
C-->E[Disk]
```

> 另外，内存中还维护着1个journal，也会存入disk，它里面包含了history period中所有的insertions和removals，下面的图示是根据JournalEntry结构体画的。这个是用来跟踪那些已经超过history scope但是必须被写入disk的data nodes。

这一部分是区别与OverlayDB和AchiveDB的根本，详细内容参见下面。  

3. commit流程
> Commit流程如下：
> * 根据transaction overlay创建1个新的journal record
> * 将transaction overlay中的所有node插入history  overlay，如果已经存在的话，就增加引用计数rc
> * 清除 transaction overlay
> * 对于1个canonical的journal record，如果它变成ancient，将其中的insertions写入disk
> * 对于每1个journal record，如果超出了history scope，也就是变成ancient，将它在history overlay中的insertions删除，降低引用计数rc，如果rc==0，删除这个entry，
> * 对于1个canonical的Journal record，如果它变成ancient，遍历其中的removals，如果key不在history overlay中，清理掉这条removal
> * 将ancient record从Memory和disk中清除掉。

4. 总结  
根据注释能够稍微有所了解，但是没有细节，对于整个的流程是比较困惑的。Transaction Overlay，History Overlay，Journal，Disk，各自存储什么内容，彼此之间如何进行交互？这些是接下来在看代码的过程中要掌握的。

#### 代码说明
1. 结构体
```
pub struct OverlayRecentDB {
	transaction_overlay: MemoryDB,
	backing: Arc<KeyValueDB>,
	journal_overlay: Arc<RwLock<JournalOverlay>>,
	column: Option<u32>,
}
```
看上去只比OverlayDB多了一个journal_overlay，但是这个journal_overlay是个比较复杂的东西。注释中说过，OverlayRecentDB有2个overlay和1个journal，但是在结构体中只看到1个transaction overlay和1个journal_overlay，那么journal_overlay应该是包含History overlay和Journal的内容的。  
下面看一下JournalOverlay的结构。  
```
#[derive(PartialEq)]
struct JournalOverlay {
	backing_overlay: MemoryDB, // Nodes added in the history period
	pending_overlay: H256FastMap<DBValue>, // Nodes being transfered from backing_overlay to backing db
	journal: HashMap<u64, Vec<JournalEntry>>,
	latest_era: Option<u64>,
	earliest_era: Option<u64>,
	cumulative_size: usize, // cumulative size of all entries.
}
```
JournalOverlay是个比较复杂的结构，里面有3个HashMap（MemoryDB也是）和2个era:
* backing_overlay是MemoryDB，就是之前说的History Overlay；
* pending_overlay似乎是backing_overlay与backing db之间的缓冲；
* journal的HashMap中Value是JournalEntry的Vector，这个是记录Journal record的，每次commit的时候都会有1个journal record。
* latest_era和earliest_era是用来保持History的，相当于2个指针，有点类似于环形缓冲区的2个指针，是用来维持History的。
* cumulative_size是用来统计backing_overlay中insertions的Value的累计长度，后面会说明一下是怎么计算的。

JournalEntry结构体如下所示，里面包含id，insertions，deletions，后两个都是key的Vector。
```
#[derive(PartialEq)]
struct JournalEntry {
	id: H256,
	insertions: Vec<H256>,
	deletions: Vec<H256>,
}
```

2. 方法
JournalEntry实现了HeapSizeOf trait，能够获取堆内存占用大小  
OverlayRecentDB实现了Clone trait，能够直接clone  
```
impl OverlayRecentDB {
	/// Create a new instance.
	pub fn new(backing: Arc<KeyValueDB>, col: Option<u32>) -> OverlayRecentDB {}

	#[cfg(test)]
	fn can_reconstruct_refs(&self) -> bool {}

	fn payload(&self, key: &H256) -> Option<DBValue> {}

	fn read_overlay(db: &KeyValueDB, col: Option<u32>) -> JournalOverlay {}
}
```

OverlayRecentDB自身实现的方法比较少，有new、payload、read_overlay、can_reconstruct_refs，功能也比较好理解：
* new：新建一个OverlayRecentDB实例，也可以理解为初始化；
* onstruct_refs：判断能否重构journal_overlay成功；
* payload：从硬盘中根据key获取value；
* read_overlay：从硬盘中获取信息来构建journal_overlay，供new使用。

从上面这些操作的实现中可以推导出很多内容：  
* new的实现  
OverlayRecentDB里面包含4项数据：transaction_overlay，backing，journal_overlay，column。这些项中，backing和column是方法入参传入进来的，transaction_overlay是新建的MemoryDB，只有journal_overlay是从backing中通过调用read_overlay方法读取出来的，下面看read_overlay的实现。

* read_overlay的实现  
read_overlay的目的是构建出JournalOverlay出来。JournalOverlay中包含6项：backing_overlay，pending_overlay，journal，latest_era，earliest_era，cumulative_size。其中pending_overlay是新构造的HashMap。我们先看流程，再分析每个数据项的由来。流程如下：  
步骤1：初始化journal、overlay、latest_era、earliest_era、cumulative_size；  
步骤2：根据LATEST_ERA_KEY作为key从db中取出value来，RLP解码之后就是era，也就是latest_era；  
步骤3：根据era、index、PADDING进行RLP编码来构造key，从db中取数据，取出的value也是RLP编码格式，里面有id、insertions、deletions，从insertions中取出k和v放入overlay中，同时在cumulative_size中累计value的长度，根据insertions和deletions构造JournalEntry，放入journal中。  
步骤4：index+=1，进入步骤3。这样会取出同一个era下的所有的insertions和deletions放入overlay和journal中。  
步骤5：era-=1，进入步骤3和4。这样会取出所有记录的era下的insertions和deletions。最后的那个era作为earliest_era。  
步骤6：根据上面的流程中得到的journal、overlay、latest_era、earliest_era、cumulative_size，再新建一个pending_overlay，构造一个JournalOverlay返回。  

根据流程可以推导出存储在db中的一些数据的信息：  
*  latest_era：  
内存中：Some(u64)；  
db中存储：key是LATEST_ERA_KEY，value是latest_era的RLP编码  
* journal的存储：  
内存中：journal是一个HashMap<u64, Vec<JournalEntry>>，entry中的value是JournalEntry的Vector。换句话说，1个journal的entry对应一个era下所有的记录，包含一系列的JournalEntry，JournalEntry之间根据id进行区分，里面包含着insertions和deletions。  
db中存储：如下图所示，era和index合起来作为1条JournalEntry存储时的key，PADDING是10个字节的0，era是8个字节，index是4个字节，所以在journal_overlay中查找时用的key都需要经过to_short_key处理，取前12个字节。  
需要注意的是：  
a) id和index不是同一个数据，id用来区分每一个entry（代码调用时id的值是header的hash），index是entry的个数。  
b) 内存中JournalEntry的insertions是Vector<H264>，即不包含value的（value存在overlay中），而在db中存储的insertions是包含value的。

![image](https://github.com/EighteenZi/Parity_DB_learning/blob/master/JournalEntry_storage_struct.png?raw=true)

* payload的实现
payload的功能是根据key从db中取出value。
从实现中可以看出：数据在db中是按照如下格式存储的：  
```
graph LR
A(key: H264)-->B(value:Transactions)
```
DBValue是如下类型：
```
pub type DBValue = ElasticArray128<u8>;
```
* can_reconstruct_refs的实现
调用read_overlay重新构建journal_overlay，然后与内存中保存的journal_overlay对比，判断是否一致。  
这个方法只是测试使用。

3. JournalDB方法
```
impl JournalDB for OverlayRecentDB {
	fn boxed_clone(&self) -> Box<JournalDB> {}

	fn mem_used(&self) -> usize {}

	fn journal_size(&self) -> usize {}

	fn is_empty(&self) -> bool {}

	fn backing(&self) -> &Arc<KeyValueDB> {}

	fn latest_era(&self) -> Option<u64> { self.journal_overlay.read().latest_era }

	fn earliest_era(&self) -> Option<u64> { self.journal_overlay.read().earliest_era }

	fn state(&self, key: &H256) -> Option<Bytes> {}

	fn journal_under(&mut self, batch: &mut DBTransaction, now: u64, id: &H256) -> Result<u32, UtilError> {}

	fn mark_canonical(&mut self, batch: &mut DBTransaction, end_era: u64, canon_id: &H256) -> Result<u32, UtilError> {}

	fn flush(&self) {}

	fn inject(&mut self, batch: &mut DBTransaction) -> Result<u32, UtilError> {}

	fn consolidate(&mut self, with: MemoryDB) {}
}
```
JournalDB方法较多，不像HashDB那样容易分类，有：
* boxed_clone：返回一份数据库的拷贝
* mem_used：堆内存占用大小
* is_empty： 判断数据库是否为空
* journal_size：返回journal大小
* backing：获取backing
* latest_era：获取latest_era
* earliest_era：后去earliest_era
* state: 根据key的前缀获取value
* flush：清理缓存
* inject：填写batch
* consolidate：合并transaction_overlay
* journal_under：填写batch
* mark_canonical：

下面一一介绍实现方法，以及我们从实现当中推导出的信息。  
1）boxed_clone的实现  
OverlayRecentDB已经实现了trait Clone，所以直接调用clone方法即可。

2）mem_used的实现  
获取堆内存占用大小，会累计transaction_overlay，journal_overlay中的backing_overlay、pending_overlay、journal的大小。缓存结构也是由这4个HashMap组成的。

3）journal_size  
直接返回journal_overlay中的cumulative_size。之前在OverlayRecentDB：：new的时候，会计算写入journal_overlay.backing_overlay中value的长度，后续是如何维持的，还需要接着看下面的代码。

4）is_empty  
判断数据库是否为空，使用的方法是去backing中获取LATEST_ERA_KEY。
这样的实现方法，说明latest_era起着关键性的作用。latest_era的写入机制需要继续分析。

5）backing、latest_era、earliest_era  
这3个方法很简单，只是获取backing、latest_era、earliest_era.

6）state  
这个方法的使用还不是特别清楚。  
流程是：  
先将key截短，取前12bytes，即era+index；  
在journal_overlay的backing_overlay上查找；  
如果找不到，在journal_overlay的pending_overlay上查找；  
如果找不到，在backing中调用get_by_prefix查找。  
  
这个方法结合read_overlay，我们知道了transaction_overlay和journal_overlay中的key的不同，journal_overlay中的key是12字节长的，是由8字节的era和4字节的index组成的。

7）journal_under的实现  
要想理解OverlayRecentDB的真正内涵，最需要搞明白的就是journal_under和mark_canonical。
Journal_under的流程如下：  
步骤1. 获取journal_overlay；  
步骤2. 清掉journal_overlay的pending_overlay  
步骤3. 取出transaction_overlay的所有数据，将所有rc>0的key放入inserted_keys数组中，将所有rc<0的key放入removed_keys数组中;将所有rc>0的key和Value放入insertions数组中；  
步骤4. 根据id、insertions、removed_keys进行RLP编码，构建db中存储的Value。  
构建的过程中insertions的构建是较复杂的（参考图七），将insertions中每一条insertion的k和v取出：  
1）参与到RLP编码中，
2）更新journal_overlay.cumulative_size，
3）放入journal_overlay.backing_overlay中。  
步骤5. 根据era、index、PADDING进行RLP编码，构建db存储中的key  
		   index是当前Journal中entries的数目，PADDING就是10个字节的0.
		步骤6. 对key和Value使用put_vec方法放入batch中
		步骤7. 更新latest_era和earliest_era
		步骤8. 新增journal的entry。

总结一下，journal_under方法中，会有一系列的操作，图三和图四中的数据基本都会涉及。首先，journal_overlay.pending_overlay会被清空；然后transaction_overlay中的数据会被取出，然后被打成各种包放入backing_overlay、journal、disk中，放入3个地方的内容是不同的：backing_overlay中放入short_key和value；journal中放入insertions和deletions，insertions中没有value；disk中放入rlp编码的key和value，具体内容参见图七。更新backing_overlay的时候会更新cumulative_size。然后根据era更新latest_era和earliest_era。  
每次journal_overlay的journal都会插入1个JournalEntry，1个era对应1系列的JournalEntry。

8）mark_canonical  
入参有end_era，canon_id，流程如下：  
步骤1. 取出journal_overlay；  
步骤2. 从journal_overlay.journal中取出end_era下包含的所有JournalEntry，然后开始遍历：  
		a. 删除db中的JournalEntry记录（图七）；  
b. 初始化canon_insertions、canon_deletions、overlay_deletions；  
c. 如果journalEntry.id等于canon_id，  
将JournalEntry中的insertions全部取出，开始遍历：  
根据insertion中的key在backing_overlay中获取出value和rc，如果rc>0，放入canon_insertions中；  
将JournalEntry中的deletions放入canon_deletions中；  
步骤3. 将JournalEntry中的deletions放入overlay_deletions；  
步骤4. 将canon_insersions中的数据放入batch中重新插入，同时也在pending_overlay中放置一份；  
步骤5. 根据overlay_deletions，更新backing_overlay和cumulative_size  
步骤6. 根据canon_deletions，如果backing_overlay不再包含指定的key，在db中也删除  
步骤7. 删除journal中end_era对应的所有JournalEntry  
步骤8. 更新earliest_era  
		
9）总结：  
每次journal_under，在db中都是插入如图七所示的数据，包含所有的insertions和deletions，如果有多次journal_under，每次db中存储的数据根据index作为区分。
最后在mark_canonical的时候会将db中指定era的所有记录的数据删除，重新插入指定id的一次记录，然后由db来执行。

4. HashDB方法
```
impl HashDB for OverlayRecentDB {
	fn keys(&self) -> HashMap<H256, i32> {}

	fn get(&self, key: &H256) -> Option<DBValue> {}

	fn contains(&self, key: &H256) -> bool {}

	fn insert(&mut self, value: &[u8]) -> H256 {}
	fn emplace(&mut self, key: H256, value: DBValue) {}
	fn remove(&mut self, key: &H256) {}
}
```
HashDB方法包含基本的插入（insert、emplace），删除（remove），获取（keys、get、contains）等接口。  
* 2个插入操作：  
		insert是插入value，返回key；  
		emplace是插入key和value，没有返回值。  
* 1个删除操作：  
		remove  
		注：insert、emplace、remove等3个操作都是在transaction_overlay上操作的。  
* 3个获取操作：  
		keys：获取所有的key及其引用计数，形成一个HashMap返回；  
		get： 根据key找到Value；  
		contains：判断是否包含key。

1）keys的实现  
遍历backing中指定column下所有的key，引用计数都设为1，形成HashMap；  
遍历transaction_overlay，获取所有的key，在HashMap的基础上插入。  
2）get的实现  
先在transaction_overlay中查找，如果rc>0，直接返回相应的Value；  
再去journal_overlay中的backing_overlay中查找；  
如果没有找到，再去journal_overlay的pending_overlay中查找；  
如果没有找到，去backing中查找。

可见OverlayRecentDB有很多层缓冲。  
另外，transaction_overlay中的key与journal_overlay中的key不同，后者是前者的头12bytes，所以叫short_key。

### EarlyMergeDB & RefCountedDB
这2个DB与OverlayRecentDB有些相似，也会在DB上存储Journal，只是具体操作的方式不同。

### JournalDB的使用 

JournalDB中最关键的功能是journal_under，即将Overlay中的数据取出放入batch中；与prune相关的功能是mark_canonical。在代码中journal_under和mark_canonical有多处调用，这里选择一个典型的例子：commit_block：  
commit_block的流程如下：  
->新建batch；  
-> check_epoch_end_signal: 修改header等并在batch中新添操作；  
-> journal_under: 将JournalDB中的内容添加到batch；  
-> chain.insert_block: 将block的数据添加batch；  
-> traced.read().import: 将tracedb中的内容添加到batch；  
-> db.read().write_buffered: 写入db的buffer中；  
-> chain.commit: chain数据的整理  
-> check_epoch_end:   
-> update_last_hashes: 更新hash  
-> prune_anient：清理state。  

这个流程中跟JournalDB紧密相关的是journal_under、prune_anient。journal_under比较简单，就是将Overlay中的数据写入batch中。



下面看一下prune_anient的流程。  
->获取latest_era；  
->进入循环  
  ->判断是否需要pruning  
  ->获取earliest_era，判断是否 earliest_era + history <= latest_era  
    ->如果是，新建batch  
    ->jounal_db.mark_canonical  
    ->db.read().write_buffered(batch)： 将batch写入数据库的buffer  
    ->journal_db.flush: 调用journal_db的flush函数。  

mark_canonical的工作主要是在batch中添加delete操作，write_buffered是将batch写入buffer，在适当的时候会有flush操作将buffer写入db中，journal_db.flush是清除juornabl_db内部的buffer，相当于clear操作。  


## 总结
上面描述了Parity中util目录下的所有DB，总结下来，给我的感觉是：Parity对DB的使用是非常充分的，代码中使用了很多的DB以及对DB的封装，几乎所有存储都提供了DB的抽象。这样的好处是：对于上层来说使用方便；不好的地方是：存储模型是较复杂的，有很多级的缓存。  
如果我们充分掌握了DB的使用，也可以定制自己的DB。