# 如何让 TPLINK-R479GP-AC v2.0 开启 V2ray
------
## 确定版本
 * 基于硬件版本 TL-R479GP-AC v2.0，已测试通过
 * 基于软件版本 1.1.4 Build 20200709 Rel.61296
 * 查看版本请在设置中[系统工具]->[设备管理]->[软件升级]中查看
----------------------------------------------------------------
## 获取ROOT权限
* R479GP系统基于Openwrt 14.07开发,在设置中[系统工具]->[诊断工具]->[故障诊断]->[开启诊断模式] 即可打开ssh通道,默认端口为`33400`,如果默认端口不对,请参考配置文件
* `ssh root@192.168.0.1 -p 33400` 即可连接上路由器
* root密码为LAN口mac地址 md5后的前8位
* 例子：`Mac下终端执行:echo -n "410EEC2A5E1E" | md5 `
* 输出结果:`c6c89e99b3bd00fc6229a14b1ebd2748` 前8位,即`c6c89e99`为root密码
* Linux下请把`md5`改为`md5sum`
* 如果无法登录,请参考以下链接获取root权限
* 如何备份配置文件以及如何添加用户:[https://www.eatm.app/archives/395.html]
----------------------------------------------------------------
## 基础信息
* CPU信息 `cat /proc/cpuinfo`
> system type		: Qualcomm Atheros QCA956X rev 0  
machine			: TP-LINK TL-R4299G 2.0  
processor		: 0  1.
cpu model		: MIPS 74Kc V5.0  
BogoMIPS		: 373.55  
wait instruction	: yes  
microsecond timers	: yes  
tlb_entries		: 32  
extra interrupt vector	: yes  
hardware watchpoint	: yes, count: 4, address/irw mask: [0x0000, 0x0ff8, > 0x0ff8, 0x0ff8]  
ASEs implemented	: mips16 dsp  
shadow register sets	: 1  
kscratch registers	: 0  
core			: 0  
VCED exceptions		: not available  
VCEI exceptions		: not available  

* Openwrt 版本 `cat /etc/openwrt_version`
>DISTRIB_ID="OpenWrt"  
DISTRIB_RELEASE="Barrier Breaker"  
DISTRIB_REVISION="r95548"  
DISTRIB_CODENAME="barrier_breaker"  
DISTRIB_TARGET="ar71xx/generic"  
DISTRIB_DESCRIPTION="OpenWrt Barrier Breaker 14.07"  
DISTRIB_TAINTS="no-all no-ipv6 busybox"  

* 该版本貌似没有编译FPU支持，所以软件只支持`softloat`
* mtd信息 `cat /proc/mtd`
> mtd0: 00030000 00010000 "factoryBoot"  
mtd1: 00008000 00008000 "factoryInfo"  
mtd2: 00008000 00008000 "art"  
mtd3: 00030000 00010000 "bootloader"  
mtd4: 00000600 00000600 "tpHead"  
mtd5: 0014fa00 00010000 "kernel"  
mtd6: 00ba0000 00010000 "rootfs"  
mtd7: 00200000 00010000 "rootfs_data"  
mtd8: 00020000 00010000 "log"  
mtd9: 01000000 00010000 "firmware"  

* 文件系统信息 `df -h`
> Filesystem                Size      Used Available Use% Mounted on  
rootfs                   61.8M    700.0K     61.1M   1% /  
/dev/root                 9.3M      9.3M         0 100% /rom  
tmpfs                    61.8M     11.9M     49.9M  19% /tmp  
root                     61.8M    700.0K     61.1M   1% /tmp/root  
overlayfs:/tmp/root      61.8M    700.0K     61.1M   1% /  
/dev/mtdblock7            2.0M    724.0K      1.3M  35% /tmp/userconfig  
tmpfs                   512.0K         0    512.0K   0% /dev  

* 其中`/dev/mtdblock7            2.0M    724.0K      1.3M  35% /tmp/userconfig` 是 `唯一可写区`,也就是用户配置存放的地方，只有可怜的2M
* 对openwrt研究不深,不知道怎么把该分区在不重新编译固件的情况下加大,目前采用折中的办法是每次重启后,下载v2ray客户端到/tmp/v2ray目录下
----------------------------------------------------------------

## 开启v2ray
* 把v2ray目录下的v2ray,config.json,编辑好后传到自己的服务器上,自己编译v2ray请记得加上`GOMIPS=softfloat`开启软浮点
* 我用的是别人编译好的v2ray客户端,不放心的可以自己编译一份
* config.json 自己配置好,我配置的是让路由器监听1080端口,我用sock5连上去
* 在线生成v2ray配置:[https://intmainreturn0.com/v2ray-config-gen]
* v2ray客户端可以放在内网,也可以放在外网,我用的是一台内网ios设备当文件服务器
* 编辑`/tmp/userconfig/etc/rc.local`文件,加入开机后的命令
```sh
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.

irq_balance()
{
    local irqnum=`cat /proc/interrupts | awk -F: '/eth/ {print $1}' | sed 's/[^0-9]//g'`

    [ "1" == `cat /proc/irq/${irqnum}/smp_affinity` ] && {
        local core_num=`cat /proc/cpuinfo | grep -w processor | awk 'END{print NR}'`
        local mask=`awk "BEGIN{f=lshift(1,$core_num)-1; print f}"`
        echo `printf %x $mask` > /proc/irq/${irqnum}/smp_affinity
    }
}
irq_balance
#上面是机器原来的命令,下面是自己加的命令
sleep 60 # 等待60秒让路由连上外网
mkdir /tmp/v2ray
cd /tmp/v2ray
wget http://192.168.0.199/v2ray # 下载v2ray到路由上
wget http://192.168.0.199/config.json #下载配置
chmod +x /tmp/v2ray/v2ray
/tmp/v2ray/v2ray -c /tmp/v2ray/config.json &

```
* 至此折腾结束,想让所有流量转发至v2ray上请自行配置iptables,接着往`rc.local`写命令即可

 
 
