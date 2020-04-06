# 16 | 如何理解TCP的“流”？

 

上一讲我们讲到了使用 SO_REUSEADDR 套接字选项，可以让服务器满足快速重启的需求。在这一讲里，我们回到数据的收发这个主题，谈一谈如何理解 TCP 的数据流特性。

### TCP 是一种流式协议

在前面的章节中，我们讲的都是单个客户端 - 服务器的例子，可能会给你造成一种错觉，好像 TCP 是一种应答形式的数据传输过程，比如发送端一次发送 network 和 program 这样的报文，在前面的例子中，我们看到的结果基本是这样的：

发送端：network ----> 接收端回应：Hi, network

发送端：program -----> 接收端回应：Hi, program

这其实是一**个假象，**之所以会这样，是因为网络条件比较好，而且发送的数据也比较少。

为了让大家理解 TCP 数据是流式的这个特性，我们分别从发送端和接收端来阐述。

我们知道，在发送端，当我们**调用 send 函数完成数据“发送”以后，数据并没有被真正从网络上发送出去，只是从应用程序拷贝到了操作系统内核协议栈中，至于什么时候真正被发送，取决于发送窗口、拥塞窗口以及当前发送缓冲区的大小等条件**。也就是说，我们不能假设每次 send 调用发送的数据，都会作为一个整体完整地被发送出去。

如果我们考虑实际网络传输过程中的各种影响，假设发送端陆续调用 send 函数先后发送 network 和 program 报文，那么实际的发送很有可能是这个样子的。

第一种情况，一次性将 network 和 program 在一个 TCP 分组中发送出去，像这样：

~~~C
...xxxnetworkprogramxxx...
~~~

第二种情况，program 的部分随 network 在一个 TCP 分组中发送出去，像这样：

TCP 分组 1：

~~~LINUX

...xxxxxnetworkpro
~~~

TCP 分组 2：

~~~linux

gramxxxxxxxxxx...
~~~

第三种情况，network 的一部分随 TCP 分组被发送出去，另一部分和 program 一起随另一个 TCP 分组发送出去，像这样。

TCP 分组 1： 	

~~~linux

...xxxxxxxxxxxnet
~~~

TCP 分组 2：

~~~linux

workprogramxxx...
~~~

实际上类似的组合可以枚举出无数种。不管是哪一种，核心的问题就是，我们不知道 network 和 program 这两个报文是如何进行 TCP 分组传输的。换言之，我们在发送数据的时候，不应该假设“数据流和 TCP 分组是一种映射关系”。就好像在前面，我们似乎觉得 network 这个报文一定对应一个 TCP 分组，这是完全不正确的。

如果我们再来看客户端，数据流的特征更明显。

我们知道，接收端缓冲区保留了没有被取走的数据，随着应用程序不断从接收端缓冲区读出数据，接收端缓冲区就可以容纳更多新的数据。如果我们使用 recv 从接收端缓冲区读取数据，发送端缓冲区的数据是以字节流的方式存在的，无论**发送端如何构造 TCP 分组，接收端最终受到的字节流总是像下面这**样：

~~~linux

xxxxxxxxxxxxxxxxxnetworkprogramxxxxxxxxxxxx
~~~

关于接收端字节流，有两点需要注意：

**第一，这里 netwrok 和 program 的顺序肯定是会保持的，也就是说，先调用 send 函数发送的字节，总在后调用 send 函数发送字节的前面，这个是由 TCP 严格保证的；**

**第二，如果发送过程中有 TCP 分组丢失，但是其后续分组陆续到达，那么 TCP 协议栈会缓存后续分组，直到前面丢失的分组到达，最终，形成可以被应用程序读取的数据流。**



### 网络字节排序

​		我们知道计算机最终保存和传输，用的都是 0101 这样的二进制数据，字节流在网络上的传输，也是通过二进制来完成的。从**二进制到字节是通过编码完成的，比如著名的 ASCII 编码，通过一个字节 8 个比特对常用的西方字母进行了编码。**

这里有一个有趣的问题，如果需要传输数字，比如 0x0201，对应的二进制为 00000010000000001，那么两个字节的数据到底是先传 0x01，还是相反？

![](image\字节序.png)

​		在计算机发展的历史上，对于如何存储这个数据没有形成标准。比如这里讲到的问题，不同的系统就会有两种存法，一种是将 0x02 高字节存放在起始地址，这个叫做大端字节序（Big-Endian）。另一种相反，将 0x01 低字节存放在起始地址，这个叫做小端字节序（Little-Endian）。但是在网络传输中，必须保证双方都用同一种标准来表达，这就好比我们打电话时说的是同一种语言，否则双方不能顺畅地沟通。这个标准就涉及到了网络字节序的选择问题，对于网络字节序，必须二选一。我们可以看到网络协议使用的是大端字节序，我个人觉得大端字节序比较符合人类的思维习惯，你可以想象手写一个多位数字，从开始往小位写，自然会先写大位，比如写 12, 1234，这个样子。

​		为了保证网络字节序一致，POSIX 标准提供了如下的转换函数：

