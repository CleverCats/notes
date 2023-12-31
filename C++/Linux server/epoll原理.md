epoll,从Linux内核2.6引入的.2.6之前没有
epoll技术:和kquene技术类似,一台计算机可以支撑少则数万,多则数十万并发连接的核心技术【支持高并发】

一、epoll原理:

1.和传统的多路复用技术select/poll不同如果同时有10W个连接,select/poll会每次检测10W个连接,然后筛选出哪些连接发送了数据,而同一时间可能只有几百个连接发送数据
  但缺检查了10W个,造成大量的性能浪费【select/poll技术通常1000-2000性能就会明显下降】
  而epoll技术只处理这几百个发送数据的客户端,不比检查10W个客户端然后在筛选
  epoll计算不会随着并发量的提高而出现性能下降的情况,但是会消耗一定的内存,所以并发量不可能是无线的

2.epoll时间驱动机制,很多服务器用多 进程/线程 实现,每个 进程/线程 对应一个客户端连接
  而epoll在单独的 进程/线程 中运行,收集/处理 数据 ,因此不会用 进程/线程 间的切换消耗

二、epoll函数:

1.函数一: int_epoll_creat(int size); 
作用: 创建一个epoll对象【struct eventpoll *】,返回这个对象的文件描述符【一切皆文件】,这个描述符就代表这个epoll对象
因为是文件描述符,所以需要记得close();【文件描述符（也叫句柄）总是需要关闭的】
size:占时不明确,先保证 >0 就行

函数解析:相当于new了一个eventpoll【结构体】,eventpoll结构体中有两个主要成员【还有其它成员,后续在讲】 
     rbr【代表一颗红黑色的根结点,刚开始指向空】 和rdlist【指向一个双向链表的表头指针】

2.函数二: int epoll_ctl(int efpd , int op ,in sockid , struct epoll_event *event);
作用:将一个socket以及这个socket相关的事件添加到这个epoll对象描述符中去,通过epoll对象来监视这个监听套接字的数据来往情况
     当有系统来往时,系统会通知我们【我们把感兴趣的事件添加上去,当这个事件来了系统就通知我们】
参数1 efpd: int_epoll_creat(int size);返回的文件描述符
参数2 op  : 动作【你要干什么】 有三个参数 添加/删除/修改  1-EPOLL_CTL_ADD、2-EPOLL_CTL_DEL、3-EPOLL_CTL_MOD 【数字-宏】

添加事件:在红黑色上添加一个结点,每个客户端连入服务器后,都会产生一个对应的socket,每个socket都不会重复
         socket就是红黑色【键/值】中的key,那这个结点添加到红黑色上去

修改事件:系统通过红黑色的key【SOCKET】来定位修改,比如你添加了三个事件,你想修改成两个

删除事件:是从红黑色上把某个结点干掉,也就是这个socket上无法接受到任何系统通知了

参数3 sockid:监听套接字             //sockid:表示客户端连接,红黑色中的key【socket】

参数4 event :事件信息,op代表的动作都需要用到这个事件信息 

函数解析:如果op = EPOLL_CTL_ADD 那么先在红黑色上查找是否有这个key【socket】,有的话就不在加入
     没有的话生成一个红黑色结点【epitem *epi】,塞入红黑树,其中epi->rbn代表三个指针,分别代表红黑色的左子树,右子树,和父亲

     如果op = EPOLL_CTL_DEL，如果这个结点存在,那么干掉该结点
     如果op = EPOLL_CTL_MOD,找到结点,修改成给定的事件

强调:红黑色的结点是epoll_ctl[EPOLL_CTL_ADD]往里增加的结点
    删除结点是epoll_ctl[EPOLL_CTL_DEL]【注意不是删除时间,而是直接把结点删除了,这个socket的所有事件都无效了】

----------------------------------------------------------------------------    

强调:这个设计很巧妙实际上epitem还同时可以作为双向链表中的结点
     epitem还有个成员epi->rlink【记录了双向链表中的前驱结点和后继结结点】

----------------------------------------------------------------------------

总结:
    EPOLL_CTL_ADD:往红黑树里增加结点
    EPOLL_CTL_DEL:往红黑树里删除结点
    EPOLL_CTL_MOD:修改已有的红黑色结点

3.函数三:int epoll_wait(int epfd,struct epoll_event *events,int maxevents,int timeout);
作用:阻塞一小段时间等待事件发送,返回事件集合,也就是获取内核的事件通知
     说白了就是遍历整个双向链表,把这个双向链表的数据拷贝出去,拷贝完毕就从双向链表里移除
注意:双向链表里面储存的是所有红黑色记录的结点中【TCP连接】,有数据发送的socket【TCP连接】,红黑树100W个结点可能同一时间只有100多个有事件发生的结点

1.参数1   efpd:   int_epoll_creat(int size);返回的文件描述符
2.参数2 events:   将发生的事件的结点往events里面拷贝,events是内存也是数组
3.参数3 maxevents:发生的事件数, 表示events的长度
4.参数4 timeout  :阻塞/卡住的时间长度,-1是永久阻塞直到事件发生，

   不想阻塞一般设置100ms
返回值:事件发生事件的TCP连接数目

内核向双向链表中增加结点的情况
1.客户端完成三次握手,服务器要accept（）
2.客户端要关闭连接时,服务器要close()
3.客户端发送数据时,服务器要read(),write()
4.当可以发送数据时【例如服务端发送数据比较快,客户端比较慢,当客户端接受完了发送包告诉服务端发送完了服务端才可以发送数据】
