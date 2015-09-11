`` 对NGX_HTTP_DYUPS_MODULE的分析：
``

# 内容： 

## 一：配置说明

## 二：使用实列

## 三：总结注意点




### 一：配置说明

```nginx
file: conf/nginx.conf

    daemon off;
    error_log logs/error.log debug;

    events {
    }

    http {

        dyups_upstream_conf  conf/upstream.conf;

        include conf/upstream.conf;

        server {
            listen   8080;

            location / { 
                proxy_pass http://$host;
            }
        }

        server {
            listen 8088;
            location / { 
                return 200 "8088";
            }
        }

        server {
            listen 8089;
            location / {
                return 200 "8089";
            }
        }

        server {
            listen 8081;
            location / {
                dyups_interface;  //表示接口站点（即管理主机）
            }
        }
    }


```

指定在某个upstream中启用动态域名解析功能。



### 二：使用实列
* 增加upstream:   
curl -d "server 127.0.0.1:801;server 127.0.0.1:802;" 127.0.0.1:81/upstream/ttlsa3   
* 删除upstream:       
curl -i -X DELETE 127.0.0.1:81/upstream/ttlsa1
* 发送请求      
curl -H "host: ttlsa1" 127.0.0.1:8080






### 三：总结注意点

dyups的总体设计思想：

１.首先是多个worker进程是如何做到信息同步的（如：发送一个delete upstream　如何让所有worker进程都删除）：         
　　　其实dyups的是这样设计的：使用一块共享内存来存放相关的操作命令，过程是这样的－－－当外部发送一个“删除一个upstream”的请求的时候，此时会有一个worker进程接受这个请求　并执行这个命令，　当这个worker进程执行完了的时候，会把该条命令插入共享消息队列中，然后其他的worker定时拉取这个命令queue队列来执行，当然有人会想　那我自己是不是下次也会再次执行这个命令恩？　其实加了相应的逻辑判断----每次获取到该命令的时候会判断这个msg->pid[i] == ngx_pid ?若当前执行过了　　则上述条件为真　会有记录的，　当然又有人会问　那么随着用户发送的请求命令越来月多了　则这个存储命令的队列是不是会越来越大，　答案：　不是的，每一个msg会有一个count计数，当count为worker_process数量的时候就会把该条msg命令删除（表示每一个worker进程都执行过了）。还有个问题就是为啥不是一个worker进程收到命令请求之后就把其命令发送到消息队列恩　这样不是可以让其他的worker进程更快的同步嘛？原因也很简单：这样可以确保这条命令的正确性，只有当前worker进程执行成功了才把该条命令存储到共享消息队列中。

　　　在这里我有一个小的想法来解决多个worker进程的数据同步，利用init_worker_by_lua阶段为每一个worker设置定时函数定期的去拉取share_dict中命令，当然操作者每次操作的命令是存放在share_dict中的（可以把命令当做key,然后把命令相应携带的数据存放在key对应的value中，最终worker获取到根据key进行相应的　参数解析就可以了）。



２．使用沙盒更新，保证后续可回滚:         
　　　这一点思想是值得我们去学习的，就是当我们做一件事情的时候－－－要是这件事会对原有的东西做改变，那么最好先去做一次虚拟性的改变，当其该次虚拟改变成功的时候才去触发我们要做的事情，这样可以让我们做事更安全一点。


３．为了最大程度的提高性能，使用两种锁－－用户可以选择：               
　　　　　  ngx_shmtx_lock和ngx_shmtx_trylock两种锁，其中ngx_shmtx_trylock这个锁每次执行的时候只是去抢一下锁，当没有抢到则立即返回失败。ngx_shmtx_lock锁相当于一个自旋锁（splinlock）当进程执行到此处的时候会是一个死循环的去进行抢锁，当然中间加了一些自己的限制机制（当抢一次失败的时候会增加下一次去抢所的时间间隔，而且抢了很多次后还是抢不到锁的时候就会先让出cpu一会（此处调用ngx_sched_yield让其他的程序先执行－－－找一个优先级等于或是高于当前程序的去执行）），相比之下ngx_shmtx_trylock的性能高多了。　这两个锁都是使用了cpu级别的原子操作，不会发生上下文的切换。        
　　　注意这两个锁和线程锁的区别，线程锁一般是要发生上下文切换的，这个比较耗费系统性能。但是自旋锁也有自身的缺点比如在极端情况下自旋锁会一直死循环占用cpu，浪费资源。


４．当进行删除一个upstream的时候，dyups在这里踩的坑还是蛮多的：       
　　　在这里我突然想到了c++中的delete+析构函数了，同一个道理，但是在我们现实codeing中还是踩上了这个地雷。delete 一个对象的时候的思想：析构一个对象的过程，其实做了两件事情：(1).调用析构函数　(2).调用释放空间的函数。其实就是先把对象给搞走，然后把对象的空间释放了。在dyups中我们删除一个upstream的时候要确保别处都没有使用这个upstream了才可以去真正的释放，比如在使用了keepalive的时候，要真的等到没有在使用了才可以真的去释放，在dyups中是这么做的使用一个ref引用计数来标记每一个upstream只有当ref为０的时候才可以删除upstream. 这里的ref引用计数分为两部分：(1).client request 和 upstream connection
在 client request 的 init_peer 和 r->pool 的 clean_handle 里对 client request 进行引用计数的加减。(2).在 upstream connection 的 free_peer 和 pc->connection->pool 的clean handler 里进行对后端连接的引用计数的加减。只有最后ref为０才去真的删除。


５．关于proxy_pass后面跟变量的时候的效率问题：                  
　　　当我们在proxy_pass后面跟一个变量的时候，其nginx会遍历所有的upstream的，如果当我们有大量的upstream的时候，这可能是一个性能瓶颈。其tengine是这样解决的使用了red_balack_tree进行存储upstream。


６．关于删除一个upstream的时候，如果proxy_pass后面没有使用变量，则删除是没有生效的－－－当来一个请求的时候还是会访问到已经删除的：        
　　　原因就是当在proxy_pass后面跟的不是变量的时候，其nginx是不会去查找upstream　list的，而是使用直接原来的配置文件中的upstream结构。
       
    if(u->resolved == NULL) {　　//不使用变量的时候resolved就是NULL 这个的原因要追杀到proxy_pass中，由于直接给的值就没有变量了也就不会执行函数ngx_http_script_compile所以proxy_length就是NULL最后就不会执行ngx_http_proxy_eval函数。

        uscf = u->conf->upstream;
    } else {}
 

## 有问题反馈
在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

* 邮件(1031379296#qq.com, 把#换成@)
* QQ: 1031379296
* weibo: [@王发康](http://weibo.com/u/2786211992/home)


## 感激

### chunshengsterATgmail.com


## 关于作者

### Linux\nginx\golang\c\c++爱好者
### 欢迎一起交流  一起学习# 
