# TCP和UDP的区别

**UDP协议：** 无连接（不需要三次握手和四次挥手）、尽最大努力交付、基于数据报（每次收发都是一整个报文段）、没有拥塞控制、没有流量控制、不可靠（只管发不管过程和结果）、支持一对一、一对多、多对一和多对多的通信方式、首部开销很小（8字节）。优点是快，没有TCP各种机制，少了很多首部信息和重复确认的过程，节省了大量的网络资源。缺点是不可靠不稳定，只管数据的发送不管过程和结果，网络不好的时候很容易造成数据丢失。又因为网络不好的时候不会影响到主机数据报的发送速率，这对很多实时的应用程序很重要，因为像语音通话、视频会议等要求源主机要以恒定的速率发送数据报，允许网络不好的时候丢失一些数据，但不允许太大的延迟，UDP很适合这种要求。

**TCP协议：** 有连接（需要三次握手四次挥手）、基于字节流（不保留数据报边界的情况下以字节流的方式进行传输，这也是长连接的由来）、有拥塞控制、有流量控制、可靠交付（有大量的机制保护TCP连接数据的可靠性）、单播（只能端对端的连接）、全双工通讯（允许双方同时发送信息，也是四次挥手的原由）、头部开销大（最少20字节）。优点是可靠、稳定，有确认、窗口、重传、拥塞控制机制，在数据传完之后，还会断开连接用来节约系统资源。缺点是慢，效率低，占用系统资源高，在传递数据之前要先建立连接，这会消耗时间，而且在数据传递时，确认机制、重传机制、拥塞机制等都会消耗大量的时间，而且要在每台设备上维护所有的传输连接。在要求数据准确、对速度没有硬性要求的场景有很好的表现，比如在FTP（文件传输）、HTTP/HTTPS（超文本传输），TCP很适合这种要求。