## 前言

作为一名开发人员我们经常会听到`HTTP协议、TCP/IP协议、UDP协议、Socket、Socket长连接、Socket连接池`等字眼，然而它们之间的关系、区别及原理并不是所有人都能理解清楚，这篇文章就从网络协议基础开始到 Socket 连接池，一步一步解释他们之间的关系。

## 七层网络模型

首先从网络通信的分层模型讲起：七层模型，亦称`OSI(Open System Interconnection)`模型。自下往上分为：物理层、数据链路层、网络层、传输层、会话层、表示层和应用层。所有有关通信的都离不开它，下面这张图片介绍了各层所对应的一些协议和硬件

![[Pasted image 20230225171450.png]]

通过上图，我知道IP协议对应于网络层，TCP、UDP协议对应于传输层，而HTTP协议对应于应用层，OSI并没有Socket，那什么是Socket，后面我们将结合代码具体详细介绍。

## TCP和UDP连接

关于传输层TCP、UDP协议可能我们平时遇见的会比较多，有人说TCP是安全的，UDP是不安全的，UDP传输比TCP快，那为什么呢，我们先从TCP的连接建立的过程开始分析，然后解释UDP和TCP的区别。

#### TCP的三次握手和四次分手

我们知道TCP建立连接需要经过三次握手，而断开连接需要经过四次分手，那三次握手和四次分手分别做了什么和如何进行的。

