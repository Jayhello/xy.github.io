---
layout: post
title: "tcp粘包以及固定长度解包方法"
date:   2022-11-22
tags: [网络编程]
comments: true
author: xy
# toc: true  # 这里加目录导致看到地方太小了
---

本文主要内容如下:
1. `tcp` 面向流的传输和 `udp`  面向的是消息传输 区别

2. `tcp` 粘包以及示例(python写的, 多次发送消息服务端一次接受)

3. `tcp` 消息边界的解决方法 `固定长度的消息 LengthCodec`

<!-- more -->

## 1. `tcp` 面向流的传输和 `udp`  面向的是消息（报文）传输 区别

而面向流则是指无保护消息保护边界的，如果发送端连续发送数据，接收端有可能在一次接收动作中，会接收两个或者更多的数据包。 `TCP send`和 `recv` 没有固定的对应关系，不定数目的`send` 可以触发不定数目的 `recv`

例如，我们连续发送三个数据包，大小分别是2k，4k ，8k,这三个数据包，都已经到达了接收端的网络堆栈中，如果使用UDP协议，不管我们使用多大的接收缓冲区去接收数据，我们必须有三次接收动作，才能够把所有的数据包接收完. 但是`TCP`可能一次或者多次才将数据全部接收完。

那么`TCP`为什么是流式呢?

1. `TCP` 为了保证可靠传输, 拥塞控制，尽量减少额外开销（每次发包都要验证），因此采用了流式传输
2. 对接收端的程序来讲，如果机器负荷很重，也会在接收缓冲里粘包


## 2. `tcp` 粘包以及示例

下面给出一个`python`的示例, 客户端发送5次数据, 服务的一次接受。服务段代码如下:
```
def test_simple():
    server_sock = socket.socket()
    server_sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_sock.bind((HOST, PORT))
    server_sock.listen(0)
    print "Listening on %s:%s..." % (HOST, PORT)

    while 1:
        client_sock, client_addr = server_sock.accept()
        print "New connection from %s:%s." % (client_addr)

        time.sleep(10)

        data = client_sock.recv(1024)
        print "recv :%s" % data

        n = client_sock.send(RESPONSE)
        print "just send %s bytes" % n
```

客户端代码如下：在server sleep 10S内，发送了5次：
```
def basic_connect_rcv():

    s = socket.socket()
    s.connect(('127.0.0.1', 8888))
    print("We are connected to %s:%d" % s.getpeername())

    buf = '123456789 -> '

    for i in xrange(5):
        n = s.send(buf)
        print "now send %s" % n
        time.sleep(2)
```

客户端输出如下：
```
We are connected to 127.0.0.1:8888
now send 13
now send 13
now send 13
now send 13
now send 13
```

服务端输出如下:

```
Listening on 127.0.0.1:8888...
New connection from 127.0.0.1:13046.
recv :123456789 -> 123456789 -> 123456789 -> 123456789 -> 123456789 ->
just send 1024 bytes
```
可以看到服务端处理不及时的情况下, 客户端的包`粘包`了。


## 3. `tcp` 消息边界的解决方法 `固定长度的消息 LengthCodec`

TCP无保护消息边界的解决, 针对这个问题，一般有3种解决方案：

1. `定长包`: 发送固定长度的消息          
2. `包尾添加特殊分隔符`: 使用特殊标记来区分消息间隔
3. `结构体包  包头包含包的大小` 把消息的尺寸与消息一块发送

`Netty` 粘包和拆包解决方案, 可以进行分包的操作，分别是： 

* LineBasedFrameDecoder 
* DelimiterBasedFrameDecoder（添加特殊分隔符报文来分包） 
* FixedLengthFrameDecoder（使用定长的报文来分包） 
* LengthFieldBasedFrameDecoder

下面给出 `LengthCodec` 的方式, 代码做了很多简化, 也不考虑效率, 便于理解

`Connection` 在读事件的时候尝试解包, 如果解包成功就将解出的`Msg`给`msg`做`handler`

```
void ConnectionBase::onRead(){
    int fd = ptrChannel_->getFd();
    while(true) {  //由于使用非阻塞IO，读取客户端buffer，一次读取buf大小数据，直到全部读取完毕
        string sData;
        int iReadSize = raw_v1::doRead(fd, sData, 1024);
        if (iReadSize > 0) {
            rBuf_.append(sData);
            Msg msg;
            int tmp = codec_.tryDecode(rBuf_, msg);
            info("get msg from fd: %d, size: %d, %s, decode_ret: %d", fd, iReadSize, sData.c_str(), tmp);
            if(tmp > 0){
                onMsg(msg);             // 完整的消息
                rBuf_.remove(0, tmp);   // 将buffer中解的去掉
            }
        } else if (0 == iReadSize) {  //EOF，客户端断开连接
            raw_v1::doClose(fd);
            break;
        } else {  // -1
            int error = errno;
            if (EAGAIN == error or EWOULDBLOCK == error) {  //非阻塞IO，这个条件表示数据全部读取完毕
                info("fd: %d no data...., ret: %d error: %d, %s", fd, iReadSize, error, strerror(error));
                break;
            } else if (EINTR == error) {  //客户端正常中断、继续读取
                info("fd: %d continue reading...., ret: %d error: %d, %s", fd, iReadSize, error,
                     strerror(error));
                continue;
            } else {
                ....
                raw_v1::doClose(fd);
                break;
            }
        }
    }
}
```

