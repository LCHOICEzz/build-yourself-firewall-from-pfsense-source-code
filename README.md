# build-yourself-firewall-from-pfsense-source-code
# 感谢
国内有关自定义编译pfsense源码的资料太少了，多亏了github上的https://github.com/Augustin-FL/building-pfsense-iso-from-source， 在这里感谢下
Augustin-FL这位大牛，撰写教程并且耐心解答issue中的问题。

# 正文
Augustin-FL教程是关于2.5这个版本的pfsense，但是在 https://github.com/pfsense/pfsense 中是没有2.5这个版本的分支，因此如果完全照搬他教程的内容会失败，因此退一步，选择2.4.5这个版本，幸运的是FreeBSD-src和FreeBSD-ports都有2.4.5这个版本。
查询下pfSense不同版本对应的FreeBSD，发现pfSense2.4.5对应的FreeBSD是11.3这个版本，此时如果你选择FreeBSD11.3后续会编译失败，原因出在某个ports的版本需要FreeBSD 12这个版本，不然会出错，可能这个地方是pfsense的作者的一个失误。
![Disk partition](https://github.com/LCHOICEzz/build-yourself-firewall-from-pfsense-source-code/blob/master/images/Versions_of_pfSense_and_FreeBSD.png)
国内用户最好不要在自己的机器上编译，抛开机器性能不说，GFW这个东西会影响你的fetch，导致你fetch超时，最终超时导致你build失败，所以最好选择VPS,日本、新加坡等亚洲地区即可。这里推荐我使用的https://my.vultr.com/ ，按小时收费，编译的机器内存不能低于8G，不然也会失败。

### FreeBSD Source
从 https://github.com/pfsense/FreeBSD-src fork到你的github上，调整到RELENG_2_4_5这个分支
在`/release/conf/`目录下, 重命名`pfSense_src-env.conf`， `pfSense_src.conf` ，`pfSense_make.conf` 为 `libreSense_src-env.conf`，`libreSense_src.conf` 以及`libreSense_make.conf`
重命名`/sys/amd64/conf/pfSense` 为`/sys/amd64/conf/libreSense`
修改`/tools/tools/crypto/Makefile`，从`PROGS`命令中移除 `cryptokeytest`

### FreeBSD Ports
从 https://github.com/pfsense/FreeBSD-ports fork到你的github上，调整到RELENG_2_4_5这个分支
把`/sysutils/pfSense-upgrade/files/`目录下的2个文件 `pfSense-upgrade` 与 `pfsense-upgrade.wrapper` 重命名为 `libreSense-upgrade` 以及`libreSense-upgrade.wrapper`.

### pfSense GUI
从 https://github.com/pfsense/pfsense fork到你的github上，调整到RELENG_2_4_5这个分支
将`/tools/templates/pkg_repos/`目录下所有包含`pfSense`的文件都 重命名为 `libreSense` (比如`pfSense-repo.abi => libreSense-repo.abi`)
补充一条在Augustin教程中没有的：
将https://github.com/pfsense/pfsense/tree/master/tools/templates/pkg_repos 下所有-245.conf and -245.descr 的文件复制一份到 https://github.com/你的github/FreeBSD-ports/tree/RELENG_2_4_5/sysutils/pfSense-repo/files 下，不然build依然会失败。

# 搭建编译环境
安装FreeBSD时候，分区记得选择ZFS
![Disk partition](
https://github.com/Augustin-FL/building-pfsense-iso-from-source/blob/master/images/ZFS.png?raw=true)