![](https://filescdn.proginn.com/e76b6896e5f3f5dda90ea2fcae6ece80/faff686aaea680e5b19a7d9f27a94c47.webp)

第一次握手：建立连接。客户端发送连接请求报文段，将SYN位置为1，Sequence Number为x；然后，客户端进入`SYN_SEND`状态，等待服务器的确认；

第二次握手：服务器收到客户端的SYN报文段，需要对这个SYN报文段进行确认，设置Acknowledgment Number为x+1(Sequence Number+1)；同时，自己自己还要发送SYN请求信息，将SYN位置为1，Sequence Number为y；服务器端将上述所有信息放到一个报文段（即SYN+ACK报文段）中，一并发送给客户端，此时服务器进入`SYN_RECV`状态；

第三次握手：客户端收到服务器的`SYN+ACK`报文段。然后将Acknowledgment Number设置为y+1，向服务器发送ACK报文段，这个报文段发送完毕以后，客户端和服务器端都进入`ESTABLISHED`状态，完成TCP三次握手。

完成了三次握手，客户端和服务器端就可以开始传送数据。以上就是TCP三次握手的总体介绍。通信结束客户端和服务端就断开连接，需要经过四次分手确认。

第一次分手：主机1（可以使客户端，也可以是服务器端），设置Sequence Number和Acknowledgment Number，向主机2发送一个FIN报文段；此时，主机1进入`FIN_WAIT_1`状态；这表示主机1没有数据要发送给主机2了；

第二次分手：主机2收到了主机1发送的FIN报文段，向主机1回一个ACK报文段，Acknowledgment Number为Sequence Number加1；主机1进入`FIN_WAIT_2`状态；主机2告诉主机1，我“同意”你的关闭请求；

第三次分手：主机2向主机1发送FIN报文段，请求关闭连接，同时主机2进入`LAST_ACK`状态；

第四次分手：主机1收到主机2发送的FIN报文段，向主机2发送ACK报文段，然后主机1进入`TIME_WAIT`状态；主机2收到主机1的ACK报文段以后，就关闭连接；此时，主机1等待2MSL后依然没有收到回复，则证明Server端已正常关闭，那好，主机1也可以关闭连接了。

可以看到一次tcp请求的建立及关闭至少进行7次通信，这还不包过数据的通信，而UDP不需3次握手和4次分手。

#### TCP和UDP的区别  

1、TCP是面向链接的，虽然说网络的不安全不稳定特性决定了多少次握手都不能保证连接的可靠性，但TCP的三次握手在最低限度上(实际上也很大程度上保证了)保证了连接的可靠性;而UDP不是面向连接的，UDP传送数据前并不与对方建立连接，对接收到的数据也不发送确认信号，发送端不知道数据是否会正确接收，当然也不用重发，所以说UDP是无连接的、不可靠的一种数据传输协议。　 　

2、也正由于1所说的特点，使得UDP的开销更小数据传输速率更高，因为不必进行收发数据的确认，所以UDP的实时性更好。知道了TCP和UDP的区别，就不难理解为何采用TCP传输协议的MSN比采用UDP的QQ传输文件慢了，但并不能说QQ的通信是不安全的，因为程序员可以手动对UDP的数据收发进行验证，比如发送方对每个数据包进行编号然后由接收方进行验证啊什么的，即使是这样，UDP因为在底层协议的封装上没有采用类似TCP的“三次握手”而实现了TCP所无法达到的传输效率。

#### TCP的核心概念

在HTTP的规范内，两台计算机的交互被视为request和response的传递。而在实际的TCP操作中，信息传递会比单纯的传递request和response要复杂。通过TCP建立的通讯往往需要计算机之间多次的交换信息才能完成一次request或response。

TCP的传输数据的核心是在于将数据分为若干段并将每段数据按顺序标记。标记后的顺序可以以不同的顺序被另一方接收并集成回完整的数据。计算机对每一段数据的成功接收都会做出相应，确保所有数据的完整性。

TCP在传递数据时依赖于实现定义好的几个标记（Flags）去向另一方表态传达数据和连接的状态：

* F : FIN - 结束; 结束会话
* S : SYN - 同步; 表示开始会话请求
* R : RST - 复位;中断一个连接
* P : PUSH - 推送; 数据包立即发送
* A : ACK - 应答
* U : URG - 紧急
* E : ECE - 显式拥塞提醒回应
* W : CWR - 拥塞窗口减少

也是基于这些标志TCP可以实现三次（three ways handshake）和四次握手 (four ways tear down)。三次握手是初步建立连接的机制，而四次握手则是断开链接。两者之间大致操作是一样的，A发出建立链接(SYN)或者断开链接(FIN)的请求，B认可(ACK)其请求然后发出同样的请求给A并等待A的认可。在双方认可后，链接正式成立或者断开。

这里有两个问题：

1. 为什么A发出请求并且得到认可后B还有重复同样的动作？

在建立连接的过程中SYN标记代表的是一个随机序列号，因为当文件被切断的时候并不是从0或者1开始标记每段的顺序，所以双方都需要通过传递SYN来告知文件片段的第一个序列是多少号。

2. 为什么同样的机制，建立链接和断开链接需要握手的次数不同？

三次和四次握手的区别在于，在建立连接时，B的ACK和SYN会一起发送回A，而在断开链接时因为B发送ACK之后还要做其他处理后才能返回FIN，，因此将两步拆开。

## HTTP协议

关于TCP/IP和HTTP协议的关系，网络有一段比较容易理解的介绍：“我们在传输数据时，可以只使用(传输层)TCP/IP协议，但是那样的话，如果没有应用层，便无法识别数据内容。如果想要使传输的数据有意义，则必须使用到应用层协议。应用层协议有很多，比如HTTP、FTP、TELNET等，也可以自己定义应用层协议。

HTTP协议即超文本传送协议(Hypertext Transfer Protocol )，是Web联网的基础，也是手机联网常用的协议之一，WEB使用HTTP协议作应用层协议，以封装HTTP文本信息，然后使用TCP/IP做传输层协议将它发到网络上。

由于HTTP在每次请求结束后都会主动释放连接，因此HTTP连接是一种“短连接”，要保持客户端程序的在线状态，需要不断地向服务器发起连接请求。通常 的做法是即时不需要获得任何数据，客户端也保持每隔一段固定的时间向服务器发送一次“保持连接”的请求，服务器在收到该请求后对客户端进行回复，表明知道 客户端“在线”。若服务器长时间无法收到客户端的请求，则认为客户端“下线”，若客户端长时间无法收到服务器的回复，则认为网络已经断开。

下面是一个简单的HTTP Post application/json数据内容的请求：

```HTTP
POST  HTTP/1.1
Host: 127.0.0.1:9017
Content-Type: application/json
Cache-Control: no-cache

{"a":"a"}   
```

#### HTTP的核心概念

除了HTTP存在于应用层之外，该协议还有5个特点。

1. HTTP的标准建立在将两台计算机视为不同的角色：客户端和服务器。客户端会向服务器传送不同的请求(request)，而服务器会对应每个请求给出回应(response)。

2. HTTP属于无状态协议(Stateless)。这表示每一个请求之间是没有相关性的。在该协议的规则中服务器是不会记录任何客户端操作，每一次请求都是独立的。（记录用户浏览行为会通过其他技术实现）

3. 客户端的请求被定义在几个动词意义范围内。最长用到的是GET和POST，其他动词还包括DELETE, HEAD等等。

4. 服务器的回应被定义在几个状态码之间：5开头表示服务器错误，4开头表示客户端错误，3开头表示需要做进一步处理，2开头表示成功，1开头表示在请求被接受处理的同时提供的额外信息。

5. 不管是客户端的请求信息还是服务器的回应，双方都拥有一块头部信息(Header)。头部信息是自定义，其用途在于传递额外信息（浏览器信息、请求的内容类型、相应的语言）。

## HTTP和TCP之间的关系

![[Pasted image 20230225172733.png]]

互联网的模型被分为4层，从上至下每一层都依赖其底层协议。换言之，Application(应用层) 的协议操作成功的前提是Transport(运输层)的存在。没有运输层就没有应用层。好比没有任何道路的前提下就没有汽车可以行驶。而这种层次上的抽象是让开发者在设定某个层面的协议时不去考虑其他层面的问题。比如我要在运输层设计协议时，我唯一要考虑的是如何将数据从一台计算机传到另外一台，我需要着重的是其稳定性和效率。在解决运输层的问题时我不需要考虑传达的数据是什么类型或内容，因为这样的问题是应用层索要操心的。在上图中可以看到HTTP和TCP是存在于不同层面的网络协议，所以他们之间必然存在着依赖关系。确切的说是HTTP所设定的所有规则都建立在一个假设之上，那就是运输层的协议有在正常运作。

#### 那HTTP和TCP分别代表了什么呢？

HTTP的责任是去定义数据，在两台计算机相互传递信息时，HTTP规定了每段数据以什么形式表达才是能够被另外一台计算机理解。而TCP所要规定的是数据应该怎么传输才能稳定且高效的传递与计算机之间。

## ## 关于Socket（套接字） --->   所以 Socket 是一套用于不同主机间通信的 API（传输层【TCP/UDP】上的接口）

![[Pasted image 20230225201623.png]]
**<center>Socket 通信示例 </center>**
![[Pasted image 20230225201801.png]]

现在我们了解到TCP/IP只是一个协议栈，就像操作系统的运行机制一样，必须要具体实现，同时还要提供对外的操作接口。就像操作系统会提供标准的编程接口，比如Win32编程接口一样，TCP/IP也必须对外提供编程接口，这就是Socket。现在我们知道，Socket跟TCP/IP并没有必然的联系。Socket编程接口在设计的时候，就希望也能适应其他的网络协议。所以，Socket的出现只是可以更方便的使用TCP/IP协议栈而已，其对TCP/IP进行了抽象，形成了几个最基本的函数接口。比如create，listen，accept，connect，read和write等等。

在标准库的net包中可供了可移植的网络I/O接口,其中就包含了Socket

Socket在TCP/IP网络分层中并不存在,是对TCP或UDP封装

如果非要给Socket一个解释

实现网络上双向通讯连接的一套API

常称Socket为"套接字"

不同语言都有对应的建立Socket服务端和客户端的库，下面举例 Go 如何创建服务端和客户端：

TCPAddr结构体表示服务器IP和端口

IP是type IP []byte

Port是服务器监听的接口

```go
// TCPAddr represents the address of a TCP end point. 
type TCPAddr struct { 
	IP IP 
	Port int 
	Zone string // IPv6 scoped addressing zone 
} 
TCPConn结构体表示连接,封装了数据读写操作 
// TCPConn is an implementation of the Conn interface for TCP network 
// connections. 
type TCPConn struct {
	conn 
} 
TCPListener负责监听服务器端特定端口 
// TCPListener is a TCP network listener. Clients should typically 
// use variables of type Listener instead of assuming TCP.
type TCPListener struct {
	fd *netFD
}
```

### **客户端向服务端发送消息**

#### 服务端代码:

```go
package main 

import ( 
	"net" 
	"fmt" 
) 

func main() { 
	//创建TCPAddress变量,指定协议tcp4,监听本机8899端口 
	addr, _ := net.ResolveTCPAddr("tcp4", "localhost:8899") 
	
	//监听TCPAddress设定的地址 
	lis, _ := net.ListenTCP("tcp4", addr) 
	
	fmt.Println("服务器已启动") 
	
	//阻塞式等待客户端消息,返回连接对象,用于接收客户端消息或向客户端发送消息 
	conn, _ := lis.Accept() 
	
	//把数据读取到切片中 
	b := make([]byte, 256) 
	fmt.Println("read之前") 
	//客户端没有发送数据且客户端对象没有关闭,Read()将会阻塞,一旦接收到数据就不阻塞 
	count, _ := conn.Read(b) 
	fmt.Println("接收到的数据:", string(b[:count])) 
	//关闭连接 
	conn.Close() 
	fmt.Println("服务器结束") 

}
```

#### 客户端代码：

```go
package main 

import ( 
	"net" 
	"fmt" 
) 

func main() { 
	//服务器端ip和端口 
	addr, _ := net.ResolveTCPAddr("tcp4", "localhost:8899") 
	//申请连接客户端 
	//第二个参数:本地地址 第三个参数:远程地址 
	conn, _ := net.DialTCP("tcp4", nil, addr) 
	//向服务端发送数据 
	count, _ := conn.Write([]byte("客户端传递的数据"))
	fmt.Println("客户端向服务端发送的数据量为:", count) 
	//通过休眠测试客户端对象不关闭,服务器是否能接收到对象 
	//time.Sleep(10 * time.Second) 
	关闭连接 
	conn.Close() 
	//fmt.Println("客户端结束") 

}
```

## JSON 与 XML

XML 和JSON是两种完全不同的数据表达方式。他们分别采用完全不同格式将原始数据转换成XML或者JOSN格式数据；然后再将XML或JOAN格式的数据还原为原始数据。

## 比喻

**一个形象的比喻：TCP/IP是由SOCKET修建公路，HTTP是公路上跑的车，XML或JSON是车装载的货物。**

