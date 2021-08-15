#  用忽略configure的方式 交叉编译 静态链接 的 tcpdump


## 环境准备

纯净的ubuntu20.04，[使用tuna换源](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)，并安装好一些基础的工具，编译tcpdump需要的依赖软件，以及交叉编译工具链：

```shell
$ sudo apt install -y vim 
$ sudo vim /etc/apt/sources.list
$ sudo apt update
$ sudo apt install -y  gcc flex bison make
$ sudo apt install -y  gcc-aarch64-linux-gnu gcc-arm-linux-gnueabi gcc-mipsel-linux-gnu gcc-mips-linux-gnu
```

交叉编译工具链可以按照如下方法搜索：

```shell
$ sudo apt search arm | grep gcc 
$ sudo apt search mip | grep gcc 
```


## 下载源码

tcpdump官网：[https://www.tcpdump.org/](https://www.tcpdump.org/)

```shell
$ wget https://www.tcpdump.org/release/tcpdump-4.99.1.tar.gz
$ wget https://www.tcpdump.org/release/libpcap-1.10.1.tar.gz
$ tar -xvzf ./tcpdump-4.99.1.tar.gz
$ tar -xvzf ./libpcap-1.10.1.tar.gz
```

## 本地编译

首先本地编译出x86_64版本的，一个是为了生成正常的makefile，另外也是最开始的排错，如果本地编译都错了，就也先别交叉编译了。

### 编译libpcap

```shell
$ cd libpcap-1.10.1/
$ ./configure 
$ make
$ sudo make install
$ ls /usr/local/lib
libpcap.a  libpcap.so  libpcap.so.1  libpcap.so.1.10.1  pkgconfig  python3.8
```

最后的路径`/usr/local/lib`可以在makefile中看到：

```shell
prefix = /usr/local
```

如果我们在`./configure`时指明参数`--prefix`，最终就会落实到makefile这个变量中，所以也可以在make时直接使用参数`prefix=/XXX`完成最终install路径的更换：

```shell
$ make prefix=./test install
$ ls -al ./test/
total 32
drwxrwxr-x  6 xuanxuan xuanxuan  4096 Aug 15 12:03 .
drwxr-xr-x 14 xuanxuan xuanxuan 12288 Aug 15 12:03 ..
drwxr-xr-x  2 xuanxuan xuanxuan  4096 Aug 15 12:03 bin
drwxr-xr-x  3 xuanxuan xuanxuan  4096 Aug 15 12:03 include
drwxr-xr-x  3 xuanxuan xuanxuan  4096 Aug 15 12:03 lib
drwxrwxr-x  3 xuanxuan xuanxuan  4096 Aug 15 12:03 share
```

### 编译tcpdump

正常install完`libpcap`到`/usr/local`后，tcpdump的`./configure`即可顺利生成makefile，然后直接make完成编译：

```shell
$ cd tcpdump-4.99.1/
$ ./configure 
$ make
$ file ./tcpdump
./tcpdump: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=ba092098ad91f35f34fd8fffbb0be737d4d40c4f, for GNU/Linux 3.2.0, with debug_info, not stripped
```

如果想编译一个静态链接的tcpdump可以在make时直接加入`LDFLAGS='-static'`参数：

```shell
$ make clean
$ make LDFLAGS='-static'
$ file ./tcpdump
./tcpdump: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=2005c7569e0585b9abe908d7fcd7a2af61edcedc, for GNU/Linux 3.2.0, with debug_info, not stripped
```

测试运行成功：

```shell
$ cp ./tcpdump ../tcpdump-x86_64
$ sudo ../tcpdump-x86_64
tcpdump-x86_64: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens33, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
```

这个`LDFLAGS='-static'`背后的道理必定在makefile中：

```makefile
PROG = tcpdump

all: $(PROG)

$(PROG): $(OBJ) ../libpcap-1.10.1/libpcap.a $(LIBNETDISSECT)
        @rm -f $@
        $(CC) $(FULL_CFLAGS) $(LDFLAGS) -o $@ $(OBJ) $(LIBNETDISSECT) $(LIBS)
```

所以其实不和其他编译代码冲突的话，改FULL_CFLAGS甚至CC也对可以，反正就是拼接：

```shell
$ make CC='gcc -static'
$ file ./tcpdump
./tcpdump: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked
```

## 交叉编译

因为我们之前本地编译以及成功了，所以makefile都已经生成好了，所以直接复用并修改参数即可

### 编译libpcap

在编译libpcap时要注意，这里make install的背后，并不是直接复制拷贝到目标目录，而是也存在着编译：

```makefile
DYEXT = so

install: install-shared install-archive libpcap.pc pcap-config
        [ -d $(DESTDIR)$(libdir) ] || \
            (mkdir -p $(DESTDIR)$(libdir); chmod 755 $(DESTDIR)$(libdir))
        [ -d $(DESTDIR)$(includedir) ] || \
            (mkdir -p $(DESTDIR)$(includedir); chmod 755 $(DESTDIR)$(includedir))
        [ -d $(DESTDIR)$(includedir)/pcap ] || \


install-shared: install-shared-$(DYEXT)

install-shared-so: libpcap.so

libpcap.so: $(OBJ)
        @rm -f $@
        VER=`cat $(srcdir)/VERSION`; \
        MAJOR_VER=`sed 's/\([0-9][0-9]*\)\..*/\1/' $(srcdir)/VERSION`; \
        $(CC) $(LDFLAGS) -shared -Wl,-soname,$@.$$MAJOR_VER \
            -o $@.$$VER $(OBJ) $(ADDLOBJS) $(LIBS)
```

所以在make install时一样要替换CC变量：

> 这里我将编译出的libpcap库放在了：`/home/xuanxuan/Desktop/tcpdump/libpcap/`

```shell
$ cd libpcap-1.10.1/
$ make clean

$ make CC=arm-linux-gnueabi-gcc
$ make CC=arm-linux-gnueabi-gcc prefix=/home/xuanxuan/Desktop/tcpdump/libpcap/arm install
$ make clean

$ make CC=aarch64-linux-gnu-gcc
$ make CC=aarch64-linux-gnu-gcc prefix=/home/xuanxuan/Desktop/tcpdump/libpcap/aarch64 install
$ make clean

$ make CC=mips-linux-gnu-gcc
$ make CC=mips-linux-gnu-gcc prefix=/home/xuanxuan/Desktop/tcpdump/libpcap/mips install
$ make clean

$ make CC=mipsel-linux-gnu-gcc
$ make CC=mipsel-linux-gnu-gcc prefix=/home/xuanxuan/Desktop/tcpdump/libpcap/mipsel install
$ make clean
```

编译好后我们可以挨个看看指令集，均成功编译出对应架构的库：

```shell
$ cd ../libpcap

$ file ./arm/lib/libpcap.so.1.10.1 
./arm/lib/libpcap.so.1.10.1: ELF 32-bit LSB shared object, ARM, EABI5 version 1 (SYSV), dynamically linked, BuildID[sha1]=389c7ca4b588fc8fd63b2449e24efe25ec887595, with debug_info, not stripped

$ file ./aarch64/lib/libpcap.so.1.10.1
./aarch64/lib/libpcap.so.1.10.1: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, BuildID[sha1]=fabffedac9c884a08d694f0400090593df692333, with debug_info, not stripped

$ file ./mips/lib/libpcap.so.1.10.1
./mips/lib/libpcap.so.1.10.1: ELF 32-bit MSB shared object, MIPS, MIPS32 rel2 version 1 (SYSV), dynamically linked, BuildID[sha1]=7b117cfd177dd759f450149744ef784d3b25313f, with debug_info, not stripped

$ file ./mipsel/lib/libpcap.so.1.10.1
./mipsel/lib/libpcap.so.1.10.1: ELF 32-bit LSB shared object, MIPS, MIPS32 rel2 version 1 (SYSV), dynamically linked, BuildID[sha1]=09204ec7fd3b18dfbbb9a752b24e273b308a5827, with debug_info, not stripped
```


### 编译tcpdump

编译tcpdump时，需要libpcap库，具体来说就是：

- 原理：编译时，静态链接需要`.a`的库，动态链接需要`.so`的库，二者都需要`.h`头文件。
- 编译：在gcc中，额外的头文件使用`-I`参数，库代码使用`-L`参数。
- 变量：在makefile的变量中，编译参数一般是`CFLAGS`，链接参数一般是`LDFLAGS`，编译器本身是`CC`

以arm为例，为完成交叉编译，make需要以下三个参数：

- CC=arm-linux-gnueabi-gcc
- CFLAGS='-I/home/xuanxuan/Desktop/tcpdump/libpcap/arm/include'
- LDFLAGS='-L/home/xuanxuan/Desktop/tcpdump/libpcap/arm/lib/ -static' 

最后编译：

```shell
$ cd tcpdump-4.99.1/
$ make clean

$ make CC=arm-linux-gnueabi-gcc CFLAGS='-I/home/xuanxuan/Desktop/tcpdump/libpcap/arm/include' LDFLAGS='-L/home/xuanxuan/Desktop/tcpdump/libpcap/arm/lib/ -static' 
$ file ./tcpdump
./tcpdump: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), statically linked, BuildID[sha1]=2f4af2d4b19f3abff33868d363b4f51530245239, for GNU/Linux 3.2.0, with debug_info, not stripped
$ cp ./tcpdump ../tcpdump-arm
$ make clean

$ make CC=aarch64-linux-gnu-gcc CFLAGS='-I/home/xuanxuan/Desktop/tcpdump/libpcap/aarch64/include' LDFLAGS='-L/home/xuanxuan/Desktop/tcpdump/libpcap/aarch64/lib/ -static' 
$ file ./tcpdump
./tcpdump: ELF 64-bit LSB executable, ARM aarch64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=5b833dabb3175ab4cc0578d0d1ee534924f1201b, for GNU/Linux 3.7.0, with debug_info, not stripped
$ cp ./tcpdump ../tcpdump-aarch64
$ make clean

$ make CC=mipsel-linux-gnu-gcc CFLAGS='-I/home/xuanxuan/Desktop/tcpdump/libpcap/mipsel/include' LDFLAGS='-L/home/xuanxuan/Desktop/tcpdump/libpcap/mipsel/lib/ -static' 
$ file ./tcpdump
./tcpdump: ELF 32-bit LSB executable, MIPS, MIPS32 rel2 version 1 (SYSV), statically linked, BuildID[sha1]=4f5688b096e43e24ef45f8f84590278ebc421c1d, for GNU/Linux 3.2.0, with debug_info, not stripped
$ cp ./tcpdump ../tcpdump-mipsel
$ make clean

$ make CC=mips-linux-gnu-gcc CFLAGS='-I/home/xuanxuan/Desktop/tcpdump/libpcap/mips/include' LDFLAGS='-L/home/xuanxuan/Desktop/tcpdump/libpcap/mips/lib/ -static' 
$ file ./tcpdump
./tcpdump: ELF 32-bit LSB executable, MIPS, MIPS32 rel2 version 1 (SYSV), statically linked, BuildID[sha1]=4f5688b096e43e24ef45f8f84590278ebc421c1d, for GNU/Linux 3.2.0, with debug_info, not stripped
$ cp ./tcpdump ../tcpdump-mips
$ make clean
```

使用qemu测试，均可运行：

```shell
$ cd ..
$ sudo apt install qemu-user
$ qemu-aarch64 ./tcpdump-aarch64 
Unknown host QEMU_IFLA type: 54
Unknown host QEMU_IFLA type: 32820
Unsupported ioctl: cmd=0x8946
tcpdump-aarch64: ens33: SIOCETHTOOL(ETHTOOL_GLINK) ioctl failed: Function not implemented
$ qemu-arm ./tcpdump-arm 
Unknown host QEMU_IFLA type: 54
Unknown host QEMU_IFLA type: 32820
Unsupported ioctl: cmd=0x8946
tcpdump-arm: ens33: SIOCETHTOOL(ETHTOOL_GLINK) ioctl failed: Function not implemented
$ qemu-mips ./tcpdump-mips
Unknown host QEMU_IFLA type: 54
Unknown host QEMU_IFLA type: 32820
Unsupported ioctl: cmd=0x8946
tcpdump-mips: ens33: SIOCETHTOOL(ETHTOOL_GLINK) ioctl failed: Function not implemented
$ qemu-mipsel  ./tcpdump-mipsel 
Unknown host QEMU_IFLA type: 54
Unknown host QEMU_IFLA type: 32820
Unsupported ioctl: cmd=0x8946
tcpdump-mipsel: ens33: SIOCETHTOOL(ETHTOOL_GLINK) ioctl failed: Function not implemented
```