# Project 2: Log-structured file I/O

## 日志文件I/O

### 目标

- [x] 鲁棒的错误处理
- [x] 序列化/反序列化
- [x] 用标准文件API向磁盘写入日志
- [x] 内存和磁盘中键值对的一致性
- [x] 日志压缩

## Intro

- bitcask储存算法的简化版
  - 简洁
  - 高效
- WAL日志
  - 初始化时读出日志，内存中重建
  - 写入时写入磁盘中写入键值对，内存中写入键和偏移量
  - 日志压缩

## Interface

- 命令行
  - 无状态
    - 加载索引
    - 执行命令
    - 退出
  - `kvs set <KEY> <VALUE>`
    - 创建包含key和value的set日志条目
    - 序列化命令并追加写入日志
    - 成功则返回0
    - 发生错误输出错误信息并以非零返回值退出
  - `kvs get <KEY>`
    - 遍历日志重建内存索引
    - 检查key是否存在
    - 不存在则输出`Key not found`并以非0返回值退出
    - 成功则读出并输出日志中该key对应的日志条目
    - 返回值0
  - `kvs rm <KEY>`
    - 遍历日志重建内存索引
    - 检查key是否存在
    - 不存在则输出`Key not found`并以非0返回值退出
    - 序列化命令并追加写入日志
    - 返回值0
- API
  - 有状态
    - 加载索引
    - 执行多次API调用
    - `drop`
  - `KvStore::set(&mut self, key: String, value: String) -> Result<()>`
    - 没有成功写入时返回错误
    - 日志中写入`set`
    - 内存中保存log pointer
  - `KvStore::get(&mut self, key: String) -> Result<Option<String>>`
    - 键不存在返回`None`
    - 没有成功读取时返回错误
  - `KvStore::remove(&mut self, key: String) -> Result<()>`
    - 键不存在/没有成功删除时返回错误
    - 日志中写入`rm`
    - 内存中删除键值对
  - `KvStore::open(path: impl Into<PathBuf>) -> Result<KvStore>`
    - 打开给定目录下的`KvStore`
    - 遍历日志重建索引
  - 压缩
    - 当未压缩键值对超过阀值

## 错误处理

- combinators
  - `ok_or(err) -> Result`
    - `Option -> Result`
  - 

- 问题

  - 多种错误的合成
  - 追踪溯源

- 不要用字符串

  - clutter your code
  - 信息丢失

- [`pub trait Error: Debug + Display {}`](http://doc.rust-lang.org/std/error/trait.Error.html)

  - 使用`Display` trait转换成字符串输出
  - `fn source(&self) -> Option<&(dyn Error + 'static)>`

- `impl From<S> for T {}`

  - S => T
  - `t = s.into()`

- `std::io::Error`

  - ```rust
    pub struct Error {
        repr: Repr,
    }
    enum Repr {
        Os(i32),
        Simple(ErrorKind),
        Custom(Box<Custom>),
    }
    struct Custom {
        kind: ErrorKind,
        error: Box<dyn error::Error + Send + Sync>,
    }
    pub enum ErrorKind {
        // ...
    }
    ```

  - Simple: 只知道类型

  - Custom: 知道底层错误类型和底层错误指针

## 结构

- Log
  - Entry
    - 一个条目
  - Segment
    - 一个日志文件
    - 包含多个条目
    - `pub fn open(path: impl Into<PathBuf>) -> Result<Self>`
      - 给定路径指向文件
      - 设置文件后缀为`log`
      - 新建一个读者一个写者
      - 读者可以随意seek
      - 写者只能append
      - 初始写偏移量为文件末尾
    - `pub fn set(&mut self, key: String, value: String) -> Result<Pointer>`
      - 获取当前写偏移量及文件名
      - 生成log pointer
      - 生成log entry
      - 将entry序列化到内存
      - 将序列化结果写入硬盘
      - 写偏移量 += sered entry len
      - 更新hint
    - `pub fn get(&mut self, key: &String) -> Result<Option<String>>`
      - 检查hint中有无键
      - 无则返回
      - 将writer中的buf flush到硬盘
      - 从hint给的offset开始反序列化
    - `pub fn remove(&mut self, key: &String) -> Result<Pointer>`
      - 不管给定键是否存在这个segment中都要写入
        - 可能在前面某个segment
      - 余下逻辑同set
  - Hint
    - 某个Segment的索引
    - 统计信息
      - 某key全部的写操作数个数
    - 成员
      - 路径
      - 偏移量表
      - 修改计数表
    - `pub fn open(path: impl Into<PathBuf>) -> Result<Self>`
      - 给定路径指向文件
      - 设置文件后缀为`hint`
      - 打开文件
      - 背靠背读入两个表
    - `pub fn empty(path: impl Into<PathBuf>) -> Self`
      - 给定路径指向文件
      - 设置文件后缀为`hint`
      - 新建两个空表
    - `pub fn set(&mut self, key: String, offset: u64)`
      - 设置对应键值对的位置
      - 不存在则新建
      - 修改计数+1
    - `pub fn get(&self, key: &String) -> Option<u64>`
      - 获取偏移量表中的值
    - `pub fn remove(&mut self, key: &String)`
      - 删除偏移量表中键值对
      - 修改计数+1
    - `fn drop(&mut self)`
      - 新建/覆盖原hint file
      - 背靠背写入两个表
  - Pointer
    - 日志指针
    - 文件名+偏移量
- KvStore
  - MemTable
    - 全部的内存索引
  - active segment
    - 当前活动日志文件
  - open时重建
    - 打开一个新的next segment
    - 记录读取过的全部文件
    - 便读取边记录统计信息
    - 读取完毕超过阀值时压缩
      - 小文件个数
      - 文件大小
      - 某key写操作数

## 日志压缩

- 扫描所有的segment文件
- flush当前segment的write buffer
- 新建一个KvStore实例
- 依次读取现有KvStore实例MemTable中的key
- 对每个key读取value，写入新KvStore打开的segment中
- segment中k/v pair数超过限制时再分segment
- 最后交换两个KvStore的MemTable
- drop掉新建实例
- 删除原来的文件

