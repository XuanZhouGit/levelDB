# 1. 简介
levelDB是Jeff Dean和Sanjay Ghemawat发起的开源key-value DB,支持读、写、删除这些基本功能，也支持快照(通过log file)，数据压缩(snappy compress)。目前有一些开源的数据库是基于levelDB实现的，比如，Tair ldb以及SSDB

levelDB的读写性能很好，参考官方benchmark数据(由于使用了cache，顺序读性能优于随机读)。比较适用于随机写操作密集或是顺序读较多的情况
## 1.1 levelDB用法
源代码: leveldb-1.15.0
levelDB的使用：
```
tar xvf FileName.tar leveldb-1.15.0
cd leveldb-1.15.0
make
```
就会在目录中生成libleveldb.a及libleveldb.so， 一个例子:
```
#include <iostream>
#include "leveldb/db.h"
int main()
{
    leveldb::DB* db;
    leveldb::Options options;
    options.create_if_missing = true;
     
    leveldb::Status status = leveldb::DB::Open(options, "/home/xuzhou/leveldb-1.15.0/test/testdb", &db);
    // put test
    char numStringPut[64];
    for (int i = 0; i < 10000; ++i)
    {
        sprintf(numStringPut, "%d" , i);
        std::string keyPut(numStringPut);
        std::string valuePut = keyPut;
        status = db->Put(leveldb::WriteOptions(), keyPut, valuePut);
        assert(status.ok());   
    }
    // get test
    char numStringGet[64];
    for (int i = 0; i < 10000; ++i)
    {
        sprintf(numStringGet, "%d" , i);
        std::string keyGet(numStringGet);
        std::string valueGet;
        status = db->Get(leveldb::ReadOptions(), keyGet, &valueGet);
        assert(status.ok());       
        std::cout << "keyGet:" << keyGet << " valueGet:" << valueGet << std::endl;       
    }  
    // delete test
    char numStringDelete[64];
    for (int i = 0; i < 10000; ++i)
    {
        sprintf(numStringDelete, "%d" , i);
        std::string keyDelete(numStringDelete);
        status = db->Delete(leveldb::WriteOptions(), keyDelete);
        assert(status.ok());       
    }  
     
    delete db;
    return 0;
}
```
编译运行：
```
bash-3.00$ g++ -o test -g main.cpp -lpthread -L./ -lleveldb -I/home/xuzhou/leveldb-1.15.0/include
bash-3.00$ ./test
bash-3.00$ pwd
/home/xuzhou/leveldb-1.15.0/test/testdb
bash-3.00$ ls
000003.log  CURRENT  LOCK  LOG  MANIFEST-000002
```
## 1.2 与其它数据库比较
### 1.2.1 Redis vs levelDB
-|Redis|levelDB
------------ | ------------- | -------------
相同点|持久化存储|写速度快
不同点|完全内存查找，速度快|随机查找性能差，可能产生磁盘IO，速度较慢
不同点|value类型不仅限于字符串（字符串列表，无序不重复的字符串集合，有序不重复的字符串集合，key和value都为字符串的hash表）|key-value均为字符串，可自定义Comparator
不同点|多语言支持|一个C++库
不同点|存储容量受限于内存，磁盘只作为备份|存储容量依赖硬盘，可支持TB级别数据

### 1.2.2 sqlite vs levelDB
1. b+ tree vs lsm tree
2. sql vs nosql

# 2. 整体结构
## 2.1 LSM tree
b+ tree是各种传统db及磁盘索引结构，它的缺点在于每次写操作都可能带来多次的磁盘IO(磁头的旋转和寻道都很耗时，所以顺序读写性能都会优于随机读写)，写开销很大，怎样避免磁盘IO呢，LSM tree(Log-Structured Merge-Tree)就是为了优化磁盘写的一种存储结构, 是由The Log-Structured Merge-Tree 提出的，主要是通过将随机写操作转换为内存写和log追加，再批量写入磁盘。levelDB就是基于这种存储结构实现的。
# 2.2 levelDB的整体结构
