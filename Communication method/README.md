# 3. 통신 방법

---

# 3.1. tew-831dr

다음은 tew-831dr공유기의 통신 방법에 대해 정리하였다.

---

## 3.1.1. tew-831dr 초기화 스크립트

---

tew-831dr 공유기의 초기화는 `/etc/init.d/rcS` 스크립트를 통해 이루어지며, 내용은 다음과 같다.

```bash
#!/bin/sh

ifconfig lo 127.0.0.1

CINIT=1

hostname rlx-linux

mount -t proc proc /proc
mount -t ramfs ramfs /var
if [ -d "/hw_setting" ];then
    mount -t yaffs2 -o tags-ecc-off -o inband-tags /dev/mtdblock1 /hw_setting
fi

mkdir /var/tmp
mkdir /var/web
mkdir /var/log
mkdir /var/run
mkdir /var/lock
mkdir /var/system
mkdir /var/dnrd
mkdir /var/avahi
mkdir /var/dbus-1
mkdir /var/run/dbus
mkdir /var/lib
mkdir /var/lib/misc
mkdir /var/home
mkdir /var/root
mkdir /var/tmp/net
###for tr069
mkdir /var/cwmp_default
mkdir /var/cwmp_config

# JCG wangxg++
eval `flash get DEVICE_NAME`
hostname $DEVICE_NAME

if [ ! -f /var/cwmp_default/DefaultCwmpNotify.txt ]; then
        cp -p /etc/DefaultCwmpNotify.txt /var/cwmp_default/DefaultCwmpNotify.txt 2>/dev/null
fi

##For miniigd
mkdir /var/linuxigd
cp /etc/tmp/pics* /var/linuxigd 2>/dev/null

##For pptp
mkdir /var/ppp
mkdir /var/ppp/peers

#smbd
mkdir /var/config
mkdir /var/private
mkdir /var/tmp/usb
mkdir /var/tmp/mmc

#snmpd
mkdir /var/net-snmp

cp /bin/pppoe.sh /var/ppp/true
echo "#!/bin/sh" > /var/ppp/true
#echo "PASS"     >> /var/ppp/true

#for console login
cp /etc/shadow.sample /var/shadow

#for weave
cp /etc/avahi-daemon.conf /var/avahi

#extact web pages
cd /web
#flash extr /web
cd /

mkdir -p /var/udhcpc
mkdir -p /var/udhcpd
cp /bin/init.sh /var/udhcpc/eth0.deconfig
echo " " > /var/udhcpc/eth0.deconfig
cp /bin/init.sh /var/udhcpc/eth1.deconfig
echo " " > /var/udhcpc/eth1.deconfig
cp /bin/init.sh /var/udhcpc/br0.deconfig
echo " " > /var/udhcpc/br0.deconfig
cp /bin/init.sh /var/udhcpc/wlan0.deconfig
echo " " > /var/udhcpc/wlan0.deconfig

if [ "$CINIT" = 1 ]; then
startup.sh
fi

# for wapi certs related
mkdir /var/myca
# wapi cert(must done before init.sh)
cp -rf /usr/local/ssl/* /var/myca/ 2>/dev/null
# loadWapiFiles >/dev/null 2>&1

# for wireless client mode 802.1x
mkdir /var/1x
cp -rf /usr/1x/* /var/1x/ 2>/dev/null
mkdir /var/openvpn
cp -rf /usr/share/openvpn/* /var/openvpn 2>/dev/null

# Start system script
init.sh gw all

# modify dst-cache setting
echo "24576" > /proc/sys/net/ipv4/route/max_size
echo "180" > /proc/sys/net/ipv4/route/gc_thresh
echo 20 > /proc/sys/net/ipv4/route/gc_elasticity
# echo 35 > /proc/sys/net/ipv4/route/gc_interval
# echo 60 > /proc/sys/net/ipv4/route/secret_interval
# echo 10 > /proc/sys/net/ipv4/route/gc_timeout

# echo "4096" > /proc/sys/net/nf_conntrack_max
echo "12288" > /proc/sys/net/netfilter/nf_conntrack_max
echo "600" > /proc/sys/net/ipv4/netfilter/ip_conntrack_tcp_timeout_established
echo "20" > /proc/sys/net/ipv4/netfilter/ip_conntrack_tcp_timeout_time_wait
echo "20" > /proc/sys/net/ipv4/netfilter/ip_conntrack_tcp_timeout_close
echo "90" > /proc/sys/net/ipv4/netfilter/ip_conntrack_udp_timeout
echo "120" > /proc/sys/net/ipv4/netfilter/ip_conntrack_udp_timeout_stream
echo "90" > /proc/sys/net/ipv4/netfilter/ip_conntrack_generic_timeout
# echo "1048576" > /proc/sys/net/ipv4/rt_cache_rebuild_count
echo "32" > /proc/sys/net/netfilter/nf_conntrack_expect_max

# modify IRQ Affinity setting
echo "3" > /proc/irq/33/smp_affinity

#echo 1 > /proc/sys/net/ipv4/ip_forward #don't enable ip_forward before set MASQUERADE
#echo 2048 > /proc/sys/net/core/hot_list_length

# start web server
ls /bin/watchdog > /dev/null && watchdog 1000&
dualrecover 3
dualrecover 4
boa
post_startup.sh
```

마지막 주석을 보면, start web server라는 주석 내용 아래에 웹 서버 실행 관련 파일들을 실행하며, boa 파일을 실행하는 것을 알 수가 있다. boa 파일은 펌웨어 내 `/bin/boa`에 있으며, 웹 서버의 모든 요청을 처리한다.

## 3.1.2. boa 웹서버

---

boa 웹서버는 임베디드에서 사용하는 경량 웹 서버로 tew-831dr공유기에서도 사용된다. 그리고 boa웹 서버는 CGI를 지원하여 동적 웹 콘텐츠를 생성한다.

![image.png](img/image1.png?raw=true)

## 3.1.3. /bin/boa 파일

---

클라이언트가 관리자 페이지의 특정 페이지에서 보내는 패킷 요청은 모두 boafrm을 통해서 boa 파일에서 처리된다. boafrm은 공유기의 웹 인터페이스에서 사용자의 요청을 처리하는 CGI 실행 파일로 동작하며, boa 웹서버는 HTTP 요청을 받을 때 CGI 요청이 포함되면 boafrm을 실행하고, 각 페이지 엔드포인트 HTML의 form태그로 인하여 `boafrm/` 파일로 이동하게 된다.

![image.png](img/image2.png?raw=true)

그리고 `boafrm/formDdns`로 이동한 요청 처리는 모두 `/bin/boa` 파일에서 이루어진다.

![image.png](img/image3.png?raw=true)

위 사진처럼 IDA에서 `/bin/boa` 파일을 확인해보면 전부 해당 태그의 처리 과정이 존재한다. IDA에서 x를 눌러 formDdns가 참조하는 영역을 보면 다음과 같다.

![image.png](img/image4.png?raw=true)

사진과 같이, `boafrm/` 파일에는 각각 sub_43EED0 등의 처리 함수가 존재한다

boafrm/formDdns의 입력값 요청을 보면, POST 요청을 통해 입력값을 `/boafrm/formDdns`로 보낸다.

![image.png](img/image5.png?raw=true)

![image.png](img/image6.png?raw=true)

관리자 페이지의 각 설정 페이지에서 CGI 요청을 하는 패킷은 모두 `boafrm/`을 거쳐 `/bin/boa`로 넘어간다.
