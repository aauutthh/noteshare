# nginx 0.1.0 
## 下载地址
[nginx 0.1.0](https://github.com/nginx/nginx/releases?after=release-0.1.2)
该地址从github下载0.1.0 

## 编译及执行环境说明
 系统: debian 8.0
 内核: 4.8.0
 gcc : 4.9.2
 CPU架构: x86_64

 注: nginx0.1只支持类unix系统，所以windows用户强行使用该指引会导致精神错乱    

## patch使用方法
 ```
 cd /tmp
 tar -zxvf nginx-release-0.1.0.tar.gz
 cd nginx-release-0.1.0
 patch -p1 < nginx-release-0.1.0-patch.txt
 ```

## patch 说明
 1. 执行`patch -p1 < nginx-release-0.1.0-patch.txt`后会输出以下信息,后附详细说明，为方便理解顺序有调整       
  ```
  patching file auto/cc
  patching file auto/configure
  patching file auto/feature
  patching file auto/fmt/fmt
  patching file auto/fmt/ptrfmt
  patching file auto/func
  patching file auto/install
  patching file auto/types/sizeof
  patching file auto/unix
  patching file conf/nginx.conf
  patching file ReadMe.txt
  patching file src/core/nginx.c
  patching file src/http/ngx_http_core_module.c
  patching file src/os/unix/ngx_alloc.c
  patching file src/os/unix/ngx_linux_config.h
  patching file src/os/unix/ngx_linux_init.c
  patching file src/os/unix/ngx_writev_chain.c
  ```
 
 1. auto中文件是自动化编译脚本,以下内容为摘抄`nginx-release-0.1.0-patch.txt`片段作说明    
   - `auto/cc`    
   ```
   @@ -60,7 +60,7 @@
   -         CFLAGS="$CFLAGS -Wno-unused"
   +         CFLAGS="$CFLAGS -Wno-unused -Wno-unused-parameter -Wno-pointer-sign"
   ```    
   该脚本检查系统的编译器，并定义`$CC`和`$CC_STRONG`编译命令,及编译选项        
   由于高版本gcc对定义但未使用变量(包括局部变量，全局变量，函数参数列表)会进行细化的告警,而nginx代码中由于系统兼容性问题，会在参数列表中增加冗余变量，但不一定会被使用。因此增加`-Wno-unused-parameter`以免告警并退出(nginx 使用了-Wall -Werror,强制所有告警都退出)。    
  `-Wno-pointer-sign`高版本gcc会严格检查指针类型，像`int*` -> `size_t*`这样的隐性转换会告警。 nginx中sysctl的使用遇到这个问题。   
   
  - `auto/configure`     
    ```
    @@ -7,6 +7,7 @@
    +test -d back || mkdir back
     test -d $OBJS || mkdir $OBJS
     echo > $NGX_AUTO_CONFIG_H
    ```   
    该脚本配置并生成编译环境      
    加入back目录是为了后续对configure生成的测试代码进行保留，以便本文后续进行讲解及让读者更深的理解configure如何对编译系统进行测试并传递信息给nginx编译     

  - `auto/install`        
    ```
    @@ -25,7 +25,7 @@
    -	test -d $PREFIX/html || cp -r html $PREFIX
    +	test -d $PREFIX/html || cp -r docs/html $PREFIX
     
     	#test -d $PREFIX/temp || mkdir -p $PREFIX/temp
     END
    @@ -51,7 +51,7 @@
     clean:
    -	rm -rf Makefile $OBJS
    +	rm -rf Makefile $OBJS back
    ```
    生成Makefile中的`install`,`clean`等target   
    html一行有bug ,需要修改路径
    back一行你懂的，参考`auto/configure`
  
  - `auto/feature`     
 
    ```
    @@ -53,4 +53,6 @@
    +base=`basename $NGX_AUTOTEST.c`
    +cp $NGX_AUTOTEST.c back/HAVE_${feature}_$base
     rm $NGX_AUTOTEST*
    ```   
    该脚本被`fmt` `unix`等脚本调用，通过接收的变量并渲染模板生成`autotest.c`文件(保存到back),当顺利编译并执行代码,则会设定HAVE_XXX这样的变量，并传递给Makefile和gcc。   
    加入了保存到back目录的逻辑    

  - `auto/fmt/fmt`        
    ```
    @@ -44,6 +44,8 @@
             fi
         fi
     
    +    base=`basename $NGX_AUTOTEST.c`
    +    cp $NGX_AUTOTEST.c back/printf_$base
         rm $NGX_AUTOTEST*
    ```   
    该脚本检查printf的格式化特性    
    修改点同`feature`   

  - `auto/fmt/ptrfmt`        

    ```
    @@ -33,6 +33,8 @@
    +    base=`basename $NGX_AUTOTEST.c`
    +    cp $NGX_AUTOTEST.c back/ptrfmt_${ngx_type}_$base
         rm $NGX_AUTOTEST*
    ```   
    该脚本检查printf的格式化特性      
    修改点同`feature`   
 
  - `auto/func`        
    ```
    @@ -40,4 +40,6 @@
    +base=`basename $NGX_AUTOTEST.c`
    +cp $NGX_AUTOTEST.c back/HAVE_${func}_$base
     rm $NGX_AUTOTEST*
    ```
    该脚本检查重要函数或系统差异化函数是否存在    
    修改点同`feature`

 
  - `auto/types/sizeof`        
    ```
    @@ -12,6 +12,7 @@
     #include <sys/types.h>
     #include <sys/time.h>
    +#include <stdio.h>
     $NGX_UNISTD_H
     #include <signal.h>
     #include <sys/resource.h>
    @@ -32,6 +33,8 @@
    +base=`basename $NGX_AUTOTEST.c`
    +cp $NGX_AUTOTEST.c back/sizeof_${ngx_type}_$base
     rm $NGX_AUTOTEST*
    ```
    该脚本检查系统`int`,`long`,`long long`等类型大小    
    修改点1,`printf`需要头文件,不需解释太多       
    修改点2同`feature`    
 
  - `auto/unix`        
    ```
    @@ -46,7 +46,7 @@
    -CC_WARN=$CC_STRONG
    +CC_WARN="$CC_STRONG -Wno-unused-but-set-variable -Wno-unused-parameter "
     ngx_fmt_collect=no
    ```
    该脚本检查unix系统的特性并配置。unix组织前面提到的`fmt` , `ptrfmt` , `sizeof`脚本按顺序执行   
    修改点: 由于高版本gcc会严格检查变量和参数，导致部分系统实际提供的接口没有正确编译通过，`configure`后缺少一些宏。

 1. `conf`目录是配置文件模板,以下内容为摘抄`nginx-release-0.1.0-patch.txt`片段作说明    

  ```
  @@ -1,5 +1,5 @@
   
  -user  nobody;
  +user  www-data; # Modify-Me if non-root user ,set user to the `whoami`
   worker_processes  3;
  @@ -14,13 +14,14 @@
   http {
       include       conf/mime.types;
       default_type  application/octet-stream;
  +    large_client_header_buffers 4 256; # the core have no default value we add it
   
       sendfile  on;
   
       #gzip  on;
   
       server {
  -        listen  80;
  +        listen  8384; # Modify-Me if non-root user , listen port do not set to <1000 
   
           charset         on;
           source_charset  koi8-r;
  ```
  `user` 如果非root用户执行nginx,不能进行setuid操作，所以user必须设置成真实执行的用户  
  `large_client_header_buffers` nginx代码中没有初始化该变量，导致其值为0, 运行会出错  
  `listen` 非root用户不能绑定1000以下端口

 1. `src`目录是所有代码,以下内容为摘抄`nginx-release-0.1.0-patch.txt`片段作说明    
   - `src/core/nginx.c`
    ```
    @@ -148,9 +148,9 @@
             log->log_level = NGX_LOG_INFO;
         }
     
    -    if (ngx_os_init(log) == NGX_ERROR) {
    -        return 1;
    -    }
    +    //if (ngx_os_init(log) == NGX_ERROR) {
    +    //    return 1;
    +    //}
     
         if (ngx_add_inherited_sockets(&init_cycle) == NGX_ERROR) {
             return 1;
    ```
    `ngx_os_init`系统参数设置, 会调用sysctl， 这个是老版本gcc和linux提供的系统调用。现在sysctl单独作为环境配置命令，不在代码中设置,这部分代码屏蔽   
    `sysctl` 请在shell中执行`sysctl -h` 或 `man sysctl`

   - `src/http/ngx_http_core_module.c`

    ```
    @@ -1345,8 +1345,11 @@
     
         if (conf->large_client_header_buffers.size < conf->connection_pool_size) {
             ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
    -                           "the \"large_client_header_buffers\" size must be "
    -                           "equal to or bigger than \"connection_pool_size\"");
    +                           "the \"large_client_header_buffers\" size [%d] must be "
    +                           "equal to or bigger than \"connection_pool_size\" [%d]",
    +                            conf->large_client_header_buffers.size , 
    +                            conf->connection_pool_size
    +                          );
             return NGX_CONF_ERROR;
         }
    ```
    `large_client_header_buffers`的报错信息，请见`conf/nginx.conf`说明    
 
   - `src/os/unix/ngx_alloc.c`

    ```
    @@ -45,7 +45,7 @@
     
     void *ngx_memalign(size_t alignment, size_t size, ngx_log_t *log)
     {
    -    void  *p;
    +    void  *p=NULL;
    ```
    高版本gcc检查未初始化的指针。   
 
   - `src/os/unix/ngx_linux_config.h`
    ```
    @@ -46,6 +46,7 @@
     #include <sys/ioctl.h>
     #include <sys/sysctl.h>
     #include <netinet/tcp.h>        /* TCP_CORK */
    +#include <limits.h>             /* IOV_MAX */
    ```
    为了宏`IOV_MAX`而加
 
   - `src/os/unix/ngx_writev_chain.c`
    ```
    @@ -3,7 +3,7 @@
    +#define __need_IOV_MAX 1
     #include <ngx_config.h>
     #include <ngx_core.h>
     #include <ngx_event.h>
    ```
    为了宏`IOV_MAX`而加。 必须在引入头文件前设置。    
 
   - `src/os/unix/ngx_linux_init.c`
    ```
    @@ -30,7 +30,8 @@
     
     ngx_int_t ngx_os_init(ngx_log_t *log)
     {
    -    int  name[2], len;
    +    int  name[2]; 
    +    size_t len;
    ```
    代码后面`sysctl`函数参数传入`&len`, 高版本gcc检查指针类型   
 
## 编译安装及执行   
 1. 编译及安装
 ```
 auto/configure --with-poll_module --prefix=/tmp
 make
 make install
 ```

 1. 运行  
 先修改`/tmp/conf/nginx.conf`，作为最小demo,只需修改`user`
 ```
 /tmp/sbin/nginx
 ps -ef|grep nginx
 ```
 之后可以在浏览器上访问到`http://127.0.0.1:8384`

 1. 停止  
 ```
 ps -ef|grep 'sbin/nginx'|awk '{print $2}'|sort -ur|xargs kill
 ```

## 缺隐
1. 没有处理子进程退出信号   
  当子进程出现意外退出时，主进程无动于衷。 子进程会变成`<defunct>`僵尸进程。    
  主进程不会fork新进程处理,主进程仍然会派发消息给子进程处理，部分连接会不响应。   
  杀进程时需要所有子进程一起杀，管理不方便。  




