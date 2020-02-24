# Project 3: Synchronous client-server networking

## Building Blocks

- `log` crate
  - 只是一个trait
    - 需要附加一个其他logging底层实现crate
  - 用户定义一个logger以及这个类型的全局变量/backend
  - 可以在编译期确定哪些log level会被处理进binary
  - 默认针对嵌入式
    - 不用std
    - 可以开启
- slog crate
  - 是一个生态
  - 不一定只用一个全局后端
  - 可读性好
  - logger
    - 写入日志的入口
  - Drain
    - 日志的管道
    - 树形结构化

### Redis Protocol

- RESP
  - REdis Serializable Protocol
  - 用于C/S通讯
- binary-safe
  - 命令前面加了长度
- 用于面向流的链接
  - TCP
  - Unix sockets
- 请求/应答模型
  - 接受命令
  - 送回响应
  - 特例
    - 支持流水线式请求处理
    - 支持订阅/推送机制
- 实际上是序列化协议
  - 数据类型按首字节区分
  - `+`字符串
    - 串中不能包含CR或者LF
    - `+OK\r\n`
  - `-`错误
    - 实际上就是错误信息/字符串
    - `-Error message\r\n`
      - `-ERR unknown command 'foobar'`
      - `-WRONGTYPE Operation against a key holding the wrong kind of value`
    - 客户端应将其当作异常处理
    - 第一个word是错误类型
      - 只是习惯不是标准
  - `:`整数
    - `:0\r\n`
    - `:1000\r\n`
  - `$`批量字符串
    - 最长512MB
    - `$+长度+CRLF+数据+CRLF`
    - `$6\r\nfoobar\r\n`
    - `$0\r\n\r\n`
    - Null Bulk String
      - `$-1\r\n` 
  - `*`数组
    - `*+长度+CRLF+RESP*长度个`
    - `*3\r\n$3\r\nfoo\r\n$-1\r\n$3\r\nbar\r\n`
      - `["foo", null, "bar"]`
    - `*5\r\n:1\r\n:2\r\n:3\r\n:4\r\n$6\r\nfoobar\r\n`
      - `[1, 2, 3, 4, "foobar"]`
    - `*-1\r\n`
      - `null`
- 命令和HTTP请求一样以CRLF结束
- 交互
  - 客户端发送批量字符串组成的RESP数组
    - `*2\r\n$4\r\nLLEN\r\n$6\r\nmylist\r\n`
    - `LLEN mylist`
  - 服务器随意回应
  - Inline mode
    - 相当于直接接受命令行命令
- 写parser
  - 无lookahead

### 重识serde

- <s>之前一直认为serde就是parser，不同的serializer/deserializer就是不同的写法</s>

- > During serialization, the [`Serialize`](https://docs.serde.rs/serde/trait.Serialize.html) trait maps a Rust data structure into Serde's [data model](https://serde.rs/data-model.html) and the [`Serializer`](https://docs.serde.rs/serde/ser/trait.Serializer.html) trait maps the data model into the output format. During deserialization, the [`Deserializer`](https://docs.serde.rs/serde/trait.Deserializer.html) maps the input data into Serde's data model and the [`Deserialize`](https://docs.serde.rs/serde/trait.Deserialize.html) and [`Visitor`](https://docs.serde.rs/serde/de/trait.Visitor.html) traits map the data model into the resulting data structure.

- serde实际上只规定了一些标准类型
- 这些类型作为所有rust中可以出现的类型和目标表示形式的桥梁
- serde保证rust所有类型都可以用serde中的类型表示/组合出来
- 那么实现serer/der时只需要支持这29种serde类型就相当于支持了rust中所有的类型
- 有关生命周期
  - `<'de, T> where T: Deserialize<'de>`
    - caller决定生命周期
    - 来源数据由caller提供
  - ` where T: DeserializeOwned`
    - callee决定生命周期
    - 返回前用做de的数据就可能要被drop了
    - T拥有de的全部数据



### RESP serde

#### data model mapping

- 14 primitive types
- bool -> 0/1
- i8, i16, i32, i64 -> i64
- u8, u16, u32, u64 -> u64
- f32, f64 -> string
- char -> string

- string/byte array -> simple/bulk string

- option -> array

- unit/unit_struct -> null/"$-1\r\n"

- unit_variant -> string of variant name

- newtype_struct -> value of wrapped type

  - struct Millimeters(u8)

- newtype_variant -> [variant, value]

  - E::N in enum E { N(u8) }

- seq -> array / unknown size would cause error

- tuple/tuple_struct -> array

- map/struct -> [[k0,v0], [k1,v1], ...]

  - struct S { r: u8, g: u8, b: u8 }

- tuple_variant -> [variant, [field0, field1, ...]]

- struct_variant -> [variant, [[k0,v0], [k1,v1], ...]]

#### parser 就不写了

### Perform evaluation

- 不确定性来源
  - JIT
  - 调度
  - GC
  - 中断
- 观测误差
  - 系统误差
    - 应由实验者避免及消除
  - 随机误差
    - 不可预测
    - 建模估计

## Project

- 测试时发现hint的rebuild没写，之前以segment当作最小单元，这个当然不对
- 每次写log都要flush，有待优化
- bench要的时间长的离谱？？？

  

  

  

  