`msg handler` 简单的打印并回包
```
void ConnectionBase::onMsg(const Msg& msg){
    info("ep: %s, on_msg: %s", ePoint_.toString().c_str(), msg.c_str());

    int iWriteSize = raw_v1::doWrite(ePoint_.fd, msg.data());  // < 0 简洁点去掉
    info("echo back size: %d", iWriteSize);
}
```

发送接收的`Buffer`定义如下(做了大量简化):
```
class Buffer{
public:
    Buffer() = default;
    Buffer(const string& str):data_(str){}

    int size()const{return data_.size();}
    const string& data()const{return data_;}
    string& data(){return data_;}

    void reserve(int size){data_.reserve(size);}
    void resize(int size){data_.resize(size);}

    void append(const char* str, int len){
        data_.append(str, len);
    }
    void append(const string& str){data_.append(str);}
    const char* c_str()const{return data_.c_str();}
    void remove(int pos, int len){data_.erase(pos, len);}
private:
    string data_;
};
```

消息`Msg`定义如下(做了大量简化):

```
class Msg{
public:
    Msg() = default;
    Msg(const string& str):data_(str){}
    const string& data()const{return data_;}
    string& data(){return data_;}
    int size()const{return data_.size();}
    void append(const string& str){data_.append(str);}
    void append(const char* str, int len){
        data_.append(str, len);
    }

    const char* c_str()const{return data_.c_str();}
private:
    string data_;
};
```

编解码如下(做了大量简化), 后期可以支持 `\n\r`:
```
const uint32_t MAGIC = 0x1234;

class CodecBase{
public:
    virtual  ~CodecBase(){};
    virtual void encode(const Msg& msg, Buffer& buf) = 0;
    // < 0 buf数据异常, = 0 数据不完整, > 0 解析出了一个多大的msg包
    virtual int tryDecode(const Buffer& buf, Msg& msg) = 0;
};

class LengthCodec: public CodecBase{
public:
    virtual void encode(const Msg& msg, Buffer& buf){
        buf.append((char*)(&MAGIC), sizeof(MAGIC));  // head magic

        int len = msg.size();
        buf.append((char*)(&len), sizeof(len));      // length
        buf.append(msg.data());                      // data        
    }

    // < 0 buf数据异常, = 0 数据不完整, > 0 解析出了一个多大的msg包
    virtual int tryDecode(const Buffer& buf, Msg& msg){
        if(buf.size() < 8){
            return 0;
        }

        if(0 != memcmp(buf.c_str(), (char*)(&MAGIC), sizeof(MAGIC))){
            return -1;
        }

        int len = *((int*)(buf.c_str() + 4));
        if(len > 10 * 1024 * 124)return -2;

        if(buf.size() >= len + 8){
            msg.append(buf.c_str() + 8, len);
            return len + 8;
        }

        return 0;        
    }
};

```

编解码示例代码如下:
```
void example_codec_1(){
    log_v1::ScopeLog Log;

    raw_comm::Msg    msg1("123456");
    raw_comm::Buffer bf1;

    raw_comm::LengthCodec cc;
    cc.encode(msg1, bf1);
    Log << "bf1_size: " << bf1.size();

    raw_comm::Msg msg2;
    int ret = cc.tryDecode(bf1, msg2);
    Log << ", decode_ret: " << ret << ", decode_msg: " << msg2.data();
}
```

运行输出如下:
![net](https://raw.githubusercontent.com/Jayhello/Jayhello.github.io/master/images/net/pack_1.png)

实验的`client`端代码如下, `encode`之后多次发送:

```
// 使用 msg encode之后再发送
void example_msg_codec(){
    log_v1::ScopeLog Log;

    int fd = raw_v1::getTcpSocket();
    return_if(fd <= 0, "get_socket_fd_fail");
    Log << "fd: " << fd;

    int ret = raw_v1::doConnect(fd, LOCAL_IP, PORT);
    //    return_if(ret < 0, "connect_fail ret: %d, %s", ret, strerror(ret));
    return_if(ret < 0, "connect_fail ret: %d, %s", ret, strerror(errno));
    Log << " connect_succ";

    raw_comm::Msg msg("123456");
    raw_comm::Buffer buf;

    raw_comm::LengthCodec cc;
    cc.encode(msg, buf);
    Log << ", bf1_size: " << buf.size();

    while(buf.size()){
        int len = std::min(3, buf.size());
        Log << ", send_len: " << len;
        string tmp = buf.data().substr(0, len);

        ret = raw_v1::doSend(fd, tmp);
        Log <<" ret: " << ret;
        if(ret > 0){
            buf.remove(0, len);
            Log << " remove_left: " << buf.size();
        }

        std::this_thread::sleep_for(std::chrono::seconds(1));   // 模拟拆包(一个包多次发完)
    }

    string sData;
    int iReadSize = raw_v1::doRecv(fd, sData, 1024);
    Log << ", read_size: " << iReadSize << " msg: " << sData;

    raw_v1::doClose(fd);
    Log << " close_exit";
}
```

输出如下：
```
INFO logging.h:129 fd:  3 connect_succ , bf1_size:  14, send_len:  3 ret:  3 remove_left:  11, send_len:  3 ret:  3 remove_left:  8, send_len:  3 ret:  3 remove_left:  5, send_len:  3 ret:  3 remove_left:  2, send_len:  2 ret:  2 remove_left:  0, read_size:  6 msg: 123456 close_exit
```

服务端输出如下:

![net](https://raw.githubusercontent.com/Jayhello/Jayhello.github.io/master/images/net/pack_2.png)