~~~linux

uint16_t htons (uint16_t hostshort)
uint16_t ntohs (uint16_t netshort)
uint32_t htonl (uint32_t hostlong)
uint32_t ntohl (uint32_t netlong)
~~~

这里函数中的 n 代表的就是 network，h 代表的是 host，s 表示的是 short，l 表示的是 long，分别表示 16 位和 32 位的整数。这些函数可以帮助我们在主机（host）和网络（network）的格式间灵活转换。当使用这些函数时，我们并不需要关心主机到底是什么样的字节顺序，只要使用函数给定值进行网络字节序和主机字节序的转换就可以了。



你可以想象，如果碰巧我们的系统本身是大端字节序，和网络字节序一样，那么使用上述所有的函数进行转换的时候，结果都仅仅是一个空实现，直接返回。

如这样：

~~~Linux

# if __BYTE_ORDER == __BIG_ENDIAN
/* The host byte order is the same as network byte order,
   so these functions are all just identity.  */
# define ntohl(x) (x)
# define ntohs(x) (x)
# define htonl(x) (x)
# define htons(x) (x)
~~~



### 报文读取和解析

​		应该看到，报文是以字节流的形式呈现给应用程序的，那么随之而来的一个问题就是，应用程序如何解读字节流呢？这就要说到报文格式和解析了。**报文格式实际上定义了字节的组织形式，发送端和接收端都按照统一的报文格式进行数据传输和解析，这样就可以保证彼此能够完成交流。**

​		只有知道了报文格式，接收端才能针对性地进行报文读取和解析工作。报文格式最重要的是如何确定报文的边界。常见的报文格式有两种方法，**一种是发送端把要发送的报文长度预先通过报文告知给接收端；另一种是通过一些特殊的字符来进行边界的划分**。

### 显式编码报文长度

#### 报文格式

下面我们来看一个例子，这个例子是把要发送的报文长度预先通过报文告知接收端，你可以看到文章中有这样一张图。

![](image\报文格式1.png)

​		由图可以看出，这个报文的格式很简单，**首先 4 个字节大小的消息长度，其目的是将真正发送的字节流的大小显式通过报文告知接收端，接下来是 4 个字节大小的消息类型，而真正需要发送的数据则紧随其后**。

#### 发送报文

发送端的程序如下：

~~~linux

int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, "usage: tcpclient <IPaddress>");
    }

    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &server_addr.sin_addr);

    socklen_t server_len = sizeof(server_addr);
    int connect_rt = connect(socket_fd, (struct sockaddr *) &server_addr, server_len);
    if (connect_rt < 0) {
        error(1, errno, "connect failed ");
    }

    struct {
        u_int32_t message_length;
        u_int32_t message_type;
        char buf[128];
    } message;

    int n;

    while (fgets(message.buf, sizeof(message.buf), stdin) != NULL) {
        n = strlen(message.buf);
        message.message_length = htonl(n);
        message.message_type = 1;
        if (send(socket_fd, (char *) &message, sizeof(message.message_length) + sizeof(message.message_type) + n, 0) <
            0)
            error(1, errno, "send failure");

    }
    exit(0);
}
~~~

程序的 

1-20 行是常规的创建套接字和地址，建立连接的过程。我们重点往下看，

21-25 行就是图示的报文格式转化为结构体，

29-37 行从标准输入读入数据，分别对消息长度、类型进行了初始化，**注意这里使用了 htonl 函数将字节大小转化为了网络字节顺序，这一点很重要**。

最后我们看到 23 行实际发送的字节流大小为消息长度 4 字节，加上消息类型 4 字节，以及标准输入的字符串大小。

#### 解析报文：程序

下面给出的是服务器端的程序，和客户端不一样的是，服务器端需要对报文进行解析。

~~~linux

static int count;

static void sig_int(int signo) {
    printf("\nreceived %d datagrams\n", count);
    exit(0);
}


int main(int argc, char **argv) {
    int listenfd;
    listenfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(SERV_PORT);

    int on = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));

    int rt1 = bind(listenfd, (struct sockaddr *) &server_addr, sizeof(server_addr));
    if (rt1 < 0) {
        error(1, errno, "bind failed ");
    }

    int rt2 = listen(listenfd, LISTENQ);
    if (rt2 < 0) {
        error(1, errno, "listen failed ");
    }

    signal(SIGPIPE, SIG_IGN);

    int connfd;
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    if ((connfd = accept(listenfd, (struct sockaddr *) &client_addr, &client_len)) < 0) {
        error(1, errno, "bind failed ");
    }

    char buf[128];
    count = 0;

    while (1) {
        int n = read_message(connfd, buf, sizeof(buf));
        if (n < 0) {
            error(1, errno, "error read message");
        } else if (n == 0) {
            error(1, 0, "client closed \n");
        }
        buf[n] = 0;
        printf("received %d bytes: %s\n", n, buf);
        count++;
    }

    exit(0);

}
~~~

这个程序 

1-41 行创建套接字，等待连接建立部分和前面基本一致。

