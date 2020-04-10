rttim
=====

IM, video and voice chat
1 video cap<br>
http://blog.chinaunix.net/uid-15063109-id-4530994.html

2 video encode and decode <br>
http://blog.chinaunix.net/uid-15063109-id-4531903.html


3 udp send and recv <br>
http://blog.chinaunix.net/uid-15063109-id-4532768.html<br>
http://blog.chinaunix.net/uid-15063109-id-4536610.html<br>
http://blog.chinaunix.net/uid-15063109-id-4544203.html

4 stun server<br>
http://blog.chinaunix.net/uid-15063109-id-4548099.html

这个项目只是演示sip视频基础通话用的。要想做成完善产品还有很多细节需要做。
如果大家有问题请发到sxcong@163.com上，收到后会及时回复。


=====================================================

【Network】UDP 大包怎么发？ MTU怎么设置？
      这里主要用UDP来发送视频，当发送的数据大于1500时分包发送，保证每包小于1500.
发送好办，分割后循环发就可以了，关键是接收时的处理。
先做一下处理的方法 ：
发送时每包上面加上标识，比如RTP的做法是加时间戳，SSRC，媒体类型还有结束标识。简单参考一下，我们也加上一些标识(直接拿RTP头也可以， 不过我们的目标是更简洁一些)。另外，我们的目的和RTP稍有不同，UDP库当时设计是传输数据，而不关心数据的内容是什么。这样，简单一化，包头第一个INT定义为类型，第二个INT为序号，第三个为结束标志，之后是数据长度和数据内容。
代码演示：

int CUDPSession::SplitData(char* pBuff, uint32_t nLen)
{
int nBlockNum = nLen / UDP_BLOCK_SIZE;
if (nLen % UDP_BLOCK_SIZE != 0)
{
nBlockNum++;
}


int sendlen = 0;
for (int i = 0; i < nBlockNum; i++)
{
int poayload_size = UDP_BLOCK_SIZE;
char* payload = pBuff + UDP_BLOCK_SIZE * (i);


if (nLen - UDP_BLOCK_SIZE * (i) < UDP_BLOCK_SIZE)
poayload_size = nLen - UDP_BLOCK_SIZE * (i);


CPackOut* pack = new CPackOut;
(*pack) << PACK_TYPE_DATA;
(*pack) << m_nFrameIndex;
if (i == nBlockNum - 1)
{
(*pack) << 1;
}
else
{
(*pack) << 0;
}
(*pack) << poayload_size;
(*pack).SetBuffer(payload, poayload_size);

int ret = SendPacket(pack, m_destIP, m_destPort);
printf("SendPacket ret = %d\n", ret);
sendlen += ret;
delete pack;
pack = NULL;
}

return sendlen;
}


发送端就这么简单。


下面详细说一下接收端:


因为UDP是不可靠的，不保证数据帧一定正常到达，即使收到，顺序也可能发生变化，比如先发的后到，当然丢包的可能最大，乱序的情况比较少。
正常的处理方法一般这样：
假设一个端口只接收固定一个对方数据源,这样，收到一个数据包放到缓冲里，然后在缓冲里根据帧的序号排序(每一帧的大序号是相同的，自己可以给每一个小片加上小序号,包头里可以加上本次数据帧一共分多少片，收到一片就统计一下，判断是否收齐)。 当收齐后，这个帧去掉包头回调给上层。当在一定时间内该帧数据还没有收齐，就说明传输过程有丢包了，把已收到的都丢掉就可以。 
当上层的应该收到回调的数据后，可以进行解码播放。不过在解码之前，先判断一下帧序列是否连续。做为视频数据，

如果中间有缺少的，就把这一序列都丢掉，直到下一个I帧。每个帧的序号，最好收发之间协商好，在发送的时候带上。


如果把上面整个过程都实现，完全自己写的话，是需要几天的时间。不过，从很多RTP开源库里发现，处理的都非常简单，很多都没有管乱序情况，简单地来一份数据就向缓冲里追加一份，直到发现mark为1。我们这里做为简单使用的项目，也采用了这种简单方法，先把功能完成，之后有时间再来优化。
简单的重组代码:

int CUDPSession::Reassemble(CPackIn& pack, uint32_t ip, uint32_t port)
{
int nSeq = 0;
int nMark = 0;
int nLen = 0;
pack >> nSeq;
pack >> nMark;
pack >> nLen;


if (m_nRecvFrameIndex != nSeq)
{
if (m_buffer)
{
evbuffer_free(m_buffer);
m_buffer = NULL;
}
m_nRecvFrameIndex = nSeq;
}


if (m_buffer == NULL)
{
m_buffer = evbuffer_new();
}

char* pBuf = 0;
int   nSize;
pack.GetBuffer(pBuf, nSize);


evbuffer_add(m_buffer, pBuf, nSize);


if (nMark == 1)
{
//回调
if (m_pCB)
{
m_pCB(m_udpIO.m_handle, (char*)(m_buffer->buffer), m_buffer->off, ip, port, 


m_pParam);
}
evbuffer_free(m_buffer);
m_buffer = NULL;
}


return 0;
}

程序里使用的evbuffer，是从libevent里面拿来的，主要用来处理数据缓冲，非常好用，效率也很好，见evbuffer.h和buffer.cpp。
完整代码在git上，这次实现的功能是：本机UDP bind5500端口-->摄像机采集-->编码-->发送给本机的5500端口-->收到后再解码--> 显示。
发送的代码：m_Sess.Send((char*)pData, nLen, inet_addr("127.0.0.1"), 5500);
这个程序可以分别运行在两台机器上，一台是发送，另一台是接收。发送方只要把上面这一句里面的127.0.0.1换上你目标的ip，另一台机器就可以接收并解码了。

本文结束后，完整的客户端功能基本就差不多了，下一步开始完成server端的stun, 协商穿透, 实p2p和中转视频。

另补充一下，如果发送的是int类型的数据，一般应该用htonl,htons和ntohl,ntohs转换一下。这里的代码暂时没有用，以后会完善的。如果自己写代码应该注意。

https://github.com/sxcong/rttim
 
参考资料
java - Which is the best approach to send large UDP packets in sequence - Stack Overflow
networking - sending a big file in UDP in C - Stack Overflow
sockets - is it possible to send very big data over UDP? - Stack Overflow
c - How to send UDP packets of size greater than 64 KB - Stack Overflow
udp - How to send large data using C# UdpClient? - Stack Overflow
sockets - Python: Sending large object over UDP - Stack Overflow
UDP and packets greater than 1500 bytes
UDP and packets larger than 1500 bytes / Networking, Server, and Protection / Arch Linux Forums
UDP主要丢包原因及具体问题分析
UDP需要自己分包么 - C/C++-ChinaUnix.net
音视频聊天开发: 5 UDP发送视频数据的分包和重组 -sxcong-ChinaUnix博客
Neutron 理解 (3): Open vSwitch + GRE/VxLAN 组网 - 极客头条 - CSDN.NET
TCP/IP具体解释--TCP/UDP优化设置总结&amp; MTU的相关介绍 - yxwkaifa - 博客园
