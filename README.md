# RouteTableChange
连接ssh vpn后，通过修改路由表，实现vpn和机器网络同时使用。主要针对MotionPro这款软件，可用于北京工业大学等
学校内部教务等系统不对外开放，但通过vpn可以实现访问，遗憾的是连上vpn后，就只能访问学校站点，无法连接到外网。针对这个问题，经过测试，可以通过修改路由表的方式，来实现外网和校园网同时访问。
解决方法分Linux版本和Windows版本

先介绍学校用的VPN类型：
--------
学校使用的是SSL VPN，根据学校vpn网站显示，电脑可用一个“信安世纪”的ssl vpn客户端连接，移动设备则通过Arry Network的MotionPro连接。笔者最初采用信安世纪的ssl vpn客户端连接，然后企图通过修改路由表的方式来实现内外网同时访问，可是每次设置后，路由表会变回来，经过查看vpn客户端的日志发现，信安世纪的vpn客户端的进程具有路由表变化检测的功能，所以再怎么修改路由也是徒劳。想通过编程方式来阻止或者欺骗vpn客户端的检测功能，短时间内不太靠谱。所以把目光放到了移动设备使用的MotionPro这款工具上来，再移动设备上只需要输入学校vpn的地址“vpn.bjut.edu.cn”和自己的学号、密码就可以实现vpn连接，完全不想PC端那么复杂，至少在用户体验层面。于是再次百度“ssl vpn客户端”关键字，基本无结果。然后Google，Bing都没有比较满意的结果，最后去搜索MotionPro的官网，也就是ArryNetwork的官网，这是一个专业做网络相关内容的公司，于是在该公司官网上瞎看一圈还被误导注册了一个账号后，还是没有结果。最后无奈又无奈，当光与奇迹就在这个时候出现了，百度“Arry network”后，第二条就是“ArrayVPN客户端软件下载页”（http://support.arraynetworks.com.cn/troubleshooting/ ） ， 打开后简直发现了新世界，里面有MotionPro的各种版本，Linxu,MacOS，Windows等。于是就开始了下面的具体实践。
### 一.Windows server 2012
   安装windows版本的MotionPro，然后连接。
   修改路由表的命令一共三句
   第一句“route add 172.0.0.0 mask 255.0.0.0 172.29.128.1”
       因为学校所有的站点IP基本都是172.0.0.0段，所以只需要添加一条路由规则，其中172.29.128.1是vpn连接的网关，具体的接口if可以省略不写，因为他默认会设置if为vpn连接对应的那个接口号
   第二句“route delete 0.0.0.0”
       这句就是删除默认路由，这时候路由表中已经显示指定的ip外的所有ip地址均不可达，删除这条的理由在于连接vpn后，针对默认路由的网关是指向vpn连接的，这样会阻止我们访问外网
   第三句"route add 0.0.0.0 mask 0.0.0.0 10.214.8.1 if 13"
       对应第二局，这句的含义很明显，就是让默认路由指向机器的物理网卡的网关，注意要指定接口号，否则接口好会被设置为vpn连接的接口号。
   以上三句后，就可以同时访问内网和外网了！
   但是上面的步骤只是针对一台物理机器可以这么设置，如果你要操作的windows是一台虚拟机，而且是远程连接的，那么这么是没有机会执行以上三句的，因为连接vpn后，虚拟机的远程连接就断开了。
   这个也好处理，就是用windows自带的定时任务来完成，可以将上面三句写入个批处理脚本中，然后到控制面板设置一个定时任务。这样让她自动完成，当处理完成后，就再次连接远程虚拟机了。
#### 二.Linux Ubuntu 14
    个人对Linux完全是新手的状态，那为什么还要做Linux的尝试呢？因为我用了反向代理去访问校内网站，Windows上IIS虽然可以反向代理，但是部分页面会出现502错误，纠结两天，无法解决，所以又选择了Nginx,Nginx倒是可以成功实现反向代理，但是速度惨不忍睹。所以决定弃用Windows!
    选择Linux平台后，也需要去安装MotionPro,但是这是一个图形化的软件，不支持命令。所以我又花了一下午的时间装了一台桌面版的Ubuntu16来测试。同样再Linux下，也需要调整路由表。
    但是在Linux中，再开启MotionPro vpn的情况下，如果删除default路由条目，然后添加修改网关和接口后的default2路由，整个路由表会被重置，这我也没搞清楚为什么。于是window中的套路不能用了。幸运的是，仅仅删除defalut路由后，并不会导致路由表重置。这就好办了，既然不能添加default路由，那就从添加从1到255的路由条目，这样同样可以实现路由的目的，而且不会导致路由表被重置。但是这么写少说也要255条命令，所以写一个sh脚本是必须的了，如是对应route_add.sh
    同样再Linux虚拟机中，也需要一个定时脚本来执行sh，否则连接vpn后会导致掉线。于是有了route.cron这个定时脚本。
    最后rant.sh和rant.cron就是为了防止长时间无请求导致vpn被断开而做的心跳脚本。