我们重点看 42-55 行的部分。45-55 行循环处理字节流，调用 read_message 函数进行报文解析工作，并把报文的主体通过标准输出打印出来。

#### 解析报文：readn 函数

在了解 read_message 工作原理之前，我们先来看第 5 讲就引入的一个函数：readn。这里一定要强调的是 readn 函数的语义，读取报文预设大小的字节，readn 调用会一直循环，尝试读取预设大小的字节，如果接收缓冲区数据空，readn 函数会阻塞在那里，直到有数据到达。

~~~linux

size_t readn(int fd, void *buffer, size_t length) {
    size_t count;
    ssize_t nread;
    char *ptr;

    ptr = buffer;
    count = length;
    while (count > 0) {
        nread = read(fd, ptr, count);

        if (nread < 0) {
            if (errno == EINTR)
                continue;
            else
                return (-1);
        } else if (nread == 0)
            break;                /* EOF */

        count -= nread;
        ptr += nread;
    }
    return (length - count);        /* return >= 0 */
}
~~~

readn 函数中使用 count 来表示还需要读取的字符数，如果 count 一直大于 0，说明还没有满足预设的字符大小，循环就会继续。

第 9 行通过 read 函数来服务最多 count 个字符。

11-17 行针对返回值进行出错判断，其中**返回值为 0 的情形是 EOF**，表示对方连接终止。

19-20 行要读取的字符数减去这次读到的字符数，同时移动缓冲区指针，这样做的目的是为了确认字符数是否已经读取完毕。

#### 解析报文: read_message 函数

有了 readn 函数作为基础，我们再看一下 read_message 对报文的解析处理：

~~~linux

size_t read_message(int fd, char *buffer, size_t length) {
    u_int32_t msg_length;
    u_int32_t msg_type;
    int rc;

    rc = readn(fd, (char *) &msg_length, sizeof(u_int32_t));
    if (rc != sizeof(u_int32_t))
        return rc < 0 ? -1 : 0;
    msg_length = ntohl(msg_length);

    rc = readn(fd, (char *) &msg_type, sizeof(msg_type));
    if (rc != sizeof(u_int32_t))
        return rc < 0 ? -1 : 0;

    if (msg_length > length) {
        return -1;
    }

    rc = readn(fd, buffer, msg_length);
    if (rc != msg_length)
        return rc < 0 ? -1 : 0;
    return rc;
}
~~~

在这个函数中，第 6 行通过调用 readn 函数获取 4 个字节的消息长度数据，紧接着，第 11 行通过调用 readn 函数获取 4 个字节的消息类型数据。第 15 行判断消息的长度是不是太大，如果大到本地缓冲区不能容纳，则直接返回错误；第 19 行调用 readn 一次性读取已知长度的消息体。



### 实验

我们依次启动作为报文解析的服务器一端，以及作为报文发送的客户端。我们看到，每次客户端发送的报文都可以被服务器端解析出来，在标准输出上的结果验证了这一点。

~~~linux

$./streamserver
received 8 bytes: network
received 5 bytes: good
~~~

~~~linux

$./streamclient
network
good
~~~





### 特殊字符作为边界

前面我提到了两种报文格式，另外一种报文格式就是通过设置特殊字符作为报文边界。HTTP 是一个非常好的例子。

![](image\报文格式2.png)

HTTP 通过**设置回车符、换行符做为 HTTP 报文协议的边界**。

下面的 read_line 函数就是在尝试读取一行数据，也就是读到回车符\r，或者读到回车换行符\r\n为止。这个函数每次尝试读取一个字节，第 9 行如果读到了回车符\r，接下来在 11 行的“观察”下看有没有换行符，如果有就在第 12 行读取这个换行符；如果没有读到回车符，就在第 16-17 行将字符放到缓冲区，并移动指针。

~~~linux

int read_line(int fd, char *buf, int size) {
    int i = 0;
    char c = '\0';
    int n;

    while ((i < size - 1) && (c != '\n')) {
        n = recv(fd, &c, 1, 0);
        if (n > 0) {
            if (c == '\r') {
                n = recv(fd, &c, 1, MSG_PEEK);
                if ((n > 0) && (c == '\n'))
                    recv(fd, &c, 1, 0);
                else
                    c = '\n';
            }
            buf[i] = c;
            i++;
        } else
            c = '\n';
    }
    buf[i] = '\0';

    return (i);
}
~~~





### 总结

和我们预想的不太一样，**TCP 数据流特性决定了字节流本身是没有边界的，一般我们通过显式编码报文长度的方式，以及选取特殊字符区分报文边界的方式来进行报文格式的设计。而对报文解析的工作就是要在知道报文格式的情况下，有效地对报文信息进行还原**。



### 思考题



1. 第一道题关于 HTTP 的报文格式，我们看到，既要处理只有回车的情景，也要处理同时有回车和换行的情景，你知道造成这种情况的原因是什么吗？
2. 第二道题是，我们这里讲到的**报文格式，和 TCP 分组的报文格式**，有什么区别和联系吗？