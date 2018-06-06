# LXD Container Build For CentOS 7 Guest

Steps for creating a [LXD](https://www.ubuntu.com/containers/lxd) golden base CentOS 7 64bit image for LXD guest container usage intended for [Centmin Mod LEMP stack](https://centminmod.com)

# Golden base CentOS 7 64bit image

Base CentOS 7 64bit image uses an updated Systemd 234 version from [Facebook RPM backports](https://github.com/facebookincubator/rpm-backports) built RPMs provided by [Jan Synacek](https://copr.fedorainfracloud.org/coprs/jsynacek/systemd-backports-for-centos-7/). This is due to a bug in native CentOS 7 Systemd 219 version when used within container environments, the max open file description limits via `NOFILE` are not correctly getting the values being set within the LXD container environment ([details](https://discuss.linuxcontainers.org/t/ulimit-nofiles-in-centos-7-5-containers-a-systemd-bug/1953)).

Create `centos75-base` LXD container to use for golden base image creation for image named `centos7-systemdfix` and ensuring to set default LXD profile to backlist syscalls for `keyctl errno 38` to ensure MariaDB MySQL server can start up within CentOS 7 LXD container environment when using newer Systemd 234 version.

```
lxc profile set default security.syscalls.blacklist "keyctl errno 38"
lxc launch images:centos/7 centos75-base
lxc exec centos75-base -- echo "export LANG=en_US.UTF-8" >> /etc/profile.d/locale.sh
lxc exec centos75-base -- echo "export LANGUAGE=en_US.UTF-8" >> /etc/profile.d/locale.sh
lxc exec centos75-base -- source /etc/profile.d/locale.sh
lxc exec centos75-base -- sed -i "s|plugins=1|plugins=1\nexclude=\*.i386 \*.i586 \*.i686|" /etc/yum.conf
lxc exec centos75-base -- yum -y update
lxc exec centos75-base -- yum -y install wget openssh openssh-server curl curl-devel libcurl libcurl-devel
lxc exec centos75-base -- wget https://copr.fedorainfracloud.org/coprs/jsynacek/systemd-backports-for-centos-7/repo/epel-7/jsynacek-systemd-backports-for-centos-7-epel-7.repo -O /etc/yum.repos.d/jsynacek-systemd-centos-7.repo
lxc exec centos75-base -- yum -y update systemd
lxc exec centos75-base -- systemctl enable sshd
lxc exec centos75-base -- systemctl restart sshd
lxc exec centos75-base -- systemctl status sshd

## changing default sshd port
#lxc exec centos75-base -- grep Port /etc/ssh/sshd_config
#lxc exec centos75-base -- sed -e 's|#Port 22|Port 622|' /etc/ssh/sshd_config | grep 622
#lxc exec centos75-base -- sed -i 's|#Port 22|Port 622|' /etc/ssh/sshd_config
#lxc exec centos75-base -- grep Port /etc/ssh/sshd_config
#lxc exec centos75-base -- systemctl restart sshd
#lxc exec centos75-base -- systemctl status sshd

lxc restart centos75-base
lxc publish centos75-base --alias centos7-systemdfix --force
lxc list
lxc image list
lxc delete centos75-base --force
lxc list
```

LXD Image List

```
lxc image list
+--------------------+--------------+--------+---------------------------------------------+--------+----------+-----------------------------+
|       ALIAS        | FINGERPRINT  | PUBLIC |                 DESCRIPTION                 |  ARCH  |   SIZE   |         UPLOAD DATE         |
+--------------------+--------------+--------+---------------------------------------------+--------+----------+-----------------------------+
| centos7-systemdfix | fc44baf0b7ca | no     |                                             | x86_64 | 158.69MB | Jun 6, 2018 at 7:41pm (UTC) |
+--------------------+--------------+--------+---------------------------------------------+--------+----------+-----------------------------+
|                    | 9879a79ac2b2 | no     | ubuntu 18.04 LTS amd64 (release) (20180522) | x86_64 | 172.97MB | Jun 4, 2018 at 5:46pm (UTC) |
+--------------------+--------------+--------+---------------------------------------------+--------+----------+-----------------------------+
|                    | e465dac68a91 | no     | Centos 7 amd64 (20180606_02:16)             | x86_64 | 83.45MB  | Jun 6, 2018 at 6:14am (UTC) |
+--------------------+--------------+--------+---------------------------------------------+--------+----------+-----------------------------+
```

# Using golden base image to launch CentOS 7 64bit LXD guest containers

Using the golden base image `centos7-systemdfix` to launch a new CentOS 7 LXD container named `centos75`

```
lxc launch centos7-systemdfix centos75
lxc config set centos75 boot.autostart true
# optionally apply memory limits
# http://lxd.readthedocs.io/en/latest/containers/
# i.e. limit container memory to 4096MB
# lxc config set centos75 limits.memory 4096MB
lxc exec centos75 -- systemctl --version
```

checking Systemd version for `centos75` container

```
lxc exec centos75 -- systemctl --version
systemd 234
+PAM +AUDIT +SELINUX +IMA -APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD -IDN2 +IDN default-hierarchy=hybrid
```

LXD container listing

```
lxc list ^centos75$
+----------+---------+----------------------+-----------------------------------------------+------------+-----------+
|   NAME   |  STATE  |         IPV4         |                     IPV6                      |    TYPE    | SNAPSHOTS |
+----------+---------+----------------------+-----------------------------------------------+------------+-----------+
| centos75 | RUNNING | 10.71.164.168 (eth0) | fd42:769c:ebd9:a0f7:216:3eff:fefd:23a2 (eth0) | PERSISTENT | 2         |
+----------+---------+----------------------+-----------------------------------------------+------------+-----------+
```

# Checking centos75 Container NOFILE Limits

Within `centos75` container checking custom set nginx process `NOFILE` limits = `524288`. Systemd 234 updated version allowed us to properly set the `NOFILE` limits. With native CentOS 7's Sysdtem 219 version it would of be set to max hardcoded limit of `65536`. If Centmin Mod LEMP stack installer didn't set nginx to `524288` value, updated and fixed Systemd 234 version would of set it to value that LXD host config sets which is `1048576` (shown at [here](#lxd-host-nofile).

```
root      2755  0.0  0.1 114716 23524 ?        Ss   Jun05   0:00 nginx: master process /usr/local/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
nginx     2756  0.0  0.2 143388 46360 ?        S    Jun05   0:33  \_ nginx: worker process
nginx     2757  0.0  0.2 143388 45864 ?        S    Jun05   0:24  \_ nginx: worker process
```

```
prlimit -p 2755
RESOURCE   DESCRIPTION                             SOFT      HARD UNITS
AS         address space limit                unlimited unlimited bytes
CORE       max core file size                         0 unlimited bytes
CPU        CPU time                           unlimited unlimited seconds
DATA       max data size                      unlimited unlimited bytes
FSIZE      max file size                      unlimited unlimited bytes
LOCKS      max number of file locks held      unlimited unlimited locks
MEMLOCK    max locked-in-memory address space  16777216  16777216 bytes
MSGQUEUE   max bytes in POSIX mqueues            819200    819200 bytes
NICE       max nice prio allowed to raise             0         0 
NOFILE     max number of open files              524288    524288 files
NPROC      max number of processes            unlimited unlimited processes
RSS        max resident set size              unlimited unlimited bytes
RTPRIO     max real-time priority                     0         0 
RTTIME     timeout for real-time tasks        unlimited unlimited microsecs
SIGPENDING max number of pending signals          63928     63928 signals
STACK      max stack size                       8388608 unlimited bytes
```

    nginx -V
    nginx version: nginx/1.13.12 (050618-001557)
    built by gcc 7.3.1 20180303 (Red Hat 7.3.1-5) (GCC) 
    built with OpenSSL 1.1.0h  27 Mar 2018
    TLS SNI support enabled
> configure arguments: --with-ld-opt='-ljemalloc -Wl,-z,relro -Wl,-rpath,/usr/local/lib' --with-cc-opt='-m64 -march=native -DTCP_FASTOPEN=23 -g -O3 -fstack-protector-strong -flto -fuse-ld=gold --param=ssp-buffer-size=4 -Wformat -Werror=format-security -Wimplicit-fallthrough=0 -fcode-hoisting -Wp,-D_FORTIFY_SOURCE=2 -gsplit-dwarf' --sbin-path=/usr/local/sbin/nginx --conf-path=/usr/local/nginx/conf/nginx.conf --build=050618-001557 --with-compat --with-http_stub_status_module --with-http_secure_link_module --with-libatomic --with-http_gzip_static_module --with-http_sub_module --with-http_addition_module --with-http_image_filter_module=dynamic --with-http_geoip_module --with-stream_geoip_module --with-stream_realip_module --with-stream_ssl_preread_module --with-threads --with-stream=dynamic --with-stream_ssl_module --with-http_realip_module --add-dynamic-module=../ngx-fancyindex-0.4.2 --add-module=../ngx_cache_purge-2.4.2 --add-module=../ngx_devel_kit-0.3.0 --add-dynamic-module=../set-misc-nginx-module-0.32 --add-dynamic-module=../echo-nginx-module-0.61 --add-module=../redis2-nginx-module-0.15 --add-module=../ngx_http_redis-0.3.7 --add-module=../memc-nginx-module-0.18 --add-module=../srcache-nginx-module-0.31 --add-dynamic-module=../headers-more-nginx-module-0.33 --with-pcre=../pcre-8.42 --with-pcre-jit --with-zlib=../zlib-cloudflare-1.3.0 --with-http_ssl_module --with-http_v2_module --with-openssl=../openssl-1.1.0h --with-openssl-opt='enable-ec_nistp_64_gcc_128'

# LXD Host NOFILE

LXD host process list output for LXD container `centos75`

```
root      6536  0.0  0.0 270664  4872 ?        Ss   Jun05   0:00 [lxc monitor] /var/snap/lxd/common/lxd/containers centos75
100000    6551  0.0  0.0  71752  5032 ?        Ss   Jun05   0:01  \_ /sbin/init
100000    6639  0.0  0.0  73112  9132 ?        Ss   Jun05   0:01      \_ /usr/lib/systemd/systemd-journald
100000    6651  0.0  0.0  45304  2184 ?        Ss   Jun05   0:02      \_ /usr/lib/systemd/systemd-udevd
100081    6660  0.0  0.0  44696  2516 ?        Ss   Jun05   0:01      \_ /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
100000    6674  0.0  0.0  58988  3252 ?        Ss   Jun05   0:00      \_ /usr/lib/systemd/systemd-logind
100038    6675  0.0  0.0  41152  3320 ?        Ss   Jun05   0:00      \_ /usr/sbin/ntpd -u ntp:ntp -g
100000    7016  0.0  0.0  98848  3860 ?        Ss   Jun05   0:00      \_ /sbin/dhclient -1 -q -lf /var/lib/dhclient/dhclient--eth0.lease -pf /var/run/dhclient-eth0.pid -H centos75 eth0
100000    7083  0.0  0.0 101580  3672 ?        Ss   Jun05   0:00      \_ /usr/sbin/sshd -D
100000    7085  0.0  0.0 203456  2636 ?        Ss   Jun05   0:00      \_ pure-ftpd (SERVER)
100000    7086  0.0  0.0 207268  5544 ?        Ssl  Jun05   0:05      \_ /usr/sbin/rsyslogd -n
100000    7088  0.0  0.0  22760  2096 ?        Ss   Jun05   0:00      \_ /usr/sbin/crond -n
100000    7089  0.0  0.0   6528   984 pts/0    Ss+  Jun05   0:00      \_ /sbin/agetty -o -p -- \u --noclear --keep-baud console 115200,38400,9600 linux
100000    7120  0.0  0.0 1085688 9076 ?        Ss   Jun05   0:03      \_ php-fpm: master process (/usr/local/etc/php-fpm.conf)
101001    7145  0.0  0.0 449772  2224 ?        Ssl  Jun05   0:15      \_ /usr/local/bin/memcached -d -m 8 -l 127.0.0.1 -p 11211 -c 2048 -b 2048 -R 200 -t 4 -n 72 -f 1.25 -u memcached -o slab_reassign,slab_automove -P /var/run/memcached/memcached1.pid
100000    7313  0.0  0.0  90352  3580 ?        Ss   Jun05   0:00      \_ /usr/libexec/postfix/master -w
100089    7338  0.0  0.0  90632  4360 ?        S    Jun05   0:00      |   \_ qmgr -l -t unix -u
100089   10852  0.0  0.0  90456  6356 ?        S    12:58   0:00      |   \_ pickup -l -t unix -u
100998    7838  0.0  3.0 6682676 501908 ?      Ssl  Jun05   0:52      \_ /usr/sbin/mysqld
100000    6888  0.0  0.1 114716 23524 ?        Ss   Jun05   0:00      \_ nginx: master process /usr/local/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
101000    6889  0.0  0.2 143388 46360 ?        S    Jun05   0:33      |   \_ nginx: worker process
101000    6891  0.0  0.2 143388 45864 ?        S    Jun05   0:24      |   \_ nginx: worker process
100000    5004  0.0  0.1  68144 23876 ?        Ss   Jun05   0:08      \_ lfd - sleeping
```

NOFILE limit of `centos75` container process ID observed from LXD host level

```
prlimit -p $(lxc info centos75 | awk '$1=="Pid:"{print $2}')
RESOURCE   DESCRIPTION                             SOFT      HARD UNITS
AS         address space limit                unlimited unlimited bytes
CORE       max core file size                 unlimited unlimited bytes
CPU        CPU time                           unlimited unlimited seconds
DATA       max data size                      unlimited unlimited bytes
FSIZE      max file size                      unlimited unlimited bytes
LOCKS      max number of file locks held      unlimited unlimited locks
MEMLOCK    max locked-in-memory address space  16777216  16777216 bytes
MSGQUEUE   max bytes in POSIX mqueues            819200    819200 bytes
NICE       max nice prio allowed to raise             0         0 
NOFILE     max number of open files             1048576   1048576 files
NPROC      max number of processes            unlimited unlimited processes
RSS        max resident set size              unlimited unlimited bytes
RTPRIO     max real-time priority                     0         0 
RTTIME     timeout for real-time tasks        unlimited unlimited microsecs
SIGPENDING max number of pending signals          63928     63928 signals
STACK      max stack size                       8388608 unlimited bytes
```