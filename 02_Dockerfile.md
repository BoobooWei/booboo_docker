# 02_Dockerfile

[TOC]

> 目标，将mysql故障检测程序打包成docker镜像，方便下次部署


##　构造Docker基础镜像

`mkimage-yum.sh`脚本就是我们用来构造Docker基础镜像的脚本。其工作原理解释如下：

```shell
#!/usr/bin/env bash
#
# Create a base CentOS Docker image.
#
# This script is useful on systems with yum installed (e.g., building
# a CentOS image on CentOS).  See contrib/mkimage-rinse.sh for a way
# to build CentOS images on other systems.

usage() {
    cat <<EOOPTS
$(basename $0) [OPTIONS] <name>
OPTIONS:
  -p "<packages>"  The list of packages to install in the container.
                   The default is blank.
  -g "<groups>"    The groups of packages to install in the container.
                   The default is "Core".
  -y <yumconf>     The path to the yum config to install packages from. The
                   default is /etc/yum.conf for Centos/RHEL and /etc/dnf/dnf.conf for Fedora
EOOPTS
    exit 1
}

# 参数 -p 设置安装包名列表
# 参数 -g 设置安装组名列表
# 参数 -y 设置yum配置文件位置
# 默认在本地yum源文件配置正常的情况下，什么参数都不需要修改
# 执行时需要给定创建的镜像名

# option defaults
yum_config=/etc/yum.conf
if [ -f /etc/dnf/dnf.conf ] && command -v dnf &> /dev/null; then
	yum_config=/etc/dnf/dnf.conf
	alias yum=dnf
fi
# 如果系统是fedora，并且使用的是dnf包管理工具，将使用dnf替换yum

install_groups="Core"
# 默认使用此脚本构建的Docker镜像仅包含核心包

while getopts ":y:p:g:h" opt; do
    case $opt in
        y)
            yum_config=$OPTARG
            ;;
        h)
            usage
            ;;
        p)
            install_packages="$OPTARG"
            ;;
        g)
            install_groups="$OPTARG"
            ;;
        \?)
            echo "Invalid option: -$OPTARG"
            usage
            ;;
    esac
done
shift $((OPTIND - 1))
name=$1

# 根据给定参数设置运行状态


if [[ -z $name ]]; then
    usage
fi

target=$(mktemp -d --tmpdir $(basename $0).XXXXXX)
# 设置临时目录存放docker镜像目录和文件

set -x
# 将下面的命令打印到屏幕

mkdir -m 755 "$target"/dev
mknod -m 600 "$target"/dev/console c 5 1
mknod -m 600 "$target"/dev/initctl p
mknod -m 666 "$target"/dev/full c 1 7
mknod -m 666 "$target"/dev/null c 1 3
mknod -m 666 "$target"/dev/ptmx c 5 2
mknod -m 666 "$target"/dev/random c 1 8
mknod -m 666 "$target"/dev/tty c 5 0
mknod -m 666 "$target"/dev/tty0 c 4 0
mknod -m 666 "$target"/dev/urandom c 1 9
mknod -m 666 "$target"/dev/zero c 1 5
# 创建系统运行所需的设备文件

# amazon linux yum will fail without vars set
# 如果是在amazon上构建，需要/etc/yum/vars目录
if [ -d /etc/yum/vars ]; then
	mkdir -p -m 755 "$target"/etc/yum
	cp -a /etc/yum/vars "$target"/etc/yum/
fi

if [[ -n "$install_groups" ]];
then
    yum -c "$yum_config" --installroot="$target" --releasever=/ --setopt=tsflags=nodocs \
        --setopt=group_package_types=mandatory -y groupinstall $install_groups
fi
# 使用yum安装软件包组，安装目录为前面设置的临时目录，不解开包中的文档以减少镜像空间


if [[ -n "$install_packages" ]];
then
    yum -c "$yum_config" --installroot="$target" --releasever=/ --setopt=tsflags=nodocs \
        --setopt=group_package_types=mandatory -y install $install_packages
fi

yum -c "$yum_config" --installroot="$target" -y clean all
# 清空yum缓冲以减小镜像空间


cat > "$target"/etc/sysconfig/network <<EOF
NETWORKING=yes
HOSTNAME=localhost.localdomain
EOF
# 初始化网络配置文件

# effectively: febootstrap-minimize --keep-zoneinfo --keep-rpmdb --keep-services "$target".
#  locales
rm -rf "$target"/usr/{{lib,share}/locale,{lib,lib64}/gconv,bin/localedef,sbin/build-locale-archive}
#  docs and man pages
rm -rf "$target"/usr/share/{man,doc,info,gnome/help}
#  cracklib
rm -rf "$target"/usr/share/cracklib
#  i18n
rm -rf "$target"/usr/share/i18n
#  yum cache
rm -rf "$target"/var/cache/yum
mkdir -p --mode=0755 "$target"/var/cache/yum
#  sln
rm -rf "$target"/sbin/sln
#  ldconfig
rm -rf "$target"/etc/ld.so.cache "$target"/var/cache/ldconfig
# 进一步清除不需要的文件，以减小镜像空间

mkdir -p --mode=0755 "$target"/var/cache/ldconfig
# 创建所需ldconfig目录


version=
for file in "$target"/etc/{redhat,system}-release
do
    if [ -r "$file" ]; then
        version="$(sed 's/^[^0-9\]*\([0-9.]\+\).*$/\1/' "$file")"
        break
    fi
done
# 根据当前系统设置镜像版本号


if [ -z "$version" ]; then
    echo >&2 "warning: cannot autodetect OS version, using '$name' as tag"
    version=$name
fi

tar --numeric-owner -c -C "$target" . | docker import - $name:$version

# 打包创建的镜像目录和文件，并导入本地docker

docker run -i -t --rm $name:$version /bin/bash -c 'echo success'
# 运行此镜像，验证镜像是否正常

rm -rf "$target"
# 删除临时目录
```

运行后可以看到新的docker镜像

```shell
Complete!
+ [[ -n '' ]]
+ yum -c /etc/yum.conf --installroot=/tmp/mkimage-yum.sh.kjRPUe -y clean all
Loaded plugins: fastestmirror, langpacks, product-id, search-disabled-repos, subscription-manager
This system is not registered to Red Hat Subscription Management. You can use subscription-manager to register.
Cleaning repos: base extras updates
Cleaning up everything
+ cat
+ rm -rf /tmp/mkimage-yum.sh.kjRPUe/usr/lib/locale /tmp/mkimage-yum.sh.kjRPUe/usr/share/locale /tmp/mkimage-yum.sh.kjRPUe/usr/lib/gconv /tmp/mkimage-yum.sh.kjRPUe/usr/lib64/gconv /tmp/mkimage-yum.sh.kjRPUe/usr/bin/localedef /tmp/mkimage-yum.sh.kjRPUe/usr/sbin/build-locale-archive
+ rm -rf /tmp/mkimage-yum.sh.kjRPUe/usr/share/man /tmp/mkimage-yum.sh.kjRPUe/usr/share/doc /tmp/mkimage-yum.sh.kjRPUe/usr/share/info /tmp/mkimage-yum.sh.kjRPUe/usr/share/gnome/help
+ rm -rf /tmp/mkimage-yum.sh.kjRPUe/usr/share/cracklib
+ rm -rf /tmp/mkimage-yum.sh.kjRPUe/usr/share/i18n
+ rm -rf /tmp/mkimage-yum.sh.kjRPUe/var/cache/yum
+ mkdir -p --mode=0755 /tmp/mkimage-yum.sh.kjRPUe/var/cache/yum
+ rm -rf /tmp/mkimage-yum.sh.kjRPUe/sbin/sln
+ rm -rf /tmp/mkimage-yum.sh.kjRPUe/etc/ld.so.cache /tmp/mkimage-yum.sh.kjRPUe/var/cache/ldconfig
+ mkdir -p --mode=0755 /tmp/mkimage-yum.sh.kjRPUe/var/cache/ldconfig
+ version=
+ for file in '"$target"/etc/{redhat,system}-release'
+ '[' -r /tmp/mkimage-yum.sh.kjRPUe/etc/redhat-release ']'
++ sed 's/^[^0-9\]*\([0-9.]\+\).*$/\1/' /tmp/mkimage-yum.sh.kjRPUe/etc/redhat-release
+ version=7.5.1804
+ break
+ '[' -z 7.5.1804 ']'
+ tar --numeric-owner -c -C /tmp/mkimage-yum.sh.kjRPUe .
+ docker import - rhel7.2:7.5.1804
sha256:60bf63c38109dba1b1accae6a3dee1c2b45d970963059034699d3e8ad501fecc
+ docker run -i -t --rm rhel7.2:7.5.1804 /bin/bash -c 'echo success'
success
+ rm -rf /tmp/mkimage-yum.sh.kjRPUe
[root@mastera toberoot]# ls
Dockerfile  mkimage-yum.sh
[root@mastera toberoot]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
rhel7.2             7.5.1804            60bf63c38109        About a minute ago   277MB
dao-2048            0.02                141c918abbbf        4 days ago           8.01MB
booboo-2048         latest              7929bcd70e47        2 years ago          8.01MB
dao-2048            0.01                7929bcd70e47        2 years ago          8.01MB
```

## 通过基础镜像构建程序镜像


