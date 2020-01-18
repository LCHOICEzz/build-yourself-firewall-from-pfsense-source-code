# build-yourself-firewall-from-pfsense-source-code
# 感谢
国内有关自定义编译pfsense源码的资料太少了，多亏了github上的https://github.com/Augustin-FL/building-pfsense-iso-from-source， 在这里感谢下
Augustin-FL这位大牛，撰写教程并且耐心解答issue中的问题。

# 正文
Augustin-FL教程是关于2.5这个版本的pfsense，但是在 https://github.com/pfsense/pfsense 中是没有2.5这个版本的分支，因此如果完全照搬他教程的内容会失败，因此退一步，选择2.4.5这个版本，幸运的是FreeBSD-src和FreeBSD-ports都有2.4.5这个版本。
![Disk partition](https://github.com/LCHOICEzz/build-yourself-firewall-from-pfsense-source-code/blob/master/images/pfsense-branch-version.png)
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

# 配置
```
# Allow SSH using root, if you want it.
echo PermitRootLogin yes >> /etc/ssh/sshd_config
service sshd restart

# Required for configuring the server
pkg install -y pkg vim nano

# Required for installing and building ports
pkg install -y git nginx poudriere-devel mkfile rsync sudo

# Required for building kernel and iso
pkg install -y vmdktool curl qemu-user-static gtar xmlstarlet pkgconf openssl111

# Required for building iso
portsnap fetch extract

# not required but advised for building/monitoring/debugging
pkg install -y htop screen wget

# Only install this if your FreeBSD is a virtual machine
pkg install -y open-vm-tools
```

```
# pfSense_gui_branch represents the branch of pfSense GUI that will be compiled, with "RELENG" replaced by "v" : master for a development ISO, v2_4_5 for a stable ISO
# pfSense_port_branch represents the branch of FreeBSD ports that will be compiled, using the same replacement ("RELENG"=>"v") : devel for a development ISO, v2_4_5 for a stable ISO
# product_name represents the name of your product.

set pfSense_gui_branch=v2_4_5 # Replace with the version you want to build
set pfSense_port_branch=v2_4_5 # Replace with the version you want to build
set product_name=libreSense # Replace with your product name
set pfSense_gui_version=RELENG_2_4_5


cd /usr/local/www/nginx/
rm -rf *
mkdir -p packages
# PKG web server for core PKG repositories (pfSense-base, pfSense-rc, etc...)
ln -s /root/pfsense/tmp/${product_name}_${pfSense_gui_branch}_amd64-core/.latest packages/${product_name}_${pfSense_gui_branch}_amd64-core
# PKG web server for other PKG repositories
ln -s /usr/local/poudriere/data/packages/${product_name}_${pfSense_gui_branch}_amd64-${product_name}_${pfSense_port_branch} packages/${product_name}_${pfSense_gui_branch}_amd64-${product_name}_${pfSense_port_branch} 
# Web server for monitoring ports build
ln -s /usr/local/poudriere/data/logs/bulk/${product_name}_${pfSense_gui_branch}_amd64-${product_name}_${pfSense_port_branch}/latest poudriere

# Allow directory indexing, and configure nginx to start automatically on boot
sed -i '' 's+/usr/local/www/nginx;+/usr/local/www/nginx; autoindex on;+g' /usr/local/etc/nginx/nginx.conf
echo nginx_enable=\"YES\" >> /etc/rc.conf
service nginx restart
```
```
mkdir -p /root/sign/
cd /root/sign/
openssl genrsa -out repo.key 2048
chmod 0400 repo.key
openssl rsa -in repo.key -out repo.pub -pubout
printf "function: sha256\nfingerprint: `sha256 -q repo.pub`\n" > fingerprint
```
在/root/sign下创建一个sign.sh的脚本文件，内容为：
```
#!/bin/sh
read -t 2 sum
[ -z "$sum" ] && exit 1
echo SIGNATURE
echo -n $sum | openssl dgst -sign /root/sign/repo.key -sha256 -binary
echo
echo CERT
cat /root/sign/repo.pub
echo END
```
设置可执行权限
```
chmod +x /root/sign/sign.sh
```

```
cd /root
git clone https://github.com/你的github/pfsense.git
cd pfsense
git checkout ${pfSense_gui_version}

# PKG signing key
rm src/usr/local/share/${product_name}/keys/pkg/trusted/*
cp /root/sign/fingerprint src/usr/local/share/${product_name}/keys/pkg/trusted/fingerprint
```

在pfsense目录下创建build.conf文件
```
export PRODUCT_NAME="libreSense" # Replace with your product name
export FREEBSD_REPO_BASE=https://github.com/你的github/FreeBSD-src.git # Location of your FreeBSD sources repository
export POUDRIERE_PORTS_GIT_URL=https://github.com/你的github/FreeBSD-ports.git # Location your FreeBSD ports repository

export FREEBSD_BRANCH=RELENG_2_4_5 # Branch of FreeBSD sources to build
# The branch of FreeBSD ports to build is set automatically based on pfSense GUI branch.
# If you would like to build a specific branch of FreeBSD ports, the variable to set is POUDRIERE_PORTS_GIT_BRANCH



# Netgate support creation of staging builds (pre-dev, nonpublic version)
unset USE_PKG_REPO_STAGING # This disable staging build
# The kind of ISO that will be built (stable or development) is defined in src/etc/version in pfSense GUI repo

export DEFAULT_ARCH_LIST="amd64.amd64" # We only want to build an x64 ISO, we don't care of ARM versions

# Signing key
export PKG_REPO_SIGNING_COMMAND="/root/sign/sign.sh ${PKG_REPO_SIGN_KEY}"

# This command retrieves the IP address of the first network interface
export myIPAddress=$(ifconfig -a | grep inet | head -1 | cut -d ' ' -f 2)

export PKG_REPO_SERVER_DEVEL="pkg+http://${myIPAddress}/packages"
export PKG_REPO_SERVER_RELEASE="pkg+http://${myIPAddress}/packages"

export PKG_REPO_SERVER_STAGING="pkg+http://${myIPAddress}/packages" # We need to also specify this variable, because even
# if we don't build staging release some ports configuration is made for staging.
```

# 构建ISO
### 安装 Jails
执行命令 ```./build.sh --setup-poudriere```
### 构建 ports
执行命令 ```./build.sh --update-pkg-repo``` 
访问 http://ipOfYourServer/poudriere 即可看到进度
![Disk partition](
https://github.com/Augustin-FL/building-pfsense-iso-from-source/blob/master/images/poudriere_build.png?raw=true)
### 构建 kernel 以及创建 ISO
最后执行`./build.sh --skip-final-rsync iso`，即可在~pfsense/tmp/${product_name}/installer/下找到
执行`gzip -kd *.gz`即可解压缩，获得iso文件
最终在VMware安装即可.

