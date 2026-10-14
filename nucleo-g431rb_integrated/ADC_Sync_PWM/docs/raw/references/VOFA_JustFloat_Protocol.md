# VOFA+ JustFloat协议说明

本文档依据VOFA+官方JustFloat协议说明整理，用于了解波形数据的组织和发送方式。

## 数据帧格式

JustFloat采用小端浮点数组形式的字节流协议。数据帧由一个或多个浮点型通道数据和4字节固定帧尾组成。

```c
#define CH_COUNT  N

struct Frame
{
    float ch_data[CH_COUNT];
    unsigned char tail[4];
};
```

其中：

* `ch_data`为需要发送的浮点型通道数据，采用小端字节序；
* `CH_COUNT`表示每帧包含的通道数量；
* `tail`为固定帧尾，内容如下：

```text
0x00 0x00 0x80 0x7F
```

每个浮点型通道数据占4字节，因此数据帧长度为：

```text
4 × CH_COUNT + 4字节
```

## 单通道数据帧示例

当`CH_COUNT`为1时，每帧包含一个浮点型数据和4字节固定帧尾，共8字节。

```text
wave_data[0]
wave_data[1]
wave_data[2]
wave_data[3]
0x00
0x00
0x80
0x7F
```

其中，前4字节是浮点型数据的小端字节表示，后4字节是JustFloat固定帧尾。

使用VOFA+接收数据时，需要在相应通信接口中选择`JustFloat`数据引擎。
