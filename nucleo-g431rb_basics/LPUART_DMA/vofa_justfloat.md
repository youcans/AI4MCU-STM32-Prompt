# VOFA+ JustFloat协议说明

本文档依据VOFA+官方JustFloat协议说明整理，仅用于本项目的波形数据发送。

## 数据帧格式

JustFloat采用小端浮点数组形式的字节流协议。数据帧由浮点型通道数据和4字节帧尾组成。

```c
#define CH_COUNT  1

struct Frame
{
    float ch_data[CH_COUNT];
    unsigned char tail[4];
};
```

其中：

- `ch_data`为需要发送的浮点型通道数据，采用小端字节序；
- `CH_COUNT`表示每帧包含的通道数量；
- `tail`为固定帧尾，内容为：

```text
0x00 0x00 0x80 0x7F
```

## 本实验数据格式

本实验每帧发送一个锯齿波数据，因此`CH_COUNT`设置为1。每帧共包含8字节：

```text
4字节float数据 + 4字节固定帧尾
```

数据排列如下：

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

其中，前4字节是锯齿波浮点数据在STM32中的小端字节表示。

## 本实验约束

- 仅实现JustFloat采样数据帧发送；
- 不实现图片数据传输；
- 不实现普通文本打印；
- 串口数据流中不添加换行符；
- 每个数据帧必须包含完整的4字节帧尾；
- VOFA+连接串口后，数据引擎选择`JustFloat`。