[自建MySQL检测代码部署]!(https://confluence.jiagouyun.com/pages/viewpage.action?pageId=20521612)

服务搭建步骤：

1. python代码部署 [http://git.jiagouyun.com/weiyaping/mysqlapp.git]
2. python第三方包
3. Nginx软件安装
4. Nginx配置反向代理
5. gunicorn启动应用
6. Nginx启动服务


```shell
[root@mastera toberoot]# git clone http://git.jiagouyun.com/weiyaping/mysqlapp.git
[root@mastera toberoot]# ll
total 8
-rw-r--r-- 1 root root    0 Jun 10 18:33 Dockerfile
-rwxr-xr-x 1 root root 4714 Jun 10 18:53 mkimage-yum.sh
drwxr-xr-x 5 root root   92 Jun 10 19:16 mysqlapp
[root@mastera toberoot]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
rhel7.2             7.5.1804            60bf63c38109        2 hours ago         277MB
dao-2048            0.02                141c918abbbf        4 days ago          8.01MB
dao-2048            0.01                7929bcd70e47        2 years ago         8.01MB
booboo-2048         latest              7929bcd70e47        2 years ago         8.01MB
[root@mastera toberoot]# docker run --name nginx_booboo -i -t rhel7.2:7.5.1804 /bin/bash
[root@26759bd75e9b /]# yum repolist
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: mirrors.163.com
 * extras: mirrors.shu.edu.cn
 * updates: mirrors.cn99.com
base                                                                                                                                | 3.6 kB  00:00:00     
extras                                                                                                                              | 3.4 kB  00:00:00     
updates                                                                                                                             | 3.4 kB  00:00:00     
(1/4): base/7/x86_64/group_gz                                                                                                       | 166 kB  00:00:00     
(2/4): extras/7/x86_64/primary_db                                                                                                   | 147 kB  00:00:02     
(3/4): updates/7/x86_64/primary_db                                                                                                  | 2.0 MB  00:00:08     
(4/4): base/7/x86_64/primary_db                                                                                                     | 5.9 MB  00:00:17     
repo id                                                                  repo name                                                                   status
base/7/x86_64                                                            CentOS-7 - Base                                                             9911
extras/7/x86_64                                                          CentOS-7 - Extras                                                            305
updates/7/x86_64                                                         CentOS-7 - Updates                                                           654
repolist: 10870
[root@26759bd75e9b /]# yum install -y epel-release
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.163.com
 * extras: mirrors.shu.edu.cn
 * updates: mirrors.cn99.com
Resolving Dependencies
--> Running transaction check
---> Package epel-release.noarch 0:7-11 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===========================================================================================================================================================
 Package                                   Arch                                Version                           Repository                           Size
===========================================================================================================================================================
Installing:
 epel-release                              noarch                              7-11                              extras                               15 k

Transaction Summary
===========================================================================================================================================================
Install  1 Package

Total download size: 15 k
Installed size: 24 k
Downloading packages:
warning: /var/cache/yum/x86_64/7/extras/packages/epel-release-7-11.noarch.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Public key for epel-release-7-11.noarch.rpm is not installed
epel-release-7-11.noarch.rpm                                                                                                        |  15 kB  00:00:00     
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-5.1804.el7.centos.x86_64 (@name/$releasever)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : epel-release-7-11.noarch                                                                                                                1/1 
  Verifying  : epel-release-7-11.noarch                                                                                                                1/1 

Installed:
  epel-release.noarch 0:7-11                                                                                                                               

Complete!
[root@26759bd75e9b /]# yum install -y python-pip
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
epel/x86_64/metalink                                                                                                                | 5.8 kB  00:00:00     
 * base: mirrors.163.com
 * epel: mirrors.ustc.edu.cn
 * extras: mirrors.shu.edu.cn
 * updates: mirrors.cn99.com
epel                                                                                                                                | 3.2 kB  00:00:00     
(1/3): epel/x86_64/group_gz                                                                                                         |  88 kB  00:00:00     
(2/3): epel/x86_64/primary                                                                                                          | 3.5 MB  00:00:07     
(3/3): epel/x86_64/updateinfo                                                                                                       | 932 kB  00:00:11     
epel                                                                                                                                           12581/12581
Resolving Dependencies
--> Running transaction check
---> Package python2-pip.noarch 0:8.1.2-6.el7 will be installed
--> Processing Dependency: python-setuptools for package: python2-pip-8.1.2-6.el7.noarch
--> Running transaction check
---> Package python-setuptools.noarch 0:0.9.8-7.el7 will be installed
--> Processing Dependency: python-backports-ssl_match_hostname for package: python-setuptools-0.9.8-7.el7.noarch
--> Running transaction check
---> Package python-backports-ssl_match_hostname.noarch 0:3.5.0.1-1.el7 will be installed
--> Processing Dependency: python-ipaddress for package: python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch
--> Processing Dependency: python-backports for package: python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch
--> Running transaction check
---> Package python-backports.x86_64 0:1.0-8.el7 will be installed
---> Package python-ipaddress.noarch 0:1.0.16-2.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===========================================================================================================================================================
 Package                                                  Arch                        Version                              Repository                 Size
===========================================================================================================================================================
Installing:
 python2-pip                                              noarch                      8.1.2-6.el7                          epel                      1.7 M
Installing for dependencies:
 python-backports                                         x86_64                      1.0-8.el7                            base                      5.8 k
 python-backports-ssl_match_hostname                      noarch                      3.5.0.1-1.el7                        base                       13 k
 python-ipaddress                                         noarch                      1.0.16-2.el7                         base                       34 k
 python-setuptools                                        noarch                      0.9.8-7.el7                          base                      397 k

Transaction Summary
===========================================================================================================================================================
Install  1 Package (+4 Dependent packages)

Total download size: 2.1 M
Installed size: 9.4 M
Downloading packages:
(1/5): python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch.rpm                                                                 |  13 kB  00:00:00     
(2/5): python-backports-1.0-8.el7.x86_64.rpm                                                                                        | 5.8 kB  00:00:00     
(3/5): python-setuptools-0.9.8-7.el7.noarch.rpm                                                                                     | 397 kB  00:00:00     
(4/5): python-ipaddress-1.0.16-2.el7.noarch.rpm                                                                                     |  34 kB  00:00:00     
warning: /var/cache/yum/x86_64/7/epel/packages/python2-pip-8.1.2-6.el7.noarch.rpm: Header V3 RSA/SHA256 Signature, key ID 352c64e5: NOKEY MB  00:00:00 ETA 
Public key for python2-pip-8.1.2-6.el7.noarch.rpm is not installed
(5/5): python2-pip-8.1.2-6.el7.noarch.rpm                                                                                           | 1.7 MB  00:00:04     
-----------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                      489 kB/s | 2.1 MB  00:00:04     
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
Importing GPG key 0x352C64E5:
 Userid     : "Fedora EPEL (7) <epel@fedoraproject.org>"
 Fingerprint: 91e9 7d7c 4a5e 96f1 7f3e 888f 6a2f aea2 352c 64e5
 Package    : epel-release-7-11.noarch (@extras)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : python-backports-1.0-8.el7.x86_64                                                                                                       1/5 
  Installing : python-ipaddress-1.0.16-2.el7.noarch                                                                                                    2/5 
  Installing : python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch                                                                                3/5 
  Installing : python-setuptools-0.9.8-7.el7.noarch                                                                                                    4/5 
  Installing : python2-pip-8.1.2-6.el7.noarch                                                                                                          5/5 
  Verifying  : python-ipaddress-1.0.16-2.el7.noarch                                                                                                    1/5 
  Verifying  : python-setuptools-0.9.8-7.el7.noarch                                                                                                    2/5 
  Verifying  : python2-pip-8.1.2-6.el7.noarch                                                                                                          3/5 
  Verifying  : python-backports-1.0-8.el7.x86_64                                                                                                       4/5 
  Verifying  : python-backports-ssl_match_hostname-3.5.0.1-1.el7.noarch                                                                                5/5 

Installed:
  python2-pip.noarch 0:8.1.2-6.el7                                                                                                                         

Dependency Installed:
  python-backports.x86_64 0:1.0-8.el7         python-backports-ssl_match_hostname.noarch 0:3.5.0.1-1.el7      python-ipaddress.noarch 0:1.0.16-2.el7     
  python-setuptools.noarch 0:0.9.8-7.el7     

Complete!

[root@26759bd75e9b /]# pip install --upgrade pip
Collecting pip
  Downloading https://files.pythonhosted.org/packages/0f/74/ecd13431bcc456ed390b44c8a6e917c1820365cbebcb6a8974d1cd045ab4/pip-10.0.1-py2.py3-none-any.whl (1.3MB)
    100% |################################| 1.3MB 284kB/s 
Installing collected packages: pip
  Found existing installation: pip 8.1.2
    Uninstalling pip-8.1.2:
      Successfully uninstalled pip-8.1.2
Successfully installed pip-10.0.1


[root@26759bd75e9b /]# yum install -y python-devel mariadb-devel gcc
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.163.com
 * epel: mirrors.tongji.edu.cn
 * extras: mirrors.shu.edu.cn
 * updates: mirrors.cn99.com
Resolving Dependencies
--> Running transaction check
---> Package gcc.x86_64 0:4.8.5-28.el7_5.1 will be installed
--> Processing Dependency: libgomp = 4.8.5-28.el7_5.1 for package: gcc-4.8.5-28.el7_5.1.x86_64
--> Processing Dependency: cpp = 4.8.5-28.el7_5.1 for package: gcc-4.8.5-28.el7_5.1.x86_64
--> Processing Dependency: libgcc >= 4.8.5-28.el7_5.1 for package: gcc-4.8.5-28.el7_5.1.x86_64
--> Processing Dependency: glibc-devel >= 2.2.90-12 for package: gcc-4.8.5-28.el7_5.1.x86_64
--> Processing Dependency: libmpfr.so.4()(64bit) for package: gcc-4.8.5-28.el7_5.1.x86_64
--> Processing Dependency: libmpc.so.3()(64bit) for package: gcc-4.8.5-28.el7_5.1.x86_64
--> Processing Dependency: libgomp.so.1()(64bit) for package: gcc-4.8.5-28.el7_5.1.x86_64
---> Package mariadb-devel.x86_64 1:5.5.56-2.el7 will be installed
--> Processing Dependency: mariadb-libs(x86-64) = 1:5.5.56-2.el7 for package: 1:mariadb-devel-5.5.56-2.el7.x86_64
--> Processing Dependency: openssl-devel(x86-64) for package: 1:mariadb-devel-5.5.56-2.el7.x86_64
--> Processing Dependency: libmysqlclient.so.18()(64bit) for package: 1:mariadb-devel-5.5.56-2.el7.x86_64
---> Package python-devel.x86_64 0:2.7.5-68.el7 will be installed
--> Running transaction check
---> Package cpp.x86_64 0:4.8.5-28.el7_5.1 will be installed
---> Package glibc-devel.x86_64 0:2.17-222.el7 will be installed
--> Processing Dependency: glibc-headers = 2.17-222.el7 for package: glibc-devel-2.17-222.el7.x86_64
--> Processing Dependency: glibc-headers for package: glibc-devel-2.17-222.el7.x86_64
---> Package libgcc.x86_64 0:4.8.5-28.el7 will be updated
---> Package libgcc.x86_64 0:4.8.5-28.el7_5.1 will be an update
---> Package libgomp.x86_64 0:4.8.5-28.el7_5.1 will be installed
---> Package libmpc.x86_64 0:1.0.1-3.el7 will be installed
---> Package mariadb-libs.x86_64 1:5.5.56-2.el7 will be installed
---> Package mpfr.x86_64 0:3.1.1-4.el7 will be installed
---> Package openssl-devel.x86_64 1:1.0.2k-12.el7 will be installed
--> Processing Dependency: zlib-devel(x86-64) for package: 1:openssl-devel-1.0.2k-12.el7.x86_64
--> Processing Dependency: krb5-devel(x86-64) for package: 1:openssl-devel-1.0.2k-12.el7.x86_64
--> Running transaction check
---> Package glibc-headers.x86_64 0:2.17-222.el7 will be installed
--> Processing Dependency: kernel-headers >= 2.2.1 for package: glibc-headers-2.17-222.el7.x86_64
--> Processing Dependency: kernel-headers for package: glibc-headers-2.17-222.el7.x86_64
---> Package krb5-devel.x86_64 0:1.15.1-19.el7 will be installed
--> Processing Dependency: libkadm5(x86-64) = 1.15.1-19.el7 for package: krb5-devel-1.15.1-19.el7.x86_64
--> Processing Dependency: krb5-libs(x86-64) = 1.15.1-19.el7 for package: krb5-devel-1.15.1-19.el7.x86_64
--> Processing Dependency: libverto-devel for package: krb5-devel-1.15.1-19.el7.x86_64
--> Processing Dependency: libselinux-devel for package: krb5-devel-1.15.1-19.el7.x86_64
--> Processing Dependency: libcom_err-devel for package: krb5-devel-1.15.1-19.el7.x86_64
--> Processing Dependency: keyutils-libs-devel for package: krb5-devel-1.15.1-19.el7.x86_64
---> Package zlib-devel.x86_64 0:1.2.7-17.el7 will be installed
--> Running transaction check
---> Package kernel-headers.x86_64 0:3.10.0-862.3.2.el7 will be installed
---> Package keyutils-libs-devel.x86_64 0:1.5.8-3.el7 will be installed
---> Package krb5-libs.x86_64 0:1.15.1-18.el7 will be updated
---> Package krb5-libs.x86_64 0:1.15.1-19.el7 will be an update
---> Package libcom_err-devel.x86_64 0:1.42.9-12.el7_5 will be installed
--> Processing Dependency: libcom_err(x86-64) = 1.42.9-12.el7_5 for package: libcom_err-devel-1.42.9-12.el7_5.x86_64
---> Package libkadm5.x86_64 0:1.15.1-19.el7 will be installed
---> Package libselinux-devel.x86_64 0:2.5-12.el7 will be installed
--> Processing Dependency: libsepol-devel(x86-64) >= 2.5-6 for package: libselinux-devel-2.5-12.el7.x86_64
--> Processing Dependency: pkgconfig(libsepol) for package: libselinux-devel-2.5-12.el7.x86_64
--> Processing Dependency: pkgconfig(libpcre) for package: libselinux-devel-2.5-12.el7.x86_64
---> Package libverto-devel.x86_64 0:0.2.5-4.el7 will be installed
--> Running transaction check
---> Package libcom_err.x86_64 0:1.42.9-11.el7 will be updated
--> Processing Dependency: libcom_err(x86-64) = 1.42.9-11.el7 for package: libss-1.42.9-11.el7.x86_64
--> Processing Dependency: libcom_err(x86-64) = 1.42.9-11.el7 for package: e2fsprogs-libs-1.42.9-11.el7.x86_64
--> Processing Dependency: libcom_err(x86-64) = 1.42.9-11.el7 for package: e2fsprogs-1.42.9-11.el7.x86_64
---> Package libcom_err.x86_64 0:1.42.9-12.el7_5 will be an update
---> Package libsepol-devel.x86_64 0:2.5-8.1.el7 will be installed
---> Package pcre-devel.x86_64 0:8.32-17.el7 will be installed
--> Running transaction check
---> Package e2fsprogs.x86_64 0:1.42.9-11.el7 will be updated
---> Package e2fsprogs.x86_64 0:1.42.9-12.el7_5 will be an update
---> Package e2fsprogs-libs.x86_64 0:1.42.9-11.el7 will be updated
---> Package e2fsprogs-libs.x86_64 0:1.42.9-12.el7_5 will be an update
---> Package libss.x86_64 0:1.42.9-11.el7 will be updated
---> Package libss.x86_64 0:1.42.9-12.el7_5 will be an update
--> Finished Dependency Resolution

Dependencies Resolved

===========================================================================================================================================================
 Package                                    Arch                          Version                                     Repository                      Size
===========================================================================================================================================================
Installing:
 gcc                                        x86_64                        4.8.5-28.el7_5.1                            updates                         16 M
 mariadb-devel                              x86_64                        1:5.5.56-2.el7                              base                           752 k
 python-devel                               x86_64                        2.7.5-68.el7                                base                           397 k
Installing for dependencies:
 cpp                                        x86_64                        4.8.5-28.el7_5.1                            updates                        5.9 M
 glibc-devel                                x86_64                        2.17-222.el7                                base                           1.1 M
 glibc-headers                              x86_64                        2.17-222.el7                                base                           678 k
 kernel-headers                             x86_64                        3.10.0-862.3.2.el7                          updates                        7.1 M
 keyutils-libs-devel                        x86_64                        1.5.8-3.el7                                 base                            37 k
 krb5-devel                                 x86_64                        1.15.1-19.el7                               updates                        269 k
 libcom_err-devel                           x86_64                        1.42.9-12.el7_5                             updates                         31 k
 libgomp                                    x86_64                        4.8.5-28.el7_5.1                            updates                        156 k
 libkadm5                                   x86_64                        1.15.1-19.el7                               updates                        175 k
 libmpc                                     x86_64                        1.0.1-3.el7                                 base                            51 k
 libselinux-devel                           x86_64                        2.5-12.el7                                  base                           186 k
 libsepol-devel                             x86_64                        2.5-8.1.el7                                 base                            77 k
 libverto-devel                             x86_64                        0.2.5-4.el7                                 base                            12 k
 mariadb-libs                               x86_64                        1:5.5.56-2.el7                              base                           757 k
 mpfr                                       x86_64                        3.1.1-4.el7                                 base                           203 k
 openssl-devel                              x86_64                        1:1.0.2k-12.el7                             base                           1.5 M
 pcre-devel                                 x86_64                        8.32-17.el7                                 base                           480 k
 zlib-devel                                 x86_64                        1.2.7-17.el7                                base                            50 k
Updating for dependencies:
 e2fsprogs                                  x86_64                        1.42.9-12.el7_5                             updates                        699 k
 e2fsprogs-libs                             x86_64                        1.42.9-12.el7_5                             updates                        167 k
 krb5-libs                                  x86_64                        1.15.1-19.el7                               updates                        747 k
 libcom_err                                 x86_64                        1.42.9-12.el7_5                             updates                         41 k
 libgcc                                     x86_64                        4.8.5-28.el7_5.1                            updates                        101 k
 libss                                      x86_64                        1.42.9-12.el7_5                             updates                         45 k

Transaction Summary
===========================================================================================================================================================
Install  3 Packages (+18 Dependent packages)
Upgrade             (  6 Dependent packages)

Total download size: 38 M
Downloading packages:
Delta RPMs disabled because /usr/bin/applydeltarpm not installed.
(1/27): e2fsprogs-1.42.9-12.el7_5.x86_64.rpm                                                                                        | 699 kB  00:00:01     
(2/27): glibc-devel-2.17-222.el7.x86_64.rpm                                                                                         | 1.1 MB  00:00:05     
(3/27): e2fsprogs-libs-1.42.9-12.el7_5.x86_64.rpm                                                                                   | 167 kB  00:00:09     
(4/27): keyutils-libs-devel-1.5.8-3.el7.x86_64.rpm                                                                                  |  37 kB  00:00:01     
(5/27): glibc-headers-2.17-222.el7.x86_64.rpm                                                                                       | 678 kB  00:00:15     
(6/27): krb5-devel-1.15.1-19.el7.x86_64.rpm                                                                                         | 269 kB  00:00:10     
(7/27): libcom_err-1.42.9-12.el7_5.x86_64.rpm                                                                                       |  41 kB  00:00:01     
(8/27): libcom_err-devel-1.42.9-12.el7_5.x86_64.rpm                                                                                 |  31 kB  00:00:01     
(9/27): libgcc-4.8.5-28.el7_5.1.x86_64.rpm                                                                                          | 101 kB  00:00:05     
(10/27): krb5-libs-1.15.1-19.el7.x86_64.rpm                                                                                         | 747 kB  00:00:13     
(11/27): libkadm5-1.15.1-19.el7.x86_64.rpm                                                                                          | 175 kB  00:00:01     
(12/27): kernel-headers-3.10.0-862.3.2.el7.x86_64.rpm                                                                               | 7.1 MB  00:00:26     
(13/27): libgomp-4.8.5-28.el7_5.1.x86_64.rpm                                                                                        | 156 kB  00:00:04     
(14/27): libselinux-devel-2.5-12.el7.x86_64.rpm                                                                                     | 186 kB  00:00:01     
(15/27): libsepol-devel-2.5-8.1.el7.x86_64.rpm                                                                                      |  77 kB  00:00:00     
(16/27): libverto-devel-0.2.5-4.el7.x86_64.rpm                                                                                      |  12 kB  00:00:00     
(17/27): libss-1.42.9-12.el7_5.x86_64.rpm                                                                                           |  45 kB  00:00:01     
(18/27): libmpc-1.0.1-3.el7.x86_64.rpm                                                                                              |  51 kB  00:00:04     
(19/27): mpfr-3.1.1-4.el7.x86_64.rpm                                                                                                | 203 kB  00:00:03     
(20/27): mariadb-libs-5.5.56-2.el7.x86_64.rpm                                                                                       | 757 kB  00:00:08     
(21/27): pcre-devel-8.32-17.el7.x86_64.rpm                                                                                          | 480 kB  00:00:04     
(22/27): python-devel-2.7.5-68.el7.x86_64.rpm                                                                                       | 397 kB  00:00:03     
(23/27): zlib-devel-1.2.7-17.el7.x86_64.rpm                                                                                         |  50 kB  00:00:00     
(24/27): openssl-devel-1.0.2k-12.el7.x86_64.rpm                                                                                     | 1.5 MB  00:00:15     
(25/27): cpp-4.8.5-28.el7_5.1.x86_64.rpm                                                                                            | 5.9 MB  00:01:12     
(26/27): mariadb-devel-5.5.56-2.el7.x86_64.rpm                                                                                      | 752 kB  00:00:43     
(27/27): gcc-4.8.5-28.el7_5.1.x86_64.rpm                                                                                            |  16 MB  00:01:25     
-----------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                      454 kB/s |  38 MB  00:01:25     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : libcom_err-1.42.9-12.el7_5.x86_64                                                                                                      1/33 
  Installing : mpfr-3.1.1-4.el7.x86_64                                                                                                                2/33 
  Installing : libmpc-1.0.1-3.el7.x86_64                                                                                                              3/33 
  Updating   : krb5-libs-1.15.1-19.el7.x86_64                                                                                                         4/33 
  Installing : libkadm5-1.15.1-19.el7.x86_64                                                                                                          5/33 
  Installing : cpp-4.8.5-28.el7_5.1.x86_64                                                                                                            6/33 
  Updating   : libss-1.42.9-12.el7_5.x86_64                                                                                                           7/33 
  Installing : libcom_err-devel-1.42.9-12.el7_5.x86_64                                                                                                8/33 
  Updating   : e2fsprogs-libs-1.42.9-12.el7_5.x86_64                                                                                                  9/33 
  Installing : kernel-headers-3.10.0-862.3.2.el7.x86_64                                                                                              10/33 
  Installing : glibc-headers-2.17-222.el7.x86_64                                                                                                     11/33 
  Installing : glibc-devel-2.17-222.el7.x86_64                                                                                                       12/33 
  Installing : libsepol-devel-2.5-8.1.el7.x86_64                                                                                                     13/33 
  Installing : 1:mariadb-libs-5.5.56-2.el7.x86_64                                                                                                    14/33 
  Updating   : libgcc-4.8.5-28.el7_5.1.x86_64                                                                                                        15/33 
  Installing : libverto-devel-0.2.5-4.el7.x86_64                                                                                                     16/33 
  Installing : libgomp-4.8.5-28.el7_5.1.x86_64                                                                                                       17/33 
  Installing : pcre-devel-8.32-17.el7.x86_64                                                                                                         18/33 
  Installing : libselinux-devel-2.5-12.el7.x86_64                                                                                                    19/33 
  Installing : keyutils-libs-devel-1.5.8-3.el7.x86_64                                                                                                20/33 
  Installing : krb5-devel-1.15.1-19.el7.x86_64                                                                                                       21/33 
  Installing : zlib-devel-1.2.7-17.el7.x86_64                                                                                                        22/33 
  Installing : 1:openssl-devel-1.0.2k-12.el7.x86_64                                                                                                  23/33 
  Installing : 1:mariadb-devel-5.5.56-2.el7.x86_64                                                                                                   24/33 
  Installing : gcc-4.8.5-28.el7_5.1.x86_64                                                                                                           25/33 
  Updating   : e2fsprogs-1.42.9-12.el7_5.x86_64                                                                                                      26/33 
  Installing : python-devel-2.7.5-68.el7.x86_64                                                                                                      27/33 
  Cleanup    : e2fsprogs-1.42.9-11.el7.x86_64                                                                                                        28/33 
  Cleanup    : e2fsprogs-libs-1.42.9-11.el7.x86_64                                                                                                   29/33 
  Cleanup    : libss-1.42.9-11.el7.x86_64                                                                                                            30/33 
  Cleanup    : krb5-libs-1.15.1-18.el7.x86_64                                                                                                        31/33 
  Cleanup    : libcom_err-1.42.9-11.el7.x86_64                                                                                                       32/33 
  Cleanup    : libgcc-4.8.5-28.el7.x86_64                                                                                                            33/33 
  Verifying  : krb5-devel-1.15.1-19.el7.x86_64                                                                                                        1/33 
  Verifying  : zlib-devel-1.2.7-17.el7.x86_64                                                                                                         2/33 
  Verifying  : keyutils-libs-devel-1.5.8-3.el7.x86_64                                                                                                 3/33 
  Verifying  : libss-1.42.9-12.el7_5.x86_64                                                                                                           4/33 
  Verifying  : 1:mariadb-devel-5.5.56-2.el7.x86_64                                                                                                    5/33 
  Verifying  : glibc-devel-2.17-222.el7.x86_64                                                                                                        6/33 
  Verifying  : pcre-devel-8.32-17.el7.x86_64                                                                                                          7/33 
  Verifying  : glibc-headers-2.17-222.el7.x86_64                                                                                                      8/33 
  Verifying  : libgomp-4.8.5-28.el7_5.1.x86_64                                                                                                        9/33 
  Verifying  : 1:openssl-devel-1.0.2k-12.el7.x86_64                                                                                                  10/33 
  Verifying  : libverto-devel-0.2.5-4.el7.x86_64                                                                                                     11/33 
  Verifying  : libselinux-devel-2.5-12.el7.x86_64                                                                                                    12/33 
  Verifying  : gcc-4.8.5-28.el7_5.1.x86_64                                                                                                           13/33 
  Verifying  : libcom_err-devel-1.42.9-12.el7_5.x86_64                                                                                               14/33 
  Verifying  : e2fsprogs-1.42.9-12.el7_5.x86_64                                                                                                      15/33 
  Verifying  : e2fsprogs-libs-1.42.9-12.el7_5.x86_64                                                                                                 16/33 
  Verifying  : libgcc-4.8.5-28.el7_5.1.x86_64                                                                                                        17/33 
  Verifying  : cpp-4.8.5-28.el7_5.1.x86_64                                                                                                           18/33 
  Verifying  : python-devel-2.7.5-68.el7.x86_64                                                                                                      19/33 
  Verifying  : libmpc-1.0.1-3.el7.x86_64                                                                                                             20/33 
  Verifying  : krb5-libs-1.15.1-19.el7.x86_64                                                                                                        21/33 
  Verifying  : libcom_err-1.42.9-12.el7_5.x86_64                                                                                                     22/33 
  Verifying  : mpfr-3.1.1-4.el7.x86_64                                                                                                               23/33 
  Verifying  : 1:mariadb-libs-5.5.56-2.el7.x86_64                                                                                                    24/33 
  Verifying  : libkadm5-1.15.1-19.el7.x86_64                                                                                                         25/33 
  Verifying  : libsepol-devel-2.5-8.1.el7.x86_64                                                                                                     26/33 
  Verifying  : kernel-headers-3.10.0-862.3.2.el7.x86_64                                                                                              27/33 
  Verifying  : libcom_err-1.42.9-11.el7.x86_64                                                                                                       28/33 
  Verifying  : libgcc-4.8.5-28.el7.x86_64                                                                                                            29/33 
  Verifying  : libss-1.42.9-11.el7.x86_64                                                                                                            30/33 
  Verifying  : e2fsprogs-libs-1.42.9-11.el7.x86_64                                                                                                   31/33 
  Verifying  : krb5-libs-1.15.1-18.el7.x86_64                                                                                                        32/33 
  Verifying  : e2fsprogs-1.42.9-11.el7.x86_64                                                                                                        33/33 

Installed:
  gcc.x86_64 0:4.8.5-28.el7_5.1                  mariadb-devel.x86_64 1:5.5.56-2.el7                  python-devel.x86_64 0:2.7.5-68.el7                 

Dependency Installed:
  cpp.x86_64 0:4.8.5-28.el7_5.1                         glibc-devel.x86_64 0:2.17-222.el7                   glibc-headers.x86_64 0:2.17-222.el7           
  kernel-headers.x86_64 0:3.10.0-862.3.2.el7            keyutils-libs-devel.x86_64 0:1.5.8-3.el7            krb5-devel.x86_64 0:1.15.1-19.el7             
  libcom_err-devel.x86_64 0:1.42.9-12.el7_5             libgomp.x86_64 0:4.8.5-28.el7_5.1                   libkadm5.x86_64 0:1.15.1-19.el7               
  libmpc.x86_64 0:1.0.1-3.el7                           libselinux-devel.x86_64 0:2.5-12.el7                libsepol-devel.x86_64 0:2.5-8.1.el7           
  libverto-devel.x86_64 0:0.2.5-4.el7                   mariadb-libs.x86_64 1:5.5.56-2.el7                  mpfr.x86_64 0:3.1.1-4.el7                     
  openssl-devel.x86_64 1:1.0.2k-12.el7                  pcre-devel.x86_64 0:8.32-17.el7                     zlib-devel.x86_64 0:1.2.7-17.el7              

Dependency Updated:
  e2fsprogs.x86_64 0:1.42.9-12.el7_5   e2fsprogs-libs.x86_64 0:1.42.9-12.el7_5   krb5-libs.x86_64 0:1.15.1-19.el7   libcom_err.x86_64 0:1.42.9-12.el7_5  
  libgcc.x86_64 0:4.8.5-28.el7_5.1     libss.x86_64 0:1.42.9-12.el7_5           

Complete!

[root@26759bd75e9b /]# pip install setuptools
Requirement already satisfied: setuptools in /usr/lib/python2.7/site-packages (0.9.8)
[root@26759bd75e9b /]# pip install flask
Collecting flask
  Downloading https://files.pythonhosted.org/packages/7f/e7/08578774ed4536d3242b14dacb4696386634607af824ea997202cd0edb4b/Flask-1.0.2-py2.py3-none-any.whl (91kB)
    100% |################################| 92kB 792kB/s 
Collecting Werkzeug>=0.14 (from flask)
  Downloading https://files.pythonhosted.org/packages/20/c4/12e3e56473e52375aa29c4764e70d1b8f3efa6682bef8d0aae04fe335243/Werkzeug-0.14.1-py2.py3-none-any.whl (322kB)
    100% |################################| 327kB 4.1MB/s 
Collecting click>=5.1 (from flask)
  Downloading https://files.pythonhosted.org/packages/34/c1/8806f99713ddb993c5366c362b2f908f18269f8d792aff1abfd700775a77/click-6.7-py2.py3-none-any.whl (71kB)
    100% |################################| 71kB 2.5MB/s 
Collecting itsdangerous>=0.24 (from flask)
  Downloading https://files.pythonhosted.org/packages/dc/b4/a60bcdba945c00f6d608d8975131ab3f25b22f2bcfe1dab221165194b2d4/itsdangerous-0.24.tar.gz (46kB)
    100% |################################| 51kB 6.0MB/s 
Collecting Jinja2>=2.10 (from flask)
  Downloading https://files.pythonhosted.org/packages/7f/ff/ae64bacdfc95f27a016a7bed8e8686763ba4d277a78ca76f32659220a731/Jinja2-2.10-py2.py3-none-any.whl (126kB)
    100% |################################| 133kB 4.5MB/s 
Collecting MarkupSafe>=0.23 (from Jinja2>=2.10->flask)
  Downloading https://files.pythonhosted.org/packages/4d/de/32d741db316d8fdb7680822dd37001ef7a448255de9699ab4bfcbdf4172b/MarkupSafe-1.0.tar.gz
Installing collected packages: Werkzeug, click, itsdangerous, MarkupSafe, Jinja2, flask
  Running setup.py install for itsdangerous ... done
  Running setup.py install for MarkupSafe ... done
Successfully installed Jinja2-2.10 MarkupSafe-1.0 Werkzeug-0.14.1 click-6.7 flask-1.0.2 itsdangerous-0.24
[root@26759bd75e9b /]# pip install paramiko
Collecting paramiko
  Downloading https://files.pythonhosted.org/packages/3e/db/cb7b6656e0e7387637ce850689084dc0b94b44df31cc52e5fc5c2c4fd2c1/paramiko-2.4.1-py2.py3-none-any.whl (194kB)
    100% |################################| 194kB 1.2MB/s 
Collecting cryptography>=1.5 (from paramiko)
  Downloading https://files.pythonhosted.org/packages/dd/c2/3a5bfefb25690725824ade71e6b65449f0a9f4b29702cce10560f786ebf6/cryptography-2.2.2-cp27-cp27mu-manylinux1_x86_64.whl (2.2MB)
    100% |################################| 2.2MB 577kB/s 
Collecting pynacl>=1.0.1 (from paramiko)
  Downloading https://files.pythonhosted.org/packages/80/3d/d709b9fbd69e21dd3a4d34eb690c5484094699e03b7447bc7eb173cfd7b6/PyNaCl-1.2.1-cp27-cp27mu-manylinux1_x86_64.whl (696kB)
    100% |################################| 706kB 635kB/s 
Collecting pyasn1>=0.1.7 (from paramiko)
  Downloading https://files.pythonhosted.org/packages/a0/70/2c27740f08e477499ce19eefe05dbcae6f19fdc49e9e82ce4768be0643b9/pyasn1-0.4.3-py2.py3-none-any.whl (72kB)
    100% |################################| 81kB 739kB/s 
Collecting bcrypt>=3.1.3 (from paramiko)
  Downloading https://files.pythonhosted.org/packages/2e/5a/2abeae20ce294fe6bf63da0e0b5a885c788e1360bbd124edcc0429678a59/bcrypt-3.1.4-cp27-cp27mu-manylinux1_x86_64.whl (57kB)
    100% |################################| 61kB 713kB/s 
Collecting idna>=2.1 (from cryptography>=1.5->paramiko)
  Downloading https://files.pythonhosted.org/packages/4b/2a/0276479a4b3caeb8a8c1af2f8e4355746a97fab05a372e4a2c6a6b876165/idna-2.7-py2.py3-none-any.whl (58kB)
    100% |################################| 61kB 830kB/s 
Collecting cffi>=1.7; platform_python_implementation != "PyPy" (from cryptography>=1.5->paramiko)
  Downloading https://files.pythonhosted.org/packages/14/dd/3e7a1e1280e7d767bd3fa15791759c91ec19058ebe31217fe66f3e9a8c49/cffi-1.11.5-cp27-cp27mu-manylinux1_x86_64.whl (407kB)
    100% |################################| 409kB 785kB/s 
Collecting enum34; python_version < "3" (from cryptography>=1.5->paramiko)
  Downloading https://files.pythonhosted.org/packages/c5/db/e56e6b4bbac7c4a06de1c50de6fe1ef3810018ae11732a50f15f62c7d050/enum34-1.1.6-py2-none-any.whl
Collecting six>=1.4.1 (from cryptography>=1.5->paramiko)
  Downloading https://files.pythonhosted.org/packages/67/4b/141a581104b1f6397bfa78ac9d43d8ad29a7ca43ea90a2d863fe3056e86a/six-1.11.0-py2.py3-none-any.whl
Collecting asn1crypto>=0.21.0 (from cryptography>=1.5->paramiko)
  Downloading https://files.pythonhosted.org/packages/ea/cd/35485615f45f30a510576f1a56d1e0a7ad7bd8ab5ed7cdc600ef7cd06222/asn1crypto-0.24.0-py2.py3-none-any.whl (101kB)
    100% |################################| 102kB 856kB/s 
Requirement already satisfied: ipaddress; python_version < "3" in /usr/lib/python2.7/site-packages (from cryptography>=1.5->paramiko) (1.0.16)
Collecting pycparser (from cffi>=1.7; platform_python_implementation != "PyPy"->cryptography>=1.5->paramiko)
  Downloading https://files.pythonhosted.org/packages/8c/2d/aad7f16146f4197a11f8e91fb81df177adcc2073d36a17b1491fd09df6ed/pycparser-2.18.tar.gz (245kB)
    100% |################################| 256kB 1.1MB/s 
Installing collected packages: idna, pycparser, cffi, enum34, six, asn1crypto, cryptography, pynacl, pyasn1, bcrypt, paramiko
  Running setup.py install for pycparser ... done
Successfully installed asn1crypto-0.24.0 bcrypt-3.1.4 cffi-1.11.5 cryptography-2.2.2 enum34-1.1.6 idna-2.7 paramiko-2.4.1 pyasn1-0.4.3 pycparser-2.18 pynacl-1.2.1 six-1.11.0
[root@26759bd75e9b /]# pip install prettytable
Collecting prettytable
  Downloading https://files.pythonhosted.org/packages/ef/30/4b0746848746ed5941f052479e7c23d2b56d174b82f4fd34a25e389831f5/prettytable-0.7.2.tar.bz2
Installing collected packages: prettytable
  Running setup.py install for prettytable ... done
Successfully installed prettytable-0.7.2
[root@26759bd75e9b /]# pip install MySQL-python==1.2.5
Collecting MySQL-python==1.2.5
  Downloading https://files.pythonhosted.org/packages/a5/e9/51b544da85a36a68debe7a7091f068d802fc515a3a202652828c73453cad/MySQL-python-1.2.5.zip (108kB)
    100% |################################| 112kB 659kB/s 
Installing collected packages: MySQL-python
  Running setup.py install for MySQL-python ... done
Successfully installed MySQL-python-1.2.5
[root@26759bd75e9b /]# pip install gunicorn
Collecting gunicorn
  Downloading https://files.pythonhosted.org/packages/55/cb/09fe80bddf30be86abfc06ccb1154f97d6c64bb87111de066a5fc9ccb937/gunicorn-19.8.1-py2.py3-none-any.whl (112kB)
    100% |################################| 122kB 1.1MB/s 
Installing collected packages: gunicorn
Successfully installed gunicorn-19.8.1
[root@26759bd75e9b /]# pip install psutil
Collecting psutil
  Downloading https://files.pythonhosted.org/packages/51/9e/0f8f5423ce28c9109807024f7bdde776ed0b1161de20b408875de7e030c3/psutil-5.4.6.tar.gz (418kB)
    100% |################################| 419kB 3.2MB/s 
Installing collected packages: psutil
  Running setup.py install for psutil ... done
Successfully installed psutil-5.4.6
[root@26759bd75e9b /]# pip install xlsxwriter
Collecting xlsxwriter
  Downloading https://files.pythonhosted.org/packages/33/50/136b801d106fcebb2428a764e5c599e020d8227a3623db078e05eb4793a5/XlsxWriter-1.0.5-py2.py3-none-any.whl (142kB)
    100% |################################| 143kB 1.5MB/s 
Installing collected packages: xlsxwriter
Successfully installed xlsxwriter-1.0.5
[root@26759bd75e9b mysqlapp]# pip install requests
Collecting requests
  Downloading https://files.pythonhosted.org/packages/49/df/50aa1999ab9bde74656c2919d9c0c085fd2b3775fd3eca826012bef76d8c/requests-2.18.4-py2.py3-none-any.whl (88kB)
    100% |################################| 92kB 721kB/s 
Collecting urllib3<1.23,>=1.21.1 (from requests)
  Downloading https://files.pythonhosted.org/packages/63/cb/6965947c13a94236f6d4b8223e21beb4d576dc72e8130bd7880f600839b8/urllib3-1.22-py2.py3-none-any.whl (132kB)
    100% |################################| 133kB 1.2MB/s 
Collecting idna<2.7,>=2.5 (from requests)
  Downloading https://files.pythonhosted.org/packages/27/cc/6dd9a3869f15c2edfab863b992838277279ce92663d334df9ecf5106f5c6/idna-2.6-py2.py3-none-any.whl (56kB)
    100% |################################| 61kB 2.1MB/s 
Collecting chardet<3.1.0,>=3.0.2 (from requests)
  Downloading https://files.pythonhosted.org/packages/bc/a9/01ffebfb562e4274b6487b4bb1ddec7ca55ec7510b22e4c51f14098443b8/chardet-3.0.4-py2.py3-none-any.whl (133kB)
    100% |################################| 143kB 1.3MB/s 
Collecting certifi>=2017.4.17 (from requests)
  Downloading https://files.pythonhosted.org/packages/7c/e6/92ad559b7192d846975fc916b65f667c7b8c3a32bea7372340bfe9a15fa5/certifi-2018.4.16-py2.py3-none-any.whl (150kB)
    100% |################################| 153kB 3.2MB/s 
Installing collected packages: urllib3, idna, chardet, certifi, requests
  Found existing installation: idna 2.7
    Uninstalling idna-2.7:
      Successfully uninstalled idna-2.7
Successfully installed certifi-2018.4.16 chardet-3.0.4 idna-2.6 requests-2.18.4 urllib3-1.22



[root@26759bd75e9b /]# 
[root@26759bd75e9b /]# pip install setuptools
Requirement already satisfied: setuptools in /usr/lib/python2.7/site-packages (0.9.8)
[root@26759bd75e9b /]# pip install flask
Requirement already satisfied: flask in /usr/lib64/python2.7/site-packages (1.0.2)
Requirement already satisfied: Werkzeug>=0.14 in /usr/lib/python2.7/site-packages (from flask) (0.14.1)
Requirement already satisfied: click>=5.1 in /usr/lib/python2.7/site-packages (from flask) (6.7)
Requirement already satisfied: itsdangerous>=0.24 in /usr/lib/python2.7/site-packages (from flask) (0.24)
Requirement already satisfied: Jinja2>=2.10 in /usr/lib/python2.7/site-packages (from flask) (2.10)
Requirement already satisfied: MarkupSafe>=0.23 in /usr/lib64/python2.7/site-packages (from Jinja2>=2.10->flask) (1.0)
[root@26759bd75e9b /]# pip install paramiko
Requirement already satisfied: paramiko in /usr/lib/python2.7/site-packages (2.4.1)
Requirement already satisfied: cryptography>=1.5 in /usr/lib64/python2.7/site-packages (from paramiko) (2.2.2)
Requirement already satisfied: pynacl>=1.0.1 in /usr/lib64/python2.7/site-packages (from paramiko) (1.2.1)
Requirement already satisfied: pyasn1>=0.1.7 in /usr/lib/python2.7/site-packages (from paramiko) (0.4.3)
Requirement already satisfied: bcrypt>=3.1.3 in /usr/lib64/python2.7/site-packages (from paramiko) (3.1.4)
Requirement already satisfied: idna>=2.1 in /usr/lib/python2.7/site-packages (from cryptography>=1.5->paramiko) (2.7)
Requirement already satisfied: cffi>=1.7; platform_python_implementation != "PyPy" in /usr/lib64/python2.7/site-packages (from cryptography>=1.5->paramiko) (1.11.5)
Requirement already satisfied: enum34; python_version < "3" in /usr/lib/python2.7/site-packages (from cryptography>=1.5->paramiko) (1.1.6)
Requirement already satisfied: six>=1.4.1 in /usr/lib/python2.7/site-packages (from cryptography>=1.5->paramiko) (1.11.0)
Requirement already satisfied: asn1crypto>=0.21.0 in /usr/lib/python2.7/site-packages (from cryptography>=1.5->paramiko) (0.24.0)
Requirement already satisfied: ipaddress; python_version < "3" in /usr/lib/python2.7/site-packages (from cryptography>=1.5->paramiko) (1.0.16)
Requirement already satisfied: pycparser in /usr/lib/python2.7/site-packages (from cffi>=1.7; platform_python_implementation != "PyPy"->cryptography>=1.5->paramiko) (2.18)
[root@26759bd75e9b /]# pip install prettytable
Requirement already satisfied: prettytable in /usr/lib/python2.7/site-packages (0.7.2)
[root@26759bd75e9b /]# pip install MySQL-python==1.2.5
Requirement already satisfied: MySQL-python==1.2.5 in /usr/lib64/python2.7/site-packages (1.2.5)
[root@26759bd75e9b /]# pip install gunicorn
Requirement already satisfied: gunicorn in /usr/lib/python2.7/site-packages (19.8.1)
[root@26759bd75e9b /]# pip install psutil
Requirement already satisfied: psutil in /usr/lib64/python2.7/site-packages (5.4.6)
[root@26759bd75e9b /]# pip install xlsxwriter
Requirement already satisfied: xlsxwriter in /usr/lib64/python2.7/site-packages (1.0.5)
[root@26759bd75e9b mysqlapp]# pip install requests
Requirement already satisfied: requests in /usr/lib/python2.7/site-packages (2.18.4)
Requirement already satisfied: urllib3<1.23,>=1.21.1 in /usr/lib/python2.7/site-packages (from requests) (1.22)
Requirement already satisfied: idna<2.7,>=2.5 in /usr/lib/python2.7/site-packages (from requests) (2.6)
Requirement already satisfied: chardet<3.1.0,>=3.0.2 in /usr/lib/python2.7/site-packages (from requests) (3.0.4)
Requirement already satisfied: certifi>=2017.4.17 in /usr/lib/python2.7/site-packages (from requests) (2018.4.16)


[root@26759bd75e9b /]# yum install -y nginx
Installed:
  nginx.x86_64 1:1.12.2-2.el7                                                                                                                              

Dependency Installed:
  fontconfig.x86_64 0:2.10.95-11.el7                   fontpackages-filesystem.noarch 0:1.44-8.el7    freetype.x86_64 0:2.4.11-15.el7                    
  gd.x86_64 0:2.0.35-26.el7                            gperftools-libs.x86_64 0:2.6.1-1.el7           libX11.x86_64 0:1.6.5-1.el7                        
  libX11-common.noarch 0:1.6.5-1.el7                   libXau.x86_64 0:1.0.8-2.1.el7                  libXpm.x86_64 0:3.5.12-1.el7                       
  libjpeg-turbo.x86_64 0:1.2.90-5.el7                  libpng.x86_64 2:1.5.13-7.el7_2                 libxcb.x86_64 0:1.12-1.el7                         
  libxslt.x86_64 0:1.1.28-5.el7                        lyx-fonts.noarch 0:2.2.3-1.el7                 make.x86_64 1:3.82-23.el7                          
  nginx-all-modules.noarch 1:1.12.2-2.el7              nginx-filesystem.noarch 1:1.12.2-2.el7         nginx-mod-http-geoip.x86_64 1:1.12.2-2.el7         
  nginx-mod-http-image-filter.x86_64 1:1.12.2-2.el7    nginx-mod-http-perl.x86_64 1:1.12.2-2.el7      nginx-mod-http-xslt-filter.x86_64 1:1.12.2-2.el7   
  nginx-mod-mail.x86_64 1:1.12.2-2.el7                 nginx-mod-stream.x86_64 1:1.12.2-2.el7         openssl.x86_64 1:1.0.2k-12.el7                     
  perl.x86_64 4:5.16.3-292.el7                         perl-Carp.noarch 0:1.26-244.el7                perl-Encode.x86_64 0:2.51-7.el7                    
  perl-Exporter.noarch 0:5.68-3.el7                    perl-File-Path.noarch 0:2.09-2.el7             perl-File-Temp.noarch 0:0.23.01-3.el7              
  perl-Filter.x86_64 0:1.49-3.el7                      perl-Getopt-Long.noarch 0:2.40-3.el7           perl-HTTP-Tiny.noarch 0:0.033-3.el7                
  perl-PathTools.x86_64 0:3.40-5.el7                   perl-Pod-Escapes.noarch 1:1.04-292.el7         perl-Pod-Perldoc.noarch 0:3.20-4.el7               
  perl-Pod-Simple.noarch 1:3.28-4.el7                  perl-Pod-Usage.noarch 0:1.63-3.el7             perl-Scalar-List-Utils.x86_64 0:1.27-248.el7       
  perl-Socket.x86_64 0:2.010-4.el7                     perl-Storable.x86_64 0:2.45-3.el7              perl-Text-ParseWords.noarch 0:3.29-4.el7           
  perl-Time-HiRes.x86_64 4:1.9725-3.el7                perl-Time-Local.noarch 0:1.2300-2.el7          perl-constant.noarch 0:1.27-2.el7                  
  perl-libs.x86_64 4:5.16.3-292.el7                    perl-macros.x86_64 4:5.16.3-292.el7            perl-parent.noarch 1:0.225-244.el7                 
  perl-podlators.noarch 0:2.5.1-3.el7                  perl-threads.x86_64 0:1.87-4.el7               perl-threads-shared.x86_64 0:1.43-6.el7            

Complete!


[root@26759bd75e9b /]# vi /etc/nginx/nginx.conf
server {
 
    listen 80;
    server_name  www.toberoot.com;
    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-Ip $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_pass http://127.0.0.1:5000;
    }
 
    error_page 404 /404.html;
        location = /40x.html {
    }
 
    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}

[root@26759bd75e9b /]# mkdir /alidata/  
[root@26759bd75e9b /]# scp -r root@192.168.1.3:/alidata/toberoot/mysqlapp /alidata/ 
root@192.168.1.3's password: 
master                                                                                                                   100%   41    35.3KB/s   00:00    
HEAD                                                                                                                     100%   32    19.2KB/s   00:00    
description                                                                                                              100%   73    46.0KB/s   00:00    
applypatch-msg.sample                                                                                                    100%  452   336.7KB/s   00:00    
commit-msg.sample                                                                                                        100%  896   627.3KB/s   00:00    
post-update.sample                                                                                                       100%  189    71.1KB/s   00:00    
pre-applypatch.sample                                                                                                    100%  398   326.8KB/s   00:00    
pre-commit.sample                                                                                                        100% 1704   845.0KB/s   00:00    
pre-push.sample                                                                                                          100% 1348   578.8KB/s   00:00    
pre-rebase.sample                                                                                                        100% 4951   998.9KB/s   00:00    
prepare-commit-msg.sample                                                                                                100% 1239     1.3MB/s   00:00    
update.sample                                                                                                            100% 3611     3.8MB/s   00:00    
exclude                                                                                                                  100%  240   125.0KB/s   00:00    
HEAD                                                                                                                     100%   23    13.7KB/s   00:00    
config                                                                                                                   100%  272   212.5KB/s   00:00    
179d64816a8dc573483c045ee9c0e58b179497                                                                                   100%  142   125.2KB/s   00:00    
d21a851a76ceeb3c4c69261dcbe52d4b6bec81                                                                                   100%  166   133.9KB/s   00:00    
6c0644d28ba1284a116390dbddb1823795e705                                                                                   100%  350   161.3KB/s   00:00    
a522573183b27ca0468273e7e028aa580599a2                                                                                   100%  197   138.9KB/s   00:00    
7ae176b04c485467007ef35496be7c648084f2                                                                                   100%   64    56.0KB/s   00:00    
ce5cda9bafd4135e57e167fe103982b574d15f                                                                                   100%  250   239.4KB/s   00:00    
57a8e736e8ec4c138564cc1b85b48f58fc9c48                                                                                   100%  201   192.0KB/s   00:00    
c47ebb67af85cb71d1877a44262b90e78e4819                                                                                   100%  186   210.2KB/s   00:00    
32adde553e276a35ac90f6652acf2a92ed50b5                                                                                   100%  340   364.1KB/s   00:00    
d28396a45905ff4242decbfadd6af0359b8ae8                                                                                   100%  825   639.2KB/s   00:00    
3887d1ba5ec409a80bca5042ecaff50706ec14                                                                                   100% 4858     3.9MB/s   00:00    
80a70a55be63c61103bceb59e84bcc626418de                                                                                   100% 1113     1.0MB/s   00:00    
e885b7ee3035c9b8f90751c200eabbec5e69f1                                                                                   100%  312   301.3KB/s   00:00    
9de29bb2d1d6434b8b29ae775ad8c2e48c5391                                                                                   100%   15     9.8KB/s   00:00    
5fb232128e5b2cfab01545960caacf22c04e55                                                                                   100%  161   103.8KB/s   00:00    
642d5df132d0923a74ef2a913d3fb33188ec88                                                                                   100% 2111     1.4MB/s   00:00    
9619a3a5bfe58e48209274cf2de6b4e984e825                                                                                   100% 2468     2.3MB/s   00:00    
a3305516fb77b494031d89aceca8d73528f85a                                                                                   100%  872   900.4KB/s   00:00    
f53e28c68706f25aee2e4141a20b910e3b6574                                                                                   100% 5600     4.0MB/s   00:00    
7bb5f144ba30ff51d0389e3e4b0d10383092cc                                                                                   100% 5302     3.5MB/s   00:00    
2c9b6545b31778c2a163c228f35be2da4780c5                                                                                   100% 8197     6.4MB/s   00:00    
a9dd5b5defee45a346e08a33240891fa8563eb                                                                                   100% 8337     4.5MB/s   00:00    
bc7cc004bc1a1a6b6d2605989777e8424c6ddb                                                                                   100%   10KB   9.2MB/s   00:00    
1eba0f0d7f2b920311342d77dd61d13a97ef97                                                                                   100%   16KB   6.4MB/s   00:00    
8ae5a5fd159a2675ea24f6b54df16fc61e7643                                                                                   100% 1578   995.2KB/s   00:00    
805a6e0acb4a1c51e8c0bc89e5f0a66b8eace9                                                                                   100% 2042     1.1MB/s   00:00    
a459e7d204e0e0e819f458dd815d2fd161d197                                                                                   100% 1244     1.4MB/s   00:00    
460d878dfa5d8855bb3284fe577d7799397cb8                                                                                   100% 1754     1.2MB/s   00:00    
06848370e6b4b59b3ce5588dc29d901dab4699                                                                                   100%   51    31.8KB/s   00:00    
dffc8c4cf4d3990386b4a64fd1bca2a06f0511                                                                                   100% 5455     2.8MB/s   00:00    
9daf137581536c267f78ebc948d4c10bfb6791                                                                                   100%  321   388.5KB/s   00:00    
e5050cb1a61dc4ff479a69a0fbc0356d7212c9                                                                                   100%  380    95.0KB/s   00:00    
90b21f8d493a6ff4cb0998ecd61caa3be365e3                                                                                   100%  962   755.0KB/s   00:00    
63b0283ea54a9db37179862157db0ec73f91ca                                                                                   100%  132   119.2KB/s   00:00    
3db7333a803467054ed725ba2e9516e7c5a76e                                                                                   100%  308   160.4KB/s   00:00    
9d66a70df504f5b79ca73e503bada76317fa58                                                                                   100%  362   161.0KB/s   00:00    
317893062441656c79e92e8eb743381c9f5527                                                                                   100%  430   355.7KB/s   00:00    
ed9f4d555eb924f28fb31492fc6e9631189998                                                                                   100%  305   292.3KB/s   00:00    
bf8698a100f8eab3e896cee570309d5ba2262f                                                                                   100%  587   688.3KB/s   00:00    
d1e72605421a23458a5c1167720f8d4a008cc5                                                                                   100%  491   281.3KB/s   00:00    
101e79611c644ea459ca8d01c320cd492eac8d                                                                                   100% 4716     4.0MB/s   00:00    
5fe9511e55fffda8f6db605a54e1c645c6c5a7                                                                                   100%  164   149.9KB/s   00:00    
1de0dd8d82ed1e31800e9eff7d359cd77a138e                                                                                   100%   11KB   8.4MB/s   00:00    
70edaf866ff4d4b0c4ae174604dd8c79e44517                                                                                   100%   15KB   6.2MB/s   00:00    
66999213615599998148203dd056a09190dbe9                                                                                   100%   30KB  13.3MB/s   00:00    
6398983410dbd5c9798aed9a489f91fd144860                                                                                   100%   20KB   5.9MB/s   00:00    
a99214063ed192b56425a142e9a35c5c37829d                                                                                   100%  986   542.6KB/s   00:00    
packed-refs                                                                                                              100%  107    44.1KB/s   00:00    
HEAD                                                                                                                     100%  191   190.4KB/s   00:00    
master                                                                                                                   100%  191    45.6KB/s   00:00    
HEAD                                                                                                                     100%  191   114.1KB/s   00:00    
index                                                                                                                    100% 3584     2.0MB/s   00:00    
Project_Default.xml                                                                                                      100%  438   570.5KB/s   00:00    
misc.xml                                                                                                                 100%  301   164.8KB/s   00:00    
modules.xml                                                                                                              100%  268    95.3KB/s   00:00    
mysqlapp.iml                                                                                                             100%  639   411.4KB/s   00:00    
workspace.xml                                                                                                            100%   34KB  19.3MB/s   00:00    
__init__.py                                                                                                              100%    0     0.0KB/s   00:00    
auto_login.py                                                                                                            100% 3122     1.3MB/s   00:00    
cloudcare_lib.py                                                                                                         100% 6658     3.4MB/s   00:00    
geckodriver.log                                                                                                          100% 2299     1.4MB/s   00:00    
__init__.pyc                                                                                                             100%  134   139.5KB/s   00:00    
cloudcare_lib.pyc                                                                                                        100% 5639     2.5MB/s   00:00    
get_weekly_case_info.py                                                                                                  100%   15KB   5.8MB/s   00:00    
get_weekly_case_info.pyc                                                                                                 100%   18KB   7.5MB/s   00:00    
get_case_info.py                                                                                                         100%   14KB   4.3MB/s   00:00    
get_os_info.py                                                                                                           100% 3630     4.0MB/s   00:00    
01.jpg                                                                                                                   100% 5712     2.2MB/s   00:00    
cloudcare_form.html                                                                                                      100%  627   431.2KB/s   00:00    
data_check.html                                                                                                          100% 2950     1.9MB/s   00:00    
get_os_info.html                                                                                                         100%  162   143.2KB/s   00:00    
home.html                                                                                                                100%  482   240.4KB/s   00:00    
index_mysqlcheck.html                                                                                                    100%  685   504.2KB/s   00:00    
linux.html                                                                                                               100%  692   423.3KB/s   00:00    
linux_form.html                                                                                                          100%  602   408.0KB/s   00:00    
linuxcurrent.html                                                                                                        100%  951   535.4KB/s   00:00    
linuxmoniter.html                                                                                                        100%  797   893.0KB/s   00:00    
get_os_info.pyc                                                                                                          100% 4365     5.0MB/s   00:00    
app.py                                                                                                                   100% 3808     1.2MB/s   00:00    
get_mysql_tuning.py                                                                                                      100%   33KB  20.8MB/s   00:00    
get_mysql_tuning.pyc                                                                                                     100%   29KB  20.4MB/s   00:00    
mysqlcon.py                                                                                                              100% 2998     2.6MB/s   00:00    
mysqlcon.pyc                                                                                                             100% 3972     3.3MB/s   00:00    
mysql故障排查思路.txt                                                                                              100%   13KB   5.1MB/s   00:00    
01.png                                                                                                                   100%   17KB   5.6MB/s   00:00    
02.png                                                                                                                   100%   13KB   9.8MB/s   00:00    
03.png                                                                                                                   100%   16KB  15.1MB/s   00:00    
04.png                                                                                                                   100%   32KB  25.6MB/s   00:00    
Image 033.png                                                                                                            100%   21KB   8.9MB/s   00:00    
readme.md                                                                                                                100% 1986     1.0MB/s   00:00  


[root@26759bd75e9b mysqlapp]# pwd
/alidata/mysqlapp/mysqlapp
[root@26759bd75e9b mysqlapp]# cat start.sh << ENDF
#!/usr/bin/bash
cd /alidata/mysqlapp/mysqlapp/
gunicorn app:app --bind 0.0.0.0:5000 --daemon
systemctl start nginx
ENDF
[root@26759bd75e9b mysqlapp]# chmod a+x start.sh
[root@26759bd75e9b mysqlapp]# ll
total 104
-rw-r--r-- 1 root root  3808 Jun 11 05:28 app.py
drwxr-xr-x 2 root root  4096 Jun 11 05:28 cc
-rw-r--r-- 1 root root 34267 Jun 11 05:28 get_mysql_tuning.py
-rw-r--r-- 1 root root 29345 Jun 11 05:28 get_mysql_tuning.pyc
-rw-r--r-- 1 root root  3630 Jun 11 05:28 get_os_info.py
-rw-r--r-- 1 root root  4365 Jun 11 05:28 get_os_info.pyc
-rw-r--r-- 1 root root  2998 Jun 11 05:28 mysqlcon.py
-rw-r--r-- 1 root root  3972 Jun 11 05:28 mysqlcon.pyc
drwxr-xr-x 2 root root    19 Jun 11 05:28 pic
-rwxr-xr-x 1 root root   111 Jun 11 05:30 start.sh
drwxr-xr-x 2 root root  4096 Jun 11 05:28 templates


[root@26759bd75e9b mysqlapp]# gunicorn app:app --bind 0.0.0.0:5000 --daemon
[root@26759bd75e9b mysqlapp]# ps -ef|grep gun
root       1574      1  0 05:37 ?        00:00:00 /usr/bin/python2 /usr/bin/gunicorn app:app --bind 0.0.0.0:5000 --daemon
root       1579   1574  4 05:37 ?        00:00:00 /usr/bin/python2 /usr/bin/gunicorn app:app --bind 0.0.0.0:5000 --daemon
root       1587      1  0 05:37 pts/0    00:00:00 grep --color=auto gun

[root@26759bd75e9b mysqlapp]# systemctl start nginx
Failed to get D-Bus connection: Operation not permitted





[root@mastera toberoot]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
rhel7.2             7.5.1804            60bf63c38109        4 hours ago         277MB
dao-2048            0.02                141c918abbbf        4 days ago          8.01MB
booboo-2048         latest              7929bcd70e47        2 years ago         8.01MB
dao-2048            0.01                7929bcd70e47        2 years ago         8.01MB
[root@mastera toberoot]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
26759bd75e9b        rhel7.2:7.5.1804    "/bin/bash"         2 hours ago         Up 2 hours                              nginx_booboo


[root@mastera toberoot]# docker run --privileged  -d --name booboo_test -p 80:80 rhel7.2_toberoot  /usr/sbin/init
3dd0a07d4445aec4100e083e3565587eeac42aab11f7944970b083dc90d1bf97
[root@mastera toberoot]# docker exec booboo_test systemctl stop firewalld
[root@mastera toberoot]# docker exec booboo_test bash /alidata/mysqlapp/mysqlapp/start.sh
[root@mastera toberoot]# docker exec booboo_test ss -luntp|grep nginx
tcp    LISTEN     0      128       *:80                    *:*                   users:(("nginx",pid=7197,fd=6),("nginx",pid=7196,fd=6),("nginx",pid=7195,fd=6))
tcp    LISTEN     0      128      :::80                   :::*                   users:(("nginx",pid=7197,fd=7),("nginx",pid=7196,fd=7),("nginx",pid=7195,fd=7))
[root@mastera toberoot]# docker exec booboo_test ss -luntp|grep 5000
tcp    LISTEN     0      128       *:5000                  *:*                   users:(("gunicorn",pid=7192,fd=5),("gunicorn",pid=7185,fd=5))




```

## 报错

### 无法执行`systemctl`

```shell
[root@26759bd75e9b mysqlapp]# systemctl start nginx
Failed to get D-Bus connection: Operation not permitted
```

### 解决方法

```shell
docker run --privileged  -d --name booboo_test -p 80:80 rhel7.2_toberoot  /usr/sbin/init
docker exec booboo_test systemctl stop firewalld
docker exec booboo_test bash /alidata/mysqlapp/mysqlapp/start.sh
```



## dockerfile

```shell
[root@mastera toberoot]# cat Dockerfile 
FROM rhel7.2:7.5.1804
MAINTAINER Booboo Wei <rgweiyaping@gmail.com>
LABEL Vendor="RHEL 7.2"
LABEL License=GPLv2
LABEL Version=3.10.0-327.el7.x86_64

RUN yum install -y epel-release && yum install -y python-pip
RUN yum install -y python-devel mariadb-devel gcc && yum clean all
RUN pip install --upgrade pip
RUN pip install setuptools flask paramiko prettytable MySQL-python==1.2.5 gunicorn psutil xlswriter requests
RUN mkdir /alidata
EXPOSE 80


ADD /alidata/mysqlapp /alidata/
ADD start.sh /alidata/mysqlapp/mysqlapp/run.sh
RUN chmod -v +x /alidata/mysqlapp/mysqlapp/run.sh
RUN systemctl stop firewalld
RUN systemctl disable firewalld
RUN yum install -y nginx && yum clean all
ADD nginx.conf /etc/nginx/nginx.conf

CMD ["/alidata/mysqlapp/mysqlapp/run.sh"]

[root@mastera toberoot]# ls
Dockerfile  mkimage-yum.sh  mysqlapp  nginx.conf
[root@mastera toberoot]# docker build --rm -t zyadmin/nginx ./

```

## 保存为tar包


`docker ps -a|awk '{if(NR!=1)print $1}'| xargs docker rm`

```shell
[root@mastera toberoot]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
rhel7.2_toberoot    latest              65bb0c9a9447        2 hours ago         488MB
rhel7.2             7.5.1804            60bf63c38109        6 hours ago         277MB
dao-2048            0.02                141c918abbbf        4 days ago          8.01MB
booboo-2048         latest              7929bcd70e47        2 years ago         8.01MB
dao-2048            0.01                7929bcd70e47        2 years ago         8.01MB
[root@mastera toberoot]# docker save rhel7.2_toberoot > mysqlapp.tar
[root@mastera toberoot]# ls
build  Dockerfile  mkimage-yum.sh  mysqlapp  mysqlapp.tar  nginx.conf
[root@mastera toberoot]# ll -h
total 489M
drwxr-xr-x 2 root root   23 Jun 11 00:29 build
-rw-r--r-- 1 root root  759 Jun 11 00:03 Dockerfile
-rwxr-xr-x 1 root root 4.7K Jun 10 18:53 mkimage-yum.sh
drwxr-xr-x 5 root root   92 Jun 10 19:16 mysqlapp
-rw-r--r-- 1 root root 489M Jun 11 00:45 mysqlapp.tar
-rw-r--r-- 1 root root 2.6K Jun 10 22:26 nginx.conf

[root@mastera toberoot]# scp mysqlapp.tar 10.200.6.239:/alidata/docker
The authenticity of host '10.200.6.239 (10.200.6.239)' can't be established.
ECDSA key fingerprint is 38:84:0c:c0:8c:d8:98:24:ac:c9:88:25:35:b3:30:2c.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.200.6.239' (ECDSA) to the list of known hosts.
root@10.200.6.239's password: 
mysqlapp.tar                                                                                                                          100%  488MB   5.0MB/s   01:37    

```
## 加载tar包为images

```shell
[root@trouble-shooting-srv1 docker]# ll
total 500224
-rw-r--r-- 1 root root 512227840 Jun 11 16:04 mysqlapp.tar
[root@trouble-shooting-srv1 docker]# docker load < mysqlapp.tar 
1d032e180337: Loading layer [==================================================>]  291.8MB/291.8MB
a6d8bde43072: Loading layer [==================================================>]  220.5MB/220.5MB
Loaded image: rhel7.2_toberoot:latest
[root@trouble-shooting-srv1 docker]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
rhel7.2_toberoot    latest              65bb0c9a9447        2 hours ago         488MB


[root@trouble-shooting-srv1 docker]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
rhel7.2_toberoot    latest              65bb0c9a9447        2 hours ago         488MB
[root@trouble-shooting-srv1 docker]# docker run --privileged  -d --name booboo_test -p 80:80 rhel7.2_toberoot  /usr/sbin/init
88ea93905d406e578382927231b88b96abd6cec62ca5f23d3e731f9ab9679795
[root@trouble-shooting-srv1 docker]# docker exec booboo_test systemctl stop firewalld
[root@trouble-shooting-srv1 docker]# docker exec booboo_test bash /alidata/mysqlapp/mysqlapp/start.sh
[root@trouble-shooting-srv1 docker]# docker exec booboo_test ps -ef|grep nginx
root       484     1  0 08:07 ?        00:00:00 nginx: master process /usr/sbin/nginx
nginx      485   484  0 08:07 ?        00:00:00 nginx: worker process
nginx      486   484  0 08:07 ?        00:00:00 nginx: worker process
[root@trouble-shooting-srv1 docker]# docker exec booboo_test ps -ef|grep gun
root       474     0  0 08:07 ?        00:00:00 /usr/bin/python2 /usr/bin/gunicorn app:app --bind 0.0.0.0:5000 --daemon
root       482   474  1 08:07 ?        00:00:00 /usr/bin/python2 /usr/bin/gunicorn app:app --bind 0.0.0.0:5000 --daemon
[root@trouble-shooting-srv1 docker]# vim /etc/rc.local 
docker start booboo_test
docker exec booboo_test systemctl stop firewalld
docker exec booboo_test bash /alidata/mysqlapp/mysqlapp/start.sh
```

