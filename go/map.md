# GO MAP

1. ### 结构体
```go
// A header for a Go map.
type hmap struct {
    // 元素个数，调用 len(map) 时，直接返回此值
    count     int
    flags     uint8
    // buckets 的对数 log_2
    B         uint8
    // overflow 的 bucket 近似数
    noverflow uint16
    // 计算 key 的哈希的时候会传入哈希函数(哈希种子)
    hash0     uint32
    // 指向 buckets 数组(bmap结构体), 大小为 2^B
    // 如果元素个数为0，就为 nil
    buckets    unsafe.Pointer
    // 扩容的时候，buckets 长度会是 oldbuckets 的两倍
    oldbuckets unsafe.Pointer
    // 指示扩容进度，小于此地址的 buckets 迁移完成
    nevacuate  uintptr
    extra *mapextra // optional fields, 避免某些bmap被回收
}

// buckets 指针, 指向存放kv的数组
// 每个桶最多装8个key
type bmap struct {
    topbits  [8]uint8 // 表示桶位置的有无
    keys     [8]keytype // key, value 分开存放是为了不用padding
    values   [8]valuetype
    pad      uintptr
    overflow uintptr // 当8个位置存放不下时可以指向多一个桶
}
```
2. ### hash函数:
程序启动时会检查cpu是否支持aes, 支持就使用aes hash, 不然就用memhash

3. ### 两种get操作:
     编译时动态调用不同的函数mapaccess1, mapaccess2 

4. ### 如何扩容:
    1. 装载因子: loadFactor = count / (2 ^ B), 源码定义阈值为6.5
    2. overflow的bucket数量超过阈值进行扩容
    


