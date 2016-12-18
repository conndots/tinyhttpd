# [tinyhttpd TLPI注释版](https://github.com/conndots/tinyhttpd)
> Forked from [cbsheng/tinyhttpd](https://github.com/cbsheng/tinyhttpd)  

tinyhttpd是一个500行的极简HTTP服务器，持CGI。代码量少，非常容易阅读，十分适合网络编程初学者学习的项目。麻雀虽小，五脏俱全。在tinyhttpd中可以学到 linux 上进程的创建，管道的使用。linux 下 socket 编程基本方法和http 协议的最基本结构。  
在cbsheng的基础上，添加了一些注释，帮助阅读源码，针对*The Linux Programming Interface*，使用了章节索引替代了原来的页码索引。  

代码非常简单，和你一样我也是初学者可以多关注一下以下两个方面：
1. Unix Socket Stream Server的通常流程  
2. 使用pipe做父子进程通信
# tinyhttpd流程    
![tinyhttpd frame](http://7xiub6.com1.z0.glb.clouddn.com/tinyhttpd.png)

流程图包含了一个典型的Unix socket stream server的流程，可详见：*TLPI* 56.5.  

# 使用pipe做相关进程通信
Pipe是Unix like系统上最古老的IPC方法。它为一个常见需求提供了一个优雅的解决方案：给定两个运行不同程序的进程，如何让一个进程的输出作为另一个进程的输入？管道可以用于在相关进程之间传递数据。  

tinyhttpd中创建子进程来执行cgi脚本的函数可以很好地用来学习pipe。  
先来看代码。  
```c
/**********************************************************************/
/* Execute a CGI script.  Will need to set environment variables as
 * appropriate.
 * Parameters: client socket descriptor
 *             path to the CGI script */
/**********************************************************************/
void execute_cgi(int client, const char *path, const char *method, const char *query_string)
{
 char buf[1024];
 int cgi_output[2];
 int cgi_input[2];
 pid_t pid;
 int status;
 int i;
 char c;
 int numchars = 1;
 int content_length = -1;

 //往 buf 中填东西以保证能进入下面的 while
 buf[0] = 'A'; buf[1] = '\0';
 //如果是 http 请求是 GET 方法的话读取并忽略请求剩下的内容
 if (strcasecmp(method, "GET") == 0)
  while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
   numchars = get_line(client, buf, sizeof(buf));
 else    /* POST */
 {
  //只有 POST 方法才继续读内容
  numchars = get_line(client, buf, sizeof(buf));
  //这个循环的目的是读出指示 body 长度大小的参数，并记录 body 的长度大小。其余的 header 里面的参数一律忽略
  //注意这里只读完 header 的内容，body 的内容没有读
  while ((numchars > 0) && strcmp("\n", buf))
  {
   buf[15] = '\0';
   if (strcasecmp(buf, "Content-Length:") == 0)
    content_length = atoi(&(buf[16])); //记录 body 的长度大小
   numchars = get_line(client, buf, sizeof(buf));
  }

  //如果 http 请求的 header 没有指示 body 长度大小的参数，则报错返回
  if (content_length == -1) {
   bad_request(client);
   return;
  }
 }

 sprintf(buf, "HTTP/1.0 200 OK\r\n");
 send(client, buf, strlen(buf), 0);

 //下面这里创建两个管道，用于两个进程间通信，参考《TLPI》44.2
 /*
 #include <unistd.h>
 int pipe(int fields); //return 0 on succ, -1 on err.
 成功的pipe()调用会在fields中返回两个打开的文件描述符：一个表示管道的读取端（fields[0]），另一个表示写入端（fields[1]）。
父子进程都通过一个pipe读写信息是可以的，但是很不常见,创建pipe，fork()创建子进程之前：
   [   parent process  ]
 - [fields[1] fields[0]]<-
|                        |
-> [-------pipe------>]-
|                       |
- [fields[1] fields[0]]<-
  [    sub process   ]

通常fork()后，其中一个进程需要立即关闭管道写入端描述符，另一个关闭读取描述符。关闭未使用描述符之后：
  [   parent process  ]
- [fields[1]          ]
|
-> [-------pipe------>]-
                        |
  [          fields[0]]<-
[    sub process   ]
 */
 if (pipe(cgi_output) < 0) {
  cannot_execute(client);
  return;
 }
 if (pipe(cgi_input) < 0) {
  cannot_execute(client);
  return;
 }
 /*
 cgi_output是子进程（执行cgi的进程）的输出管道，子进程写，父进程读；
 cgi_input是子进程（执行cgi的进程）的输入管道，父进程写，子进程读。
 */

 //创建一个子进程 参考《TLPI》 24.2
 /*
 #include <unistd.h>
 pid_t fork(void); //in parent, return processID of child on success or -1 on error; in successfully created child: always return 0
 */
 if ( (pid = fork()) < 0 ) {
  cannot_execute(client);
  return;
 }

 //子进程用来执行 cgi 脚本
 if (pid == 0)  /* child: CGI script */
 {
  char meth_env[255];
  char query_env[255];
  char length_env[255];

  //dup2()包含<unistd.h>中，参读《TLPI》5.5
  //将子进程的输出由标准输出重定向到 cgi_ouput 的管道写端上
  /*
  #include <unistd.h>
  int dup2(int oldfd, int newfd); //return (new) file descritor on succ, -1 on err
  为oldfd指定文件描述符创建副本，其编号由newfd指定。
  */
  dup2(cgi_output[1], 1);
  //将子进程的输出由标准输入重定向到 cgi_ouput 的管道读端上
  dup2(cgi_input[0], 0);
  //关闭 cgi_ouput 管道的读端与cgi_input 管道的写端
  close(cgi_output[0]);
  close(cgi_input[1]);

  //构造一个环境变量
  sprintf(meth_env, "REQUEST_METHOD=%s", method);
  //putenv()包含于<stdlib.h>中，参读《TLPI》6.7
  //将这个环境变量加进子进程的运行环境中
  /*
  #include <stdlib.h>
  int putenv(char *string); //return 0 on succ, nonzero on err.
  */
  putenv(meth_env);

  //根据http 请求的不同方法，构造并存储不同的环境变量
  if (strcasecmp(method, "GET") == 0) {
   sprintf(query_env, "QUERY_STRING=%s", query_string);
   putenv(query_env);
  }
  else {   /* POST */
   sprintf(length_env, "CONTENT_LENGTH=%d", content_length);
   putenv(length_env);
  }

  //execl()包含于<unistd.h>中，参读《TLPI》P567
  //最后将子进程替换成另一个进程并执行 cgi 脚本
  /*
  #include <unistd.h>
  int execl(const char* pathname, const char *arg, ...); //not return on succ;return -1 on error.
  */
  execl(path, path, NULL);
  exit(0);

 } else {    /* parent */
  //父进程则关闭了 cgi_output管道的写端和 cgi_input 管道的读端
  close(cgi_output[1]);
  close(cgi_input[0]);

  //如果是 POST 方法的话就继续读 body 的内容，并写到 cgi_input 管道里让子进程去读
  if (strcasecmp(method, "POST") == 0)
   for (i = 0; i < content_length; i++) {
    recv(client, &c, 1, 0);
    write(cgi_input[1], &c, 1);
   }

  //然后从 cgi_output 管道中读子进程的输出，并发送到客户端去
  while (read(cgi_output[0], &c, 1) > 0)
   send(client, &c, 1, 0);

  //关闭管道
  close(cgi_output[0]);
  close(cgi_input[1]);
  //等待子进程的退出 《TLPI》26.1.2
  /*
  #include <sys/wait.h>
  pid_t waitpid(pid_t pid, int *status, int options); //return process ID of child, 0, or -1 on err.
  */
  waitpid(pid, &status, 0);
 }
}
```  

![pipe_fork](http://7xiub6.com1.z0.glb.clouddn.com/fork_pipe.png)
