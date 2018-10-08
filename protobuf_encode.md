## protobuf
protobuf是google开源的用于服务之间的通信协议，被应用于各种rpc框架以及服务中。protobuf是平台无关，语言无关，可扩展性强的数据结构序列化协议。跟xml类似，但是protobuf更小，更快，更简单。用户只需要定义protobuf数据结构，然后可以使用protobuf生产当前主流的各种语言。proto3已经支持go,php等，proto2只支持java,c++,python。

## A Simple Message
```
syntax = "proto3";
message Test1 {
  int32 a = 1;
}
```
如果创建Test1的a为150，则经过编码,会变成下面三个字节
>08 96 01
>
那么protobuf是如何进行编码的呢
## Base 128 Varints
varints编码方式使用一个或者多个字节对整形进行序列化。varints的每一个字节分为两部分，最高一位标识着是否为varints的最后一个字节，剩余的低7位标识正常的数组。下面是1的varints编码
>0000 0001

只需要一个字节，最高位为0标识这是varints的最后一个字节，低7位标识的int为1。那么varints如何表示300呢
>1010 1100 0000 0010

上面两个字节第一个字节的最高位为1表示后面还有字节，第二个字节的最高位为0则表示为最后一个字节，我们去掉两个字节的最高位，将有效的低7位合并，后面的低7位为真实int的高位，得到
>1010 1100 0000 0010
>--->0000 010  010 1100
>--->01 0010 1100
>256 + 32 + 8 + 4 = 300

标题`Base 128 Varints`的命名我猜是因为，真实编码int的有效位数为7位，也就是中间的128；这种编码用来对整形进行编码所以是varints。

可以看到对于一些小的数字，varints可以节省很多空间。以int32为例，(这里先不考虑负数，下面会讨论到，varints对负数不友好)一个int为4个字节，对于0-127 varints只需要一个字节，节省了75%的空间，对于128-65536则只需要两个字节。当然由于varints用每一个字节的最高位判断varints的结束，因此对于一些大的整数，varints是不友好的。下面会给出protobuf的其他方案
## Message结构
protobuf的消息编码是以key-value对的方式表示的。key就是我们proto文件中定义的id，外加proto中的wire type。

Type        | Meaning      | Used For
------------------------ | ------------------------ | --------------
0 | Varint | int32, int64, uint32, uint64, sint32, sint64, bool, enum
1 | 64-bit | fixed64, sfixed64, double
2 | Length-delimited | string, bytes, embedded messages, packed repeated fields
3 | Start group | groups (deprecated)
4 | End group | groups (deprecated)
5 | 32-bit | fixed32, sfixed32, float

proto中共有5中wire type。
1. 0就是我们上面介绍的varint编码
2. 1 表示64位的number
3. 2 是一个很重要的wire type 是混用的。表示字符串，字节，嵌入消息，以及packed repeated
4. 3，4以及废弃了
5. 5 表示32为的number

key使用`(field_number << 3) | wire_type`来表示，也就是说低三为用来表示wire type。key也是使用varint进行编码的，因此最高位用作标识是否结束，中间四位表示field_number，因此field_number从1到15只需要一个字节，15之后就要多于一个字节了，我们在写proto结构的时候，应该将经常有值得字段定为15以内的field_number,偶尔有值得字段定义可以大一些的field_number。可以带来性能提升。

这是我们回头看一下最上面小例子的第一个字节`08`，去掉最高一位，剩下7位为`000 1000` 后三位为0表示字段的编码方式为varint，前四位为0001表示field_number为1。因此字段采用varint编码，采用varint解码后面的`96 01`就表示数字150。
## Signed Integers

上面的数字采用varint编码，然后在有符号整数(sint32 and sint64)和标准的整数(int32 and int64)对于负数的编码是有很大不同的。上面说了varint对于负数是很不友好的，如果对于Test1中的a赋值-1,将得到`18 ff ff ff ff ff ff ff ff ff 01`的编码。因此上面的例子中c的类型是标准整形int32,原因是int32会将-1看待成一个很大的unsigned integer。如果使用proto中的signed类型，varint将使用ZigZag编码，会更加的高效。

ZigZag编码会在signed integers与unsigned integers之间做一个映射，以保证数字有一个相对小的值(如-1)。
Signed Original           | Encoded As        
-------------------------- | -------------------------- 
`0` | `0`
`-1` | `1`
`1` | `2`
`-2` | `3`
`2147483647`|`	4294967294`
`-2147483648`|`	4294967295`
对于一个sint32，编码方式为
>(n << 1) ^ (n >> 31)

对于一个sint64,编码方式为
(n << 1) ^ (n >> 63)

## Non-varint Numbers
对于Non-varint编码的数字包括两种，一种为double和fixed64，他们的wire type为1, 表示定长的64位数字，对于float以及fixed32则表示定长的32为数字，他们的wire type为5。
## Strings
字符串比较简单了，采用边长的方式进行编码，例子如下
```
message Test2 {
  optional string b = 2;
}
```
如何这是b的值为`testing`，将得到下面的二禁止编码`12 07 74 65 73 74 69 6e 67`,第一个字节`12`表示wire type为2，field_number为2，第二个字节表示长度，后面的有效长度为7，然后解析7个字符。
## Embedded Messages
```
message Test3 {
  optional Test1 c = 3;
}
```
embedded消息与字符串共用wire type。如果仍然赋值c.a=150，则得到`1a 03 08 96 01`
可以看到第一个字节`1a`表示wire type为2，field_number为3，第二字节表示长度为3，后面三个字节就表示Test1 c的具体内容了，与上面是一样的。
## Packed Repeated Fields
proto2启动对repeated字段进行pack需要对field设置`[packed=true]`，proto3是默认Pack的。这里其实挺好理解，如果不进行pack，对于repeat的字段，每一个内容都需要进行完整的编码，头部的overhead其实是不需要的，因为一个repeat字段的多个元素的overhead应该是一样的，可以进行压缩，设置一个字节长度就可以了。举个例子
```
message Test4 {
  repeated int32 d = 4 [packed=true];
}

对d赋值3, 270, 86942三个数字编码后得到
22        // key (field number 4, wire type 2)
06        // payload size (6 bytes)
03        // first element (varint 3)
8E 02     // second element (varint 270)
9E A7 05  // third element (varint 86942)
```
pack只对简单字段有效，对于字符串或者一个message就没什么作用了。
