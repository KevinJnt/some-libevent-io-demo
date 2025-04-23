服务器：
1 创建event_base（底座）
2 创建服务器连接监听器 evconnlistener_new_bind()；它内部会监听客户端的连接，有的话就调用回调
3 在evconnlistener_new_bind()的回调函数中，处理接受与客户端连接后的操作
4 监听器回调函数被调用，说明有一个新客户端连接上来，会得到一个新的cfd，用于跟客户端通信
5 在监听器的回调中使用 bufferevent_socket_new() 创建一个新 bufferevent事件，将cfd封装到这个事件对象中
6 在监听器的回调中使用 bufferevent_setcb() 给这个事件对象的 read、write、event设置回调
7 在监听器的回调中设置 bufferevent的读写缓冲区 enable/disable
8 接受、发送数据 bufferevent_read()/bufferevent_write()-->在bufferevent的读写回调中进行
9 启动循环监听 event_base_dispatch(base)
10 释放资源 free
```cpp 
#include<stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <sys/socet.h>
#include <apra/inet.h>
#include <errno.h>
#include <event2/listener.h>
#include <event2/bufferevent.h>

//bev的读写回调
void read_cb(struct bufferevent *bev,void *arg){
    char buf[1024];

    //从客户端读
    bufferevent_read(bev,buf,sizeof(buf));
    printf("client say : %s\n",buf);

    char *p ="我是服务器，已经成功收到你发送的数据！"；

    //写数据给客户端
    bufferevent_write(bev,p,strlen(p)+1);

    sleep(1);
}

void write_cb(struct bufferevent *bev ,void *arg){
    printf("我是服务器，成功写数据给客户端了，写缓冲区回调函数被调用\n");
}

//事件回调
void event_cb(struct bufferevent *bev ,short events, void *arg){
    
    if(events &BEV_EVENT_EOF)
    {
        printf("connection closed\n");
    }
    else if(events & BEV_EVENT_ERROR)
    {
        printf("some other error \n");
    }

    bufferevent_free(bev);
    printf("bufferevent 资源已经被释放 \n");
}
//监听回调
void cb_listener(
        struct evconnlistener *listenner,
        evutil_socket_t fd,
        struct sockaddr *addr,
        int len,void *ptr )
        {
            printf("connect new client\n");
            
            //接收监听器传进来的base
            struct event_base * base =(struct event_base *)P_DETACH;
            
            //创建bev对象，用于监听客户端的读写事件
            struct buffervent *bev;
            bev = buffervent_socket_new(base,fd,BEV_OPT_CLOSE_ON_FREE);

            //给bev对象的读写回调
            buffervent_setcb(bev,read_cb,write_cb,event_cb,NULL);

            //启用 buffervent的读缓冲， 默认关闭
            buffervent_enable(bev,EV_READ);
        }

int main(int argc,const char *){
    struct sockaddr_in serv;

    memset(&serv,0,sizeof(serv));
    serv.sin_family=AF_INET;
    serv.sin_port= htons(2005);
    serv.sin.addr.s_addr =htonl(INADDR_ANY);

    // 创建底座
    struct event_base* base = event_base_new();

    //创建listener监听对象  （是否有客户端连接）
    struct evconnlistenlistener * listener;
    listener = evconnlistenlistener_new_bind(base,cb_listener,base,
                                            LEV_OPT_CLOSE_REEE | LEV_OPt_REUSEABLE,
                                            36,(struct sockaddr *)&serv,sizeof(serv));
    //启动循环监听
    event_base_dispatch(base);
    
    //释放
    evconnlistenlistener_free(listener);//监听
    event_base_free(base);//底座
    return 0;
}
```
客户端：

``` cpp
#include<stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <sys/socet.h>
#include <apra/inet.h>
#include <errno.h>
#include <event2/listener.h>
#include <event2/bufferevent.h>

//读写回调
void read_cb(struct bufferevent *bev,void *arg)
{
    char buf[1024]=[0];
    bufferevent_read(bev,sizeof(buf));

    printf("服务器 say：%s\n",buf);
    bufferevent_write(bev,buf,strlen(buf)+1);
    sleep(1);
}

void write_cb(struct bufferevent *bev,void *arg)
{
    printf("--------我是客户端的写回调函数，没卵用\n");
}

void event_cb(struct bufferevent *bev,short events,void *arg)
{
    if (events &BEV_EVENT_EOF)
    {
        printf("connection closed\n");
    }
    else if (events &BEV_EVENTS_ERROR)
    {
        printf("some other error\n");
    }
    else if (events & BEV_EVENT_CONNECTED)
    {
        printf("已经连接服务器，\\(^o^)//\n");
        return;
    }
    //释放资源
    bufferevent_free(bev);    

}
int main()
{
    //创建base底座
    struct event_base * base = event_base_new();

    //描述符
    int fd=socket(AF_INET,,SOCK_STREAM,0);

    //地址结构
    struct sockaddr_in client_addr;
    memset(&client_addr,0,sizeof(client_addr));
    client_addr.sin_family=AF_INET;
    client_addr.sin_port=htons(2005);
    inet_pton(AF_INET,"127.1",&client_addr.sin_addr.s_addr);

    //创建bev对象
    struct bufferevent *bev=NULL;
    bev=bufferevent_socket_new(base,fd,BEV_OPt_CLOSE_ON_FREE);

    //连接服务器
    bufferevent_socket_connect(bev,(struct sockaddr *)&client_addr,sizeof(client_addr));

    //设置回调
    bufferevent_setcb(bev,read_cb,write_cb,event_cb,NULL);

    //设置读回调生效
    bufferevent_enable(bev,EV_READ);

    //启动循环监听
    event_base_diapatch(base);

    //销毁
    event_base_free(base);
    bufferevent_free(bev);


    return 0;
}
```