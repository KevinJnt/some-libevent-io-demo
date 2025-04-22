读：
```cpp 
  1 #include<stdio.h>
  2 #include<string.h>
  3 #include<stdlib.h>
  4 #include<unistd.h>
  5 #include<error.h>
  6 #include<fcntl.h>
  7 #include <pthread.h>
  8 #include<event2/event.h>
  9 #include <sys/stat.h>
 10 
 11 void sys_err(const char *str){
 12     perror(str);
 13     exit(1);
 14 }
 15 
 16     void read_cb(evutil_socket_t fd,short what,void * arg){
 17     char buf[BUFSIZ] = {0};
 18     int len = read(fd, buf, sizeof(buf));
 19 
 20     printf("what = %s, read form write : %s\n",
 21     what & EV_READ ? "READ YES":"NO",buf);
 22     what & EV_READ ? "READ YES":"NO",buf);
 23     what & EV_READ ? "READ YES":"NO",buf);
 24 
 25     
 26 
 27     sleep(1);
 28 }
 29 int main(int argc, char *argv[]) {
 30 
 31     //创建fifo
 32     unlink("testfifo");
 33     mkfifo("testfifo",0644);
 34 
 35     //打开fifo读端
 36     int fd =open("testfifo",O_RDONLY | O_NONBLOCK);
 37     if (fd== -1 ){
 38     sys_err("open error ");
 39     }
 40 
 41     //创建event_base
 42     struct event_base *base =event_base_new();
 43 
 44     //创建事件
 45     struct event *ev =NULL;
 46 
 47     ev=event_new(base,fd,EV_READ | EV_PERSIST,read_cb,NULL);
 48 
 49     //添加事件到event_base上
 50     event_add(ev,NULL);
 51 
 52     //启动循环
 53     event_base_dispatch(base);
 54 
 55     //销毁event_base;
 56     event_base_free(base);
 57 
 58     return 0;
 59 }
写：
1 #include<stdio.h>
  2 #include<string.h>
  3 #include<stdlib.h>
  4 #include<unistd.h>
  5 #include<error.h>
  6 #include<fcntl.h>
  7 #include <pthread.h>
  8 #include<event2/event.h>
  9 #include <sys/stat.h>
 10 
 11 void sys_err(const char *str){
 12     perror(str);
 13     exit(1);
 14 }
 15 
 16 void write_cb(evutil_socket_t fd,short what,void * arg){
 17     char buf[] = "kevin";
 18     write(fd, buf, strlen(buf)+1);
 19 
 20     sleep(1);
 21 }
 22 int main(int argc, char *argv[]) {
 23 
 24     //打开fifo写端
 25     int fd =open("testfifo",O_WRONLY | O_NONBLOCK);
 26     if (fd== -1 ){
 27     sys_err("open error ");
 28     }
 29 
 30     //创建event_base
 31     struct event_base *base =event_base_new();
 32 
 33     //创建事件
 34     struct event *ev =NULL;
 35 
 36     ev=event_new(base,fd,EV_WRITE | EV_PERSIST,write_cb,NULL);
 37 
 38     //添加事件到event_base上
 39     event_add(ev,NULL);
 40 
 41     //启动循环
 42     event_base_dispatch(base);
 43 
 44     //销毁event_base;
 45     event_base_free(base);
 46 
 47     return 0;
 48 }








```