# 2. 부팅 프로세스 분석

## 2.1. tew-831dr 부팅 프로세스 분석

---

### 2.1.1. Bootloader 단계

```bash
Booting...
init_ram
W init ddr ok
DRAM Type: DDR2
DRAM frequency: 533MHz
DRAM Size: 64MB
JEDEC id EF4017, EXT id 0x0000
found w25q64
flash vendor: Winbond
w25q64, size=8MB, erasesize=4KB, max_speed_hz=29000000Hz
auto_mode=0 addr_width=3 erase_opcode=0x00000020
=>CPU Wake-up interrupt happen! GISR=89000004
--Realtek RTL8197F boot code at 2020.08.04-09:01+0800 v3.4.11E (999MHz)
no sys signature at 00010000!
no sys signature at 00020000!.
.
.
no rootfs signature at 002DF000!
Jump to image start=0x80a00000...

```

1. 초기화 작업 수행(init_ram)
    - DRAM 및 플래시 메모리(w25Q64)를 초기화한다.
    - found w25q64에서 8MB 플래시 메모리를 감지한다.
    - CPU 실행속도(999MHz)를 확인한다.
2. 부트코드 실행
    - Realtek RTL8197F 부트코드를 실행한다.
    - `no sys signature` ,`no rootfs signature` 등의 메시지는 특정 영역에서 실패를 의미한다.
    - 이후 `Jump to image start=0x80a00000...` 로 주소를 jump한다.

### 2.1.2. 커널 로딩 단계

1. 커널 로딩 및 압축 해제

```bash
Jump to image start=0x80a00000...

decompressing kernel:
Uncompressing Linux... done, booting the kernel.
done decompressing kernel.
```

- `Jump to image start=0x80a00000...` 에서 커널 이미지가 메모리에서 실행될 주소로 이동된다.
- `decompressing kernel:` 압축된 커널 이미지를 해제하는 과정을 시작한다.
- `Uncompressing Linux... done, booting the kernel.`압축 해제가 완료되었고 커널이 실행된다.

1. 커널 실행 및 초기화

```bash
start address: 0x8046a8a0
Linux version 3.10.90 (shuangru@devsrv8) (gcc version 4.4.7 (Realtek MSDK-4.4.7 Build 2001) ) #543 Fri May 20 19:51:58 CST 2022
bootconsole [early0] enabled
CPU revision is: 00019385 (MIPS 24Kc)
```

- `start address: 0x8046a8a0` 커널의 실행 시작주소이다.
- `Linux version 3.10.90` Realtek MSDK-4.4.7 기반의 실행되는 커널 버전을 나타낸다.
- CPU 아키텍처는 MIPS 24Kc다.

2. 메모리 및 파일 시스템 초기화

```bash
Determined physical RAM map:
memory: 04000000 @ 00000000 (usable)
Zone ranges:
Normal [mem 0x00000000-0x03ffffff]
Movable zone start for each node
Early memory node ranges
node 0: [mem 0x00000000-0x03ffffff]
Primary instruction cache 64kB, VIPT, 4-way, linesize 32 bytes.
Primary data cache 32kB, 4-way, PIPT, no aliases, linesize 32 bytes
Built 1 zonelists in Zone order, mobility grouping off. Total pages: 4088
Kernel command line: console=ttyS0,38400 root=/dev/mtdblock1
```

- `memory: 04000000 @ 00000000 (usable)`  64MB의 RAM이 사용가능하고, 메모리는 `0x00000000` ~ `0x03ffffff`이다.
- `Kernel command line: console=ttyS0,38400 root=/dev/mtdblock1` 시리얼 콘솔에서 출력을 진행한다. 또한 root파일을 `mtdblock1` 마운트 한다.

 

3. 파티션 생성
    
    ```
    Creating 5 MTD partitions on "m25p80":
    0x000000000000-0x0000002e0000 : "boot+cfg+linux"
    0x0000002e0000-0x0000007a0000 : "rootfs"
    0x0000007a0000-0x0000007c0000 : "cwmp transfer"
    0x0000007c0000-0x0000007e0000 : "cwmp notification"
    0x0000007e0000-0x000000800000 : "cwmp cacert"
    ```
    
    - 플래시 디바이스에서, 위와 같이 5개의 파티션이 생성된다.
    - 이후 `rootfs` 파티션(mtdblock1)이 `squashfs` 파일 시스템으로 마운트된다.(읽기 전용)
4. 주요 하드웨어 드라이버 초기화

```bash
VFS: Mounted root (squashfs filesystem) readonly on device 31:1.
Freeing unused kernel memory: 208K (806ac000 - 806e0000)
```

- `VFS: Mounted root (squashfs filesystem) readonly on device 31:1.` 에서 `squashfs` 파일 시스템이 `/dev/mtdblock1`에 Read-Only로 마운트 된다.
- `Freeing unused kernel memory: 208K (806ac000 - 806e0000)` 208KB 해제 한다.

### 2.1.3 bin/init

```bash

init started: BusyBox v1.13.4 (2022-05-20 19:45:38 CST)
```

- `/bin/init`은 아래와 같이 busybox로 링크되어있다.

![image.png](img/image3.png?raw=true)

- 따라서 `/bin/init`실행 시 `/bin/busybox init`으로 busybox가 실행된다.

### 2.1.4. bin/busybox

`busybox init`은 아래와 같은 코드를 실행한다.

```c
int __fastcall sub_427DCC(int a1, int *a2)
{
  //(일부 변수 초기화 부분 생략)
  v2 = a2[1];
  dword_4465E4 = 2592000;
  v4 = a2 + 1;
  if ( !v2 || (v5 = strcmp(v2, "-q"), v2 = 1, v5) )
  {
    if ( getpid(v2) != 1 )
      sub_404020();
    signal(3, sub_4281EC);
    sub_42E5FC(229376, sub_427A74);
    signal(2, sub_4281E0);
    signal(25, sub_42E5F0);
    sub_42E5FC(25165824, sub_42796C);
    sub_4272FC(0);
    v7 = getenv("CONSOLE");
    if ( v7 || (v7 = getenv("console")) != 0 )
    {
      v8 = open(v7, 2178);
      v9 = v8;
      if ( v8 >= 0 )
      {
        dup2(v8, 0);
        dup2(v9, 1);
        sub_404D20(v9, 2);
      }
    }
    else
    {
      sub_42F0EC();
    }
    v11 = getenv("TERM");
    if ( ioctl(0, 21636, v24) )
    {
      if ( v11 )
        goto LABEL_19;
      v12 = "TERM=linux";
    }
    else
    {
      v10 = 4390912;
      if ( v11 && strcmp(v11, "linux") )
      {
LABEL_19:
        sub_427338();
        sub_404AC4("/");
        setsid();
        putenv("HOME=/", v13);
        putenv("PATH=/sbin:/usr/sbin:/bin:/usr/bin", v14);
        putenv("SHELL=/bin/sh", v15);
        putenv("USER=root", v16);
        if ( *v4 )
          sub_404B4C("RUNLEVEL");
        sub_427510(3, (int)"init started: %s", "BusyBox v1.13.4 (2022-05-20 19:45:38 CST)");
        v17 = (_BYTE *)*v4;
        if ( *v4 && (!strcmp(*v4, "single") || !strcmp(v17, "-s") || *v17 == 49 && !v17[1]) )
        {
		          sub_4273D8(2, "-/bin/sh", "");
          v18 = *a2;
        }
        else
        {
          sub_4275F0();
          v18 = *a2;
        }
        v19 = strlen(v18);
        strncpy(v18, "init", v19);
        for ( i = a2 + 1; *i; ++i )
        {
          v21 = strlen(*i);
          memset(*i, 0, v21);
        }
        sub_427D14(1);
        sub_427D14(8);
        sub_427D14(16);
        signal(1, sub_428154);
        while ( 1 )
        {
          sub_427D14(6);
          sleep(1);
          for ( j = wait(0); j > 0; j = sub_42F17C(0) )
          {
            for ( k = dword_4467A0; k; k = *(_DWORD *)k )
            {
              if ( *(_DWORD *)(k + 4) == j )
              {
                *(_DWORD *)(k + 4) = 0;
                v25 = j;
                sub_427510(1, (int)"process '%s' (pid %d) exited. Scheduling for restart.", (const char *)(k + 41), j);
                j = v25;
              }
            }
          }
        }
      }
      v12 = "TERM=vt102";
    }
    putenv(v12, v10);
    goto LABEL_19;
  }
  return kill(1, 1);
}
```

- 먼저 두 번째 인자 `a2[1]`를 확인 한다. 만약 인자가 없거나 “-q”가 아닌 일반 실행 모드라면, 메인 초기화 작업을 진행하고, 전역 변수(`dword_4465E4`)를 2592000으로 설정한다.
- PID가 1이 아니면, `sub404020()` 을 실행하여 오류를 출력하고 프로세스를 종료한다. 다음으로, 각 시그널 3, 2, 25에 대해 각각 `sub_4281EC`, `sub_4281E0`, `sub42E5F0`의 함수들을 등록한다.
- 환경 변수 “CONSOLE”을 가져와서, 만약 설정되어 있다면 해당 장치를 열어 표준 입출력 0, 1번 파일 디스크립터에 연결한다. 만약 없으면 `sub_42F0EC()`를 호출한다.
    
    ```c
    int __fastcall sub_42EFE4(char a1)
    {
      int v2; // $s0
      int v3; // $v0
      int v4; // $s1
      int result; // $v0
    
      if ( (a1 & 1) != 0 )
        sub_404AC4("/");
      if ( (a1 & 2) != 0 )
      {
        close(0);
        close(1);
        close(2);
      }
      v2 = open("/dev/null", 2);
      if ( v2 < 0 )
        v2 = sub_404E14("/", 0);
      while ( 1 )
      {
        v3 = a1 & 8;
        if ( (unsigned int)v2 >= 2 )
          break;
        v2 = dup(v2);
      }
      v4 = a1 & 4;
      if ( v3 )
        goto LABEL_12;
      sub_42EFAC();
      setsid();
      dup2(v2, 0);
      dup2(v2, 1);
      dup2(v2, 2);
      for ( result = v2 < 3; !result; result = v2 < 3 )
      {
        result = close(v2--);
        if ( !v4 )
          break;
    LABEL_12:
        ;
      }
      return result;
    }
    ```
    
    - `sub_42F0EC()`는 `sub_42EFE4()`함수를 실행한다. `sub_42EFE4()`함수는 플래그에 따라 다음 작업들을 실행하는 함수이다.
        - 비트 0 (a1 & 1): 작업 디렉터리를 루트로 변경한다.
        - 비트 1 (a1 & 2): 표준 입출력(0, 1, 2)을 닫는다.
        - 비트 3 (a1 & 8): 데몬화(setsid, 표준 입출력 재지정) 작업을 건너뛴다.
        - 비트 2 (a1 & 4): 표준 디스크립터 외의 다른 열린 디스크립터들을 닫는다.
- 환경 변수 `TERM`을 확인하고, 터미널 특성을 ioctl()로 조사한 후 `TERM`값이 없거나 특정 값(”linux” 혹은 “vt102”)이 아니면 기본값으로 설정하고, `putenv()`로 터미널 환경 변수를 설정한다.
- `sub_427338()`(터미널 제어 함수)와 `sub_404AC4("/")`를 호출한 후, setsid()로 새로운 세션을 시작한다. 새로운 세션의 `HOME`, `PATH`, `SHELL`, `USER` 등 기본 환경 변수를 설정한 다음, 만약 두 번째 인자가 존재하면 `sub_404B4C(”RUNLEVEL”)`(”RUNLEVEL” 인자 관련 처리)함수를 실행하여, 런레벨을 결정한다.
- `sub_427510()`함수로 “init started: BusyBox v1.13.4 …” 로그 메시지를 출력한다.
- 인자값을 검사하여, 만약 인자가 “single”, “-s”, 혹은 “1”이라면 단일 사용자 모드로 `sub_4273D8()`을 실행하고, 그렇지 않으면 `sub_4275F0()`를 호출해 초기 자식 프로세스(쉘이나 다른 데몬)를 실행한다.
    
    ```c
    int __fastcall sub_4273D8(char a1, int a2, int a3)
    {
      int *v3; // $s1
      int i; // $s0
      int result; // $v0
      int v9; // $v0
      int v10; // $s0
    
      v3 = (int *)dword_4467A0;
      for ( i = dword_4467A0; i; i = *(_DWORD *)i )
      {
        if ( !strcmp(i + 41, a2) )
        {
          result = strcmp(i + 9, a3);
          v3 = (int *)i;
          if ( !result )
          {
            *(_BYTE *)(i + 8) = a1;
            return result;
          }
        }
        else
        {
          v3 = (int *)i;
        }
      }
      v9 = sub_404F6C(300);
      v10 = v9;
      if ( v3 )
        *v3 = v9;
      else
        dword_4467A0 = v9;
      *(_BYTE *)(v9 + 8) = a1;
      sub_42E498(v9 + 41, a2, 256);
      return sub_42E498(v10 + 9, a3, 32);
    }
    ```
    
    `sub_4273D8()`함수는 동일한 식별자a2와 값a3로 등록된 항목이 있으면 플래그만 업데이트하고, 없으면 새 항목을 추가한다.
    
    ```bash
    int sub_4275F0()
    {
    	//(변수 초기화 생략)
      v0 = sub_42D51C("/etc/inittab", sub_42F1A0);
      if ( !v0 )
      {
        sub_4273D8(32, "reboot", "");
        sub_4273D8(64, "umount -a -r", "");
        sub_4273D8(128, "init", "");
        sub_4273D8(4, "-/bin/sh", "");
        sub_4273D8(4, "-/bin/sh", "/dev/tty2");
        sub_4273D8(4, "-/bin/sh", "/dev/tty3");
        sub_4273D8(4, "-/bin/sh", "/dev/tty4");
        return sub_4273D8(1, "/etc/init.d/rcS", "");
      }
      while ( sub_42D238(v0, v8, 262148, "#:") )
      {
        v2 = (_BYTE *)v8[0];
        if ( v9 && (v3 = sub_428890("sysinit", v8[2]), v3 >= 0) )
        {
          if ( !*v9 )
          {
            v4 = *(_DWORD *)(v0 + 12);
            goto LABEL_14;
          }
          if ( *v2 )
          {
            v10 = v3;
            v5 = strncmp(v2, "/dev/", 5);
            v6 = v2 + 5;
            if ( v5 )
              v6 = v2;
            v7 = sub_428904("/dev/", v6);
            LOBYTE(v3) = v10;
            v2 = (_BYTE *)v7;
          }
          sub_4273D8((unsigned __int8)(1 << v3), v9, v2);
          if ( *v2 )
            free(v2);
        }
        else
        {
          v4 = *(_DWORD *)(v0 + 12);
    LABEL_14:
          sub_427510(3, "Bad inittab entry at line %d", v4);
        }
      }
      return sub_42D4D4(v0);
    }
    ```
    
    `sub_4275F0()`함수는 `/etc/inittab`을 열고 파일에 정의된 엔트리들을 읽어서 각 엔트리에서 지정한 초기화 명령어들을 실행(혹은 예약)한다.
    
    `/etc/inittab`파일은 아래와 같이 정의되어 있다.
    
    ```bash
    # Boot-time system configuration/initialization script.
    ::sysinit:/etc/init.d/rcS
    
    # Start an "askfirst" shell on the console (whatever that may be)
    #::askfirst:-/bin/sh
    ::respawn:-/bin/sh
    
    # Start an "askfirst" shell on /dev/tty2-4
    #tty2::askfirst:-/bin/sh
    #tty3::askfirst:-/bin/sh
    #tty4::askfirst:-/bin/sh
    ```
    
    - `sub_42D51C("/etc/inittab", sub_42F1A0)` 를 호출하여 `/etc/inittab`파일을 열고, 파일 핸들을 v0에 저장한다.
    - 파일 핸들이 0이면(파일을 열지 못하면), 기본 초기화 명령어들 `reboot`, `umount -a -r`, `init`, 그리고 여러 터미널에서 `/bin/sh`을 실행하며, 마지막으로 `/etc/init.d/rcS` 실행을 하드코딩으로 등록한 후 반환한다.
    - 파일이 존재하는 경우에는 `while ( sub_42D238(v0, v8, 262148, "#:") )` 루프를 통해 `/etc/inittab`의 각 라인을 읽어온다.
    - `sysinit`문자열이 일치하면, 명령어 실행 파일경로(v2)가 `/dev/`로 시작하는 경우 그 접두어를 제거한 후, 다시 `/dev/`를 붙여서 올바른 디바이스 경로를 만든다. 이 후 `sub_4273D8((1 << v3), v9, v2)`를 호출하여, 해당 명령어(v9)와 인자(v2)를 실행 대기상태로 등록한다.
    - 마지막으로 `sub_42D4D4(v0)`를 호출하여 파일을 닫고 함수가 종료된다.
    - 실행에 사용될 첫 번째 인자(init)을 “init”문자열로 덮어 씌우고, 나머지 인자들은 모두 0으로 초기화하여, 재사용을 막는다.
    - `sub_427D14()`함수를 여러번 호출하여(인자 1, 8, 16) 초기화 또는 타이머 설정을 실행하고, 시그널 1에 대해 `sub_428154()`를 등록한다.
    - 주기적으로  `sub_427D14(6)`을 호출하고 1초마다 `sleep()`을 호출하며, 자식 프로세스의 종료를 `wait()`을 통해 모니터링한다.
    - 자식 프로세스가 종료되면, 내부 연결 리스트(`dword_4467A0`)를 순회하며 해당 PID를 찾아 로그를 남기고, 재시작 스케줄링을 수행한다.

### 2.1.5. etc/init.d/rcS

```bash
Init Start...
sysconf wlanapp kill wlan0
sysconf wlanapp kill wlan1
```

<부팅 로그>

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

<rcS>

1. 시스템 기본 설정
    - `hostname` 을 `rlx-linux` 로 설정한다.
    - `/proc`, `/var`를 `ramfs`에 마운트한다.
    - `/var/tmp`, `/var/web`등 디렉터리를 생성한다.
2. 네트워크 설정
    - `flash get DEVICE_NAME` 명령어는 플래시 메모리에서 장치 이름을 가져온다.
    - `/etc/shadow.sample`을 비밀번호 설정하는 /var/shadow에 복사한다.
3. CINIT 플래그 확인
    
    ```bash
    if [ "$CINIT" = 1 ]; then
        startup.sh
    fi
    ```
    
    - `CINIT=1`이면 `startup.sh`을 실행한다.

### 2.1.6. bin/startup.sh

```bash
#!/bin/sh
#
# script file to startup
TOOL=flash
GETMIB="$TOOL get"
LOADDEF="$TOOL default"
LOADDEFBLUETOOTHHW="$TOOL default-bluetoothhw"
LOADDEFCUSTOMERHW="$TOOL default-customerhw"
LOADDEFSW="$TOOL default-sw"
LOADDS="$TOOL reset"
# See if flash data is valid
$TOOL virtual_flash_init
$TOOL test-hwconf
if [ $? != 0 ]; then
	echo 'HW configuration invalid, reset default!'
	$LOADDEF
fi
$TOOL test-bluetoothhwconf
if [ $? != 0 ]; then
        echo 'BLUETOOTH HW configuration invalid, reset default!'
        $LOADDEFBLUETOOTHHW
fi
$TOOL test-customerhwconf
if [ $? != 0 ]; then
        echo 'CUSTOMER HW configuration invalid, reset default!'
        $LOADDEFCUSTOMERHW
fi

$TOOL test-dsconf
if [ $? != 0 ]; then
	echo 'Default configuration invalid, reset default!'
	$LOADDEFSW
fi

$TOOL test-csconf
if [ $? != 0 ]; then
	echo 'Current configuration invalid, reset to default configuration!'
	$LOADDS
fi
$TOOL test-alignment
if [ $? != 0 ]; then
        echo 'Please refine linux/.config change offset to fit flash erease size!!!!!!!!!!!!!!!!!'
fi

# voip flash check
if [ "$VOIP_SUPPORT" != "" ]; then
$TOOL voip check
fi

if [ ! -e "$SET_TIME" ]; then
	flash settime
fi

#for auto gen env
$TOOL jcg-auto-gen

# Generate WPS PIN number
# eval `$GETMIB HW_WSC_PIN`
# JCG wangxg+-: use real MIB to check values
eval `$GETMIB HW_WLAN0_WSC_PIN`
if [ "$HW_WLAN0_WSC_PIN" = "" ]; then
        echo 'HW_WLAN0_WSC_PIN = NULL!!!!!!!!!!!!!!!!!'
	$TOOL gen-pin
fi

# Enable Multicast and Broadcast Strom control and disable it in post_startup.sh
echo "1 3" > /proc/StormCtrl

```

<startup.sh>

1. 하드웨어 및 소프트웨어 설정 확인
    
    ```bash
    $TOOL test-hwconf
    if [ $? != 0 ]; then
        echo 'HW configuration invalid, reset default!'
        $LOADDEF
    fi
    ```
    
    - 하드웨어 설정이(`hwconf`)손상되었는지 확인하고 손상되었다면 기본값으로 초기화한다.
2. 네트워크 관련 초기화
    - `flash get HW_WLAN0_WSC_PIN` 에서 무선 보안 설정을 확인한다.
    - `echo "1 3" > /proc/StormCtrl` 멀티캐스트 트래픽을 설정한다.

### 2.1.7.  bin/init.sh

```bash
-> % cat init.sh
#!/bin/sh
#
# script file to start network
#
# Usage: init.sh {gw | ap} {all | bridge | wan}
#

sysconf init $*

```

1. init.sh 실행
    
    ```bash
    sysconf init gw all
    ```
    
    - `rcs`에서 `init.sh gw all`을 실행한다.
    - `init.sh` 내부에서 `sysconf init gw all`을 실행한다.
2. sysconf 명령어 실행
    - `sysconf wlanapp kill wlan0` , `sysconf wlanapp kill wlan1` 명령어를 실행하여 초기화 한다.
    - 이후 기존 무선 네트워크 프로세스 종료 후 재설정 한다.

### 2.1.8 boa process

```bash

# Boa v0.94 configuration file
# File format has not changed from 0.93
# File format has changed little from 0.92
# version changes are noted in the comments
#
# The Boa configuration file is parsed with a custom parser.  If it
# reports an error, the line number will be provided; it should be easy
# to spot.  The syntax of each of these rules is very simple, and they
# can occur in any order.  Where possible these directives mimic those
# of NCSA httpd 1.3; I saw no reason to introduce gratuitous
# differences.

# $Id: boa.conf,v 1.3.2.6 2003/02/02 05:02:22 jnelson Exp $

# The "ServerRoot" is not in this configuration file.  It can be
# compiled into the server (see defines.h) or specified on the command
# line with the -c option, for example:
#
# boa -c /usr/local/boa

# Port: The port Boa runs on.  The default port for http servers is 80.
# If it is less than 1024, the server must be started as root.

Port 80

# Listen: the Internet address to bind(2) to.  If you leave it out,
# it takes the behavior before 0.93.17.2, which is to bind to all
# addresses (INADDR_ANY).  You only get one "Listen" directive,
# if you want service on multiple IP addresses, you have three choices:
#    1. Run boa without a "Listen" directive
#       a. All addresses are treated the same; makes sense if the addresses
#          are localhost, ppp, and eth0.
#       b. Use the VirtualHost directive below to point requests to different
#          files.  Should be good for a very large number of addresses (web
#          hosting clients).
#    2. Run one copy of boa per IP address, each has its own configuration
#       with a "Listen" directive.  No big deal up to a few tens of addresses.
#       Nice separation between clients.
# The name you provide gets run through inet_aton(3), so you have to use dotted
# quad notation.  This configuration is too important to trust some DNS.

#Listen 192.68.0.5

#  User: The name or UID the server should run as.
# Group: The group name or GID the server should run as.

#User nobody
#Group nogroup
User root
Group root

# ServerAdmin: The email address where server problems should be sent.
# Note: this is not currently used, except as an environment variable
# for CGIs.

#ServerAdmin root@localhost

# PidFile: where to put the pid of the process.
# Comment out to write no pid file.
# Note: Because Boa drops privileges at startup, and the
# pid file is written by the UID/GID before doing so, Boa
# does not attempt removal of the pid file.
# PidFile /var/run/boa.pid
PidFile /var/run/webs.pid

# ErrorLog: The location of the error log file. If this does not start
# with /, it is considered relative to the server root.
# Set to /dev/null if you don't want errors logged.
# If unset, defaults to /dev/stderr
# Please NOTE: Sending the logs to a pipe ('|'), as shown below,
#  is somewhat experimental and might fail under heavy load.
# "Usual libc implementations of printf will stall the whole
#  process if the receiving end of a pipe stops reading."
#ErrorLog "|/usr/sbin/cronolog --symlink=/var/log/boa/error_log /var/log/boa/error-%Y%m%d.log"

#ErrorLog /var/log/boa/error_log

# AccessLog: The location of the access log file. If this does not
# start with /, it is considered relative to the server root.
# Comment out or set to /dev/null (less effective) to disable.
# Useful to set to /dev/stdout for use with daemontools.
# Access logging.  
# Please NOTE: Sending the logs to a pipe ('|'), as shown below,
#  is somewhat experimental and might fail under heavy load.
# "Usual libc implementations of printf will stall the whole
#  process if the receiving end of a pipe stops reading."
#AccessLog  "|/usr/sbin/cronolog --symlink=/var/log/boa/access_log /var/log/boa/access-%Y%m%d.log"

#AccessLog /var/log/boa/access_log

# CGILog /var/log/boa/cgi_log
# CGILog: The location of the CGI stderr log file. If this does not
# start with /, it is considered relative to the server root.
# The log file would contain any contents send to /dev/stderr
# by the CGI. If this is commented out, it defaults to whatever 
# ErrorLog points.  Set to /dev/null to disable CGI stderr logging.
# Please NOTE: Sending the logs to a pipe ('|'), as shown below,
#  is somewhat experimental and might fail under heavy load.
# "Usual libc implementations of printf will stall the whole
#  process if the receiving end of a pipe stops reading."
#CGILog  "|/usr/sbin/cronolog --symlink=/var/log/boa/cgi_log /var/log/boa/cgi-%Y%m%d.log"

# CGIumask 027 (no mask for user, read-only for group, and nothing for user)
# CGIumask 027
# The CGIumask is set immediately before execution of the CGI.

# UseLocaltime: Logical switch.  Uncomment to use localtime 
# instead of UTC time
#UseLocaltime

# VerboseCGILogs: this is just a logical switch.
#  It simply notes the start and stop times of cgis in the error log
# Comment out to disable.

#VerboseCGILogs

# ServerName: the name of this server that should be sent back to 
# clients if different than that returned by gethostname + gethostbyname 

#ServerName www.your.org.here
#ServerName ""

# VirtualHost: a logical switch.
# Comment out to disable.
# Given DocumentRoot /var/www, requests on interface 'A' or IP 'IP-A'
# become /var/www/IP-A.
# Example: http://localhost/ becomes /var/www/127.0.0.1
#
# Not used until version 0.93.17.2.  This "feature" also breaks commonlog
# output rules, it prepends the interface number to each access_log line.
# You are expected to fix that problem with a postprocessing script.

#VirtualHost 

# VHostRoot: the root location for all virtually hosted data
# Comment out to disable.
# Incompatible with 'Virtualhost' and 'DocumentRoot'!!
# Given VHostRoot /var/www, requests to host foo.bar.com,
# where foo.bar.com is ip a.b.c.d,
# become /var/www/a.b.c.d/foo.bar.com 
# Hostnames are "cleaned", and must conform to the rules
# specified in rfc1034, which are be summarized here:
# 
# Hostnames must start with a letter, end with a letter or digit, 
# and have as interior characters only letters, digits, and hyphen.
# Hostnames must not exceed 63 characters in length.

#VHostRoot /var/www

# DefaultVHost
# Define this in order to have a default hostname when the client does not
# specify one, if using VirtualHostName. If not specified, the word
# "default" will be used for compatibility with older clients.

#DefaultVHost foo.bar.com

# DocumentRoot: The root directory of the HTML documents.
# Comment out to disable server non user files.

#DocumentRoot /var/www
DocumentRoot /web

# UserDir: The name of the directory which is appended onto a user's home
# directory if a ~user request is received.

UserDir public_html

# DirectoryIndex: Name of the file to use as a pre-written HTML
# directory index.  Please MAKE AND USE THESE FILES.  On the
# fly creation of directory indexes can be _slow_.
# Comment out to always use DirectoryMaker

DirectoryIndex index.html

# DirectoryMaker: Name of program used to create a directory listing.
# Comment out to disable directory listings.  If both this and
# DirectoryIndex are commented out, accessing a directory will give
# an error (though accessing files in the directory are still ok).

#DirectoryMaker /usr/lib/boa/boa_indexer

# DirectoryCache: If DirectoryIndex doesn't exist, and DirectoryMaker
# has been commented out, the the on-the-fly indexing of Boa can be used
# to generate indexes of directories. Be warned that the output is 
# extremely minimal and can cause delays when slow disks are used.
# Note: The DirectoryCache must be writable by the same user/group that 
# Boa runs as.

#DirectoryCache /var/spool/boa/dircache
DirectoryCache /tmp

# KeepAliveMax: Number of KeepAlive requests to allow per connection
# Comment out, or set to 0 to disable keepalive processing

#KeepAliveMax 1000
KeepAliveMax 0

# KeepAliveTimeout: seconds to wait before keepalive connection times out

KeepAliveTimeout 10

# MimeTypes: This is the file that is used to generate mime type pairs
# and Content-Type fields for boa.
# Set to /dev/null if you do not want to load a mime types file.
# Do *not* comment out (better use AddType!)

#MimeTypes /etc/mime.types
MimeTypes /etc/boa/mime.types

# DefaultType: MIME type used if the file extension is unknown, or there
# is no file extension.

#DefaultType text/plain
DefaultType text/html

# CGIPath: The value of the $PATH environment variable given to CGI progs.

CGIPath /bin:/usr/bin:/web/cgi-bin/

# SinglePostLimit: The maximum allowable number of bytes in 
SinglePostLimit 0x800000 
# a single POST.  Default is normally 1MB.

# AddType: adds types without editing mime.types
# Example: AddType type extension [extension ...]

# Uncomment the next line if you want .cgi files to execute from anywhere
AddType application/x-httpd-cgi cgi
AddType application/x-httpd-cgi php

# Redirect, Alias, and ScriptAlias all have the same semantics -- they
# match the beginning of a request and take appropriate action.  Use
# Redirect for other servers, Alias for the same server, and ScriptAlias
# to enable directories for script execution.

# Redirect allows you to tell clients about documents which used to exist in
# your server's namespace, but do not anymore. This allows you to tell the
# clients where to look for the relocated document.
# Example: Redirect /bar http://elsewhere/feh/bar

# Aliases: Aliases one path to another.
# Example: Alias /path1/bar /path2/foo

# Alias /doc /usr/doc

# ScriptAlias: Maps a virtual path to a directory for serving scripts
# Example: ScriptAlias /htbin/ /www/htbin/

ScriptAlias /cgi-bin/ /web/cgi-bin/
```

boa.conf

1. boa 설정 파일 (`boa.conf`)로드
    
    ```bash
    Port 80
    DocumentRoot /web
    ScriptAlias /cgi-bin/ /web/cgi-bin/
    ```
    
    - 80번 포트에서 웹 서버 실행한다.
    - `/web` 디렉터리를 루트로 사용한다.
    - CGI 실행가능하다.(`/cgi-bin/`)
2. boa 프로세스 시작
    
    ```bash
    boa
    ```
    
    - `/etc/init.d/rcS`에서 boa 실행한다.
    - 웹 인터페이스 활성화


## 2.2. MEX605 부팅 프로세스 분석

---


### 2.2.1. 부트로더 단계

```
[  230.986131] WiFi@C14L1,cache_dpp_frame_rx_event() 1461: wdev->wdev_idx 0 event->ifindex 13,store fid 0
[  230.995625] 7981@C14L1,RTMPAPQueryInformation() 11784: GET_DPP_FRAME wdev idx 0
[  231.002949] WiFi@C14L1,wext_send_dpp_cached_frame() 26327: event->ifindex 13
��������
F0: 102B 0000
FA: 1040 0000
FA: 1040 0000 [0200]
F9: 0000 0000
V0: 0000 0000 [0001]
00: 0000 0000
BP: 2400 0041 [0000]
G0: 1190 0000
EC: 0000 0000 [1000]
T0: 0000 024F [010F]
Jump to BL
```

칩 내부에 저장된 Boot ROM이 실행되며, 최초 하드웨어를 초기화한다.

이후에 2차 부트로더(BL2)를 로드하고 실행한다.

```
NOTICE:  BL2: v2.7(release):
NOTICE:  BL2: Built : 20:12:36, Aug  1 2023
NOTICE:  WDT: disabled
NOTICE:  EMI: Using DDR3 settings

dump toprgu registers data: 
1001c000 | 00000000 0000ffe0 00000000 00000000
1001c010 | 00000fff 00000000 00f00000 00000000
1001c020 | 00000000 00000000 00000000 00000000
1001c030 | 003c0003 003c0003 00000000 00000000
1001c040 | 00000000 00000000 00000000 00000000
1001c050 | 00000000 00000000 00000000 00000000
1001c060 | 00000000 00000000 00000000 00000000
1001c070 | 00000000 00000000 00000000 00000000
1001c080 | 00000000 00000000 00000000 00000000

dump drm registers data: 
1001d000 | 00000000 00000000 00000000 00000000
1001d010 | 00000000 00000000 00000000 00000000
1001d020 | 00000000 00000000 00000000 00000000
1001d030 | 00a003f1 000000ff 00100000 00000000
1001d040 | 00027e71 000200a0 00020303 000000ff
1001d050 | 00000000 00000000 00000000 00000000
1001d060 | 00000002 00000000 00000000 00000000
drm: 500 = 0xc 
[DDR Reserve] ddr reserve mode not be enabled yet
DDR RESERVE Success 0
[EMI] ComboMCP not ready, using default setting
BYTE_swap:0 
BYTE_swap:0 
Window Sum 520, worse bit 7, min window 60
Window Sum 524, worse bit 10, min window 64
Window Sum 256, worse bit 1, min window 28
Window Sum 274, worse bit 11, min window 28
Window Sum 280, worse bit 1, min window 32
Window Sum 286, worse bit 11, min window 28
Window Sum 300, worse bit 7, min window 34
Window Sum 302, worse bit 11, min window 30
Window Sum 312, worse bit 5, min window 36
Window Sum 312, worse bit 11, min window 32
Window Sum 328, worse bit 5, min window 38
Window Sum 332, worse bit 11, min window 36
Window Sum 342, worse bit 5, min window 40
Window Sum 346, worse bit 11, min window 38
Window Sum 352, worse bit 1, min window 42
Window Sum 358, worse bit 11, min window 40
Window Sum 372, worse bit 1, min window 44
Window Sum 366, worse bit 11, min window 40
Window Sum 374, worse bit 3, min window 44
Window Sum 380, worse bit 11, min window 42
Window Sum 380, worse bit 1, min window 46
Window Sum 388, worse bit 11, min window 44
Window Sum 384, worse bit 3, min window 46
Window Sum 394, worse bit 11, min window 46
NOTICE:  EMI: Detected DRAM size: 256MB
NOTICE:  EMI: complex R/W mem test passed
NOTICE:  CPU: MT7981 (1300MHz)
NOTICE:  SPI_NAND parses attributes from parameter page.
NOTICE:  SPI_NAND Detected ID 0xc8
NOTICE:  Page size 2048, Block size 131072, size 134217728
NOTICE:  Initializing NMBM ...
NOTICE:  Signature found at block 1023 [0x07fe0000]
NOTICE:  First info table with writecount 0 found in block 960
NOTICE:  Second info table with writecount 0 found in block 963
NOTICE:  NMBM has been successfully attached in read-only mode
NOTICE:  BL2: Booting BL31
NOTICE:  BL31: v2.7(release):
NOTICE:  BL31: Built : 20:12:38, Aug  1 2023
NOTICE:  Hello BL31!!!
```

플래시 메모리에서 2차 부트로더가 실행된 모습이다. 이 단계에서 추가적인 하드웨어 초기화가 이루어진다.

BL2는 실행 이후 BL31을 실행시킨다. BL31은 ARM 기반 시스템에서 보안 부팅 관련 처리를 위해 실행되는 ARM Trusted Firmware이다.

```
U-Boot 2022.10 (Aug 01 2023 - 20:12:09 +0900)

CPU:   MediaTek MT7981
Model: mt7981-rfb
DRAM:  256 MiB
Core:  32 devices, 12 uclasses, devicetree: separate

Initializing NMBM ...
spi-nand: spi_nand spi_nand@0: GigaDevice SPI NAND was found.
spi-nand: spi_nand spi_nand@0: 128 MiB, block size: 128 KiB, page size: 2048, OOB size: 64
Could not find a valid device for nmbm0
Signature found at block 1023 [0x07fe0000]
First info table with writecount 0 found in block 960
Second info table with writecount 0 found in block 963
NMBM has been successfully attached 

Loading Environment from UBI... ubi0: attaching mtd6
ubi0: scanning is finished
### drivers/mtd/ubi/build.c:1016:ubi_attach_mtd_dev ubi_name: autoresize_vol_id:-1
ubi0: attached mtd6 (name "ubi", size 114 MiB)
ubi0: PEB size: 131072 bytes (128 KiB), LEB size: 126976 bytes
ubi0: min./max. I/O unit sizes: 2048/2048, sub-page size 2048
ubi0: VID header offset: 2048 (aligned 2048), data offset: 4096
ubi0: good PEBs: 916, bad PEBs: 0, corrupted PEBs: 0
ubi0: user volume: 8, internal volumes: 1, max. volumes count: 128
ubi0: max/mean erase counter: 4/1, WL threshold: 4096, image sequence number: 1687227669
ubi0: available PEBs: 491, total reserved PEBs: 425, PEBs reserved for bad PEB handling: 19
Read 524288 bytes from volume u-boot-env to 000000004f77e240
OK
In:    serial@11002000
Out:   serial@11002000
Err:   serial@11002000
Net:   
Warning: ethernet@15100000 (eth0) using random MAC address - 2a:3a:1e:12:ac:81
eth0: ethernet@15100000
*** U-Boot Boot Menu ***Press UP/DOWN to move, ENTER to select, ESC/CTRL+C to quit1. Startup system (Default)2. Upgrade firmware3. Upgrade ATF BL24. Upgrade ATF FIP5. Upgrade single image6. Load image0. U-Boot consoleHit any key to stop autoboot: 4 Hit any key to stop autoboot: 3 Hit any key to stop autoboot: 2 Hit any key to stop autoboot: 1 UBI partition 'ubi' already selected
Trying to boot from image slot 1
Read 64 bytes from volume kernel2 to 0000000046000000
Read 2826211 bytes from volume kernel2 to 0000000046000000
## Checking hash(es) for FIT Image at 46000000 ...
   Hash(es) for Image 0 (kernel-1): crc32+ sha1+ 
   Hash(es) for Image 1 (fdt-1): crc32+ sha1+ 
Read 48 bytes from volume rootfs2 to 00000000462b1fe4
Read 13123000 bytes from volume rootfs2 to 00000000462b1fe4
   Hash(es) for rootfs: crc32+ sha1+ 
Firmware integrity verification passed
## Loading kernel from FIT Image at 46000000 ...
   Using 'config-1' configuration
   Trying 'kernel-1' kernel subimage
     Description:  ARM64 OpenWrt Linux-5.4.225
     Type:         Kernel Image
     Compression:  lzma compressed
     Data Start:   0x460000e8
     Data Size:    2805983 Bytes = 2.7 MiB
     Architecture: AArch64
     OS:           Linux
     Load Address: 0x48080000
     Entry Point:  0x48080000
     Hash algo:    crc32
     Hash value:   5ba0875e
     Hash algo:    sha1
     Hash value:   7c8e1c2914276319fb804d9071f08bba6e03f8db
   Verifying Hash Integrity ... crc32+ sha1+ OK
## Loading fdt from FIT Image at 46000000 ...
   Using 'config-1' configuration
   Trying 'fdt-1' fdt subimage
     Description:  ARM64 OpenWrt mt7981-spim-nand-rfb device tree blob
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x462ad30c
     Data Size:    18176 Bytes = 17.8 KiB
     Architecture: AArch64
     Hash algo:    crc32
     Hash value:   1a7b4cb5
     Hash algo:    sha1
     Hash value:   d39ab6c6eeff4c23f1485a99e306b22b00b8ca5c
   Verifying Hash Integrity ... crc32+ sha1+ OK
   Booting using the fdt blob at 0x462ad30c
   Uncompressing Kernel Image
   Loading Device Tree to 000000004f7f1000, end 000000004f7f86ff ... OK
```

BL2(및 BL31)부팅이 완료되면 U-Boot가 실행된다.

U-Boot는 커널 이미지와 디바이스 트리, 펌웨어 파라미터 등을 플래시에서 메모리로 로드한다.

### 2.2.2. 커널 부팅 단계

```
Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[    0.000000] Linux version 5.4.225 (yun@ubuntu) (gcc version 8.4.0 (OpenWrt GCC 8.4.0 unknown)) #0 SMP Tue Jun 20 02:21:09 2023
[    0.000000] Machine model: MediaTek MT7981 RFB
(중략)
[    1.536486] VFS: Mounted root (squashfs filesystem) readonly on device 254:0.
[    1.543841] Freeing unused kernel memory: 384K
[    1.563770] Run /sbin/init as init process
[    1.761249] init: Console is alive
[    1.764764] init: - watchdog -
[    2.114245] kmodloader: loading kernel modules from /etc/modules-boot.d/*
[    2.145464] conninfra@(mtk_conninfra_drv_init:644) Before platform_driver_register
```

부트로더 단계가 끝나면 커널 부팅이 시작된다. 

이 구간에서 CPU, 메모리, 인터럽트, 클럭 등의 하드웨어 초기화와 기본 드라이버들이 로드된다.

이후 `Run /sbin/init as init process` 라는 메시지가 나타나며, init 프로세스가 실행된다.

### 2.2.3. init process

IDA를 통해 `/sbin/init`바이너리를 디스어셈블한 결과이다.

```c
  ulog_open(1LL, 24LL, "init");
  v0 = (void (__fastcall *)(int, const struct sigaction *, struct sigaction *))sigaction_0;
  ((void (__fastcall *)(int, const struct sigaction *, struct sigaction *))sigaction_0)(
    15,
    (const struct sigaction *)sub_2A7C,
    0LL);
  v0(10, (const struct sigaction *)sub_2A7C, 0LL);
  v0(12, (const struct sigaction *)sub_2A7C, 0LL);
  v0(30, (const struct sigaction *)sub_2A7C, 0LL);
```

```c
void __noreturn sub_2A7C()
{
  FILE *v0; // x19

  v0 = *(FILE **)stderr_0;
  fputs("reboot\n", *(FILE **)stderr_0);
  fflush(v0);
  sync();
  sleep(2u);
  reboot(19088743);
  while ( 1 )
    ;
}
```

`ulog_open()` 함수를 통해 로깅 시스템을 열고, `sigaction()`을 사용해 시그널(15, 10, 12, 30)에 대해 동일한 핸들러 함수를 등록하여 시스템 재부팅 관련 시그널을 처리하도록 설정한다.

```c
if ( get_pid() == 1 )
  {
    v1 = umask(0);
    v2 = stat("/.dockerenv", (struct stat *)&ptr);
    if ( !get_env("container") && v2 )
    {
      v3 = (void (__fastcall *)(const char *, const char *, const char *, unsigned __int64, const void *))mount_0;
      mount("proc", "/proc", "proc", 0x40EuLL, 0LL);
      v3("sysfs", "/sys", "sysfs", 0x40EuLL, 0LL);
      v3("cgroup2", "/sys/fs/cgroup", "cgroup2", 0x20000EuLL, "nsdelegate");
      v3("tmpfs", "/dev", "tmpfs", 0x402uLL, "mode=0755,size=512K");
      symlink("/tmp/shm", "/dev/shm");
      mkdir("/dev/pts", 0x1EDu);
      v3("devpts", "/dev/pts", "devpts", 0x40AuLL, "mode=600");
      v4 = (void (__fastcall *)(const char *))chdir_0;
      if ( !chdir("/dev") )
      {
        v5 = malloc(3uLL);
        *v5 = 42;
        strcpy(v5 + 1, "*");
        ptr.tv_sec = (__time_t)v5;
        qword_14160 = (__int64)&ptr;
        dword_14168 = 1;
        dword_14024 = 384;
        sub_2450(1LL);
        sub_2450(0LL);
        free((void *)ptr.tv_sec);
        v4("/");
      }
      mknod("/dev/null", 0x1B6u, 0x103uLL);
    }
    if ( stat("/dev/console", (struct stat *)&ptr) )
    {
      ulog(3LL, "Failed to stat %s: %m\n", "/dev/console");
    }
    else if ( (unsigned int)sub_2154("/dev/console") )
    {
      ulog(3LL, "Failed to setup i/o redirection\n");
    }
    else
    {
      v9 = (void (*)(int, int, ...))off_13FE0;
      v10 = off_13FE0(2, 3);
      v9(2, 4, v10 | 0x800u);
    }
    mount("tmpfs", "/tmp", "tmpfs", 0x406uLL, "mode=01777");
    v6 = (void (__fastcall *)(const char *, __mode_t))mkdir;
    mkdir("/tmp/shm", 0x3FFu);
    v6((const char *)&loc_2DA4, 0x1EDu);
    v6("/tmp/lock", 0x1EDu);
    v6("/tmp/state", 0x1EDu);
    umask(v1);
    setenv("PATH", "/usr/sbin:/usr/bin:/sbin:/bin", 1);
    ulog(6LL, "Console is alive\n");
  }
```

프로세스 ID가 1인 경우 아래와 같은 과정에 따라 기본 시스템 환경을 구성한다.

- 현재 umask 값을 0으로 설정하고 나중에 복원할 값을 저장한다.
- `stat(”/.dockererenv”, …)`, `get_env("container")`를 통해 컨테이너 환경 여부를 확인한 뒤, 컨테이너가 아닌 정상 환경이라면 필수 파일 시스템들을 마운트 한다. 이 과정에서 “proc”를 `/proc`에, “sysfs”을 `/sys`에, “cgroup2”를 `/sys/fs/cgroup`에, “tmpfs”을 `/dev`에 마운트한다. 그 후 `/dev/pts`를 생성하고 “devpts”를 마운트하며, `/dev/shm` symbolic link를 생성한다.
- `/dev`디렉터리로 이동한 후 `malloc(3)`으로 임시 메모리를 할당하여 아래와 같은 작업을 처리한다.
    
    ```c
    DIR *__fastcall sub_2450(unsigned __int8 a1)
    {
      //(변수 초기화 부분 생략)
      v1 = a1;
      v2 = "/sys/dev/char";
      if ( a1 )
        v2 = "/sys/dev/block";
      result = opendir_0(v2);
      if ( result )
      {
        v4 = result;
        v5 = &byte_151F0[sprintf(byte_151F0, "%s/", v2)];
        if ( v1 )
          v6 = 24576;
        else
          v6 = 0x2000;
        v21 = v6;
        v7 = "character";
        if ( v1 )
          v7 = "block";
        v22 = v7;
        v23 = (struct dirent *(__fastcall *)(DIR *))readdir;
    LABEL_10:
        while ( 1 )
        {
          v8 = v23(v4);
          if ( !v8 )
            return (DIR *)closedir(v4);
          d_type = v8->d_type;
          v24 = 0;
          v25 = 0;
          if ( d_type == 10 )
          {
            d_name = v8->d_name;
            if ( sscanf(v8->d_name, "%d:%d", (unsigned int)&v25, (unsigned int)&v24) == 2 )
            {
              strcpy(v5, d_name);
              v11 = readlink(byte_151F0, byte_141F0, 0x1000uLL);
              if ( v11 > 0 )
              {
                byte_141F0[v11] = 0;
                v12 = (int (__fastcall *)(const char *, const char *, int))off_13F28;
                v13 = 0LL;
                while ( dword_14168 > (int)v13 )
                {
                  v14 = *(const char **)(qword_14160 + 8 * v13++);
                  if ( !v12(v14, byte_141F0, 0) )
                  {
                    v15 = strrchr(byte_141F0, 47);
                    if ( v15 )
                    {
                      v16 = v15 + 1;
                      v18 = v24;
                      v17 = v25;
                      v19 = umask(0);
                      v20 = v21 | dword_14024;
                      if ( (unsigned int)dword_1415C > 3 )
                        ulog(5LL, "Creating %s device %s(%d,%d)\n", v22, v16, v17, v18);
                      mknod(
                        v16,
                        v20,
                        (v17 << 32) & 0xFFFFF00000000000LL | ((__int64)v18 << 12) & 0xFFFFFF00000LL | ((unsigned __int64)(v17 & 0xFFF) << 8) | (unsigned __int8)v18);
                      umask(v19);
                    }
                    goto LABEL_10;
                  }
                }
              }
            }
          }
        }
      }
      return result;
    }
    ```
    
    위 코드를 통해 `/sys/dev/char`, `/sys/dev/block`디렉터리에서 “major:minor” 형식의 심볼릭 링크를 찾아, 해당 심볼릭 링크의 대상 경로가 미리 정의된 dtb와 일치하는지 확인 후 해당 대상의 basename을 이용하여 올바른 디바이스 번호(major, minor)를 인코딩한 디바이스 노드를 생성한다.
    
- 실행 후 `/dev/null`노드를 생성하고, `/dev/console`에 대한 상태를 확인 후 I/O 리다이렉션을 설정한다.
- `/tmp`를 “tmpfs”로 마운트 하고, `/tmp/shm`, `/tmp/lock`, `/tmp/state` 등의 디렉터리 생성 후 umask를 설정한다.
- PATH 환경 변수를 설정하여 “Console is alive” 메시지를 출력한다.

```
[    1.761249] init: Console is alive
```

```c
  if ( sub_2008(&ptr) )
  {
  //(변수 초기화 부분 생략)
    v7 = strtol((const char *)&ptr, 0LL, 10);
    if ( (unsigned __int64)(v7 + 0x7FFFFFFFFFFFFFFFLL) <= 0xFFFFFFFFFFFFFFFDLL )
      dword_1415C = v7;
  }
  qword_141B0 = (__int64)sub_2420;
  v8 = get_env("WDTFD");
  if ( (dword_14020 & 0x80000000) == 0 )
    goto LABEL_14;
  if ( v8 )
  {
    if ( (unsigned int)dword_1415C > 1 )
      ulog(5LL, "Watchdog handover: fd=%s\n", v8);
    dword_14020 = atoi(v8);
    unsetenv("WDTFD");
  }
  else
  {
    dword_14020 = unsetenv("/dev/watchdog", 1);
  }
  if ( (dword_14020 & 0x80000000) == 0 )
  {
LABEL_14:
    ulog_0(6LL, "- watchdog -\n");
    if ( (dword_14020 & 0x80000000) == 0 )
      ioctl_0(dword_14020, 0xC0045706uLL, &dword_14028);
    sub_2420(&unk_14198);
    if ( (unsigned int)dword_1415C > 3 )
      ulog_0(5LL, "Opened watchdog with timeout %ds\n", dword_14028);
  }
```

- `sub_2008(&ptr)` 호출
    
    ```c
    char *__fastcall sub_2008(char *a1)
    {
      v2 = unsetenv("/proc/cmdline", 0);
      v3 = read(v2, v13, 0x800uLL);
      close(v2);
      if ( v3 > 0 )
      {
        v5 = strtok_r_0;
        v13[v3] = 0;
        v6 = v5(v13, " \t\n", &save_ptr);
        v7 = (char *(__fastcall *)(const char *, int))off_13FC0;
        for ( i = v6; i; i = strtok_r(0LL, " \t\n", &save_ptr) )
        {
          v9 = v7(i, 61);
          v10 = v9;
          if ( v9 )
          {
            v11 = v9 - i;
            if ( !strcmp("init_debug", i, v9 - i) && !aInitDebug[v11] )
            {
              strcpy(a1, v10 + 1, 0x14uLL);
              a1[19] = 0;
              return a1;
            }
          }
        }
      }
      return 0LL;
    }
    ```
    
    `sub_2008()` 함수는 `/proc/cmdline`(부팅 커맨드 라인)에서 “init_debug=값”과 같이 지정된 옵션을 찾아 그 값을 a1 버퍼에 저장하고 반환한다. 값이 성공적으로 반환되면 &ptr문자열을 숫자로 변환하여 전역변수 `dword_1415C`에 저장한다.
    
- 환경변수 “WDTFD”를 읽어 `watchdog` 파일 디스크립터를 가져오고, 없으면 `/dev/watchdog`를 통해 `watchdog`를 초기화한다.
- 이후 `ioctl()`등을 통해 `watchdog` 타임아웃을 설정하고, 관련함수(`sub_2420()`)를 호출한다. `sub_2420()`함수는 `sub_23A4()`를 호출한다.
    
    ```c
    __int64 sub_23A4()
    {
      __int64 result; // x0
    
      if ( (unsigned int)dword_1415C > 3 )
        ulog(5LL, "Ping\n");
      result = (unsigned int)dword_14020;
      if ( (dword_14020 & 0x80000000) == 0 )
      {
        result = write(dword_14020, "X", 1uLL);
        if ( result < 0 )
          return ulog(3LL, "WDT failed to write: %m\n");
      }
      return result;
    }
    ```
    
    이 함수는 `watchdog`를 “ping”하여 시스템이 살아있음을 `watchdog`에 알리고, 만약 쓰기 실패 시 적절한 오류를 출력한다. 이 함수로 `watchdog` 핸들링을 수행하며, 타임아웃 값이 3초보다 크다면 로그를 남긴다.
    

```c
 	v11 = fork();
  v12 = v11;
  if ( !v11 )
  {
    ptr = off_14030;
    *(_QWORD *)&v24 = qword_14040;
    if ( (unsigned int)dword_1415C <= 2 )
      sub_2154("/dev/null");
    execvp_0((const char *)ptr.tv_sec, (char *const *)&ptr);
    ulog_0(3LL, "Failed to start kmodloader: %m\n");
LABEL_36:
    exit_0(1);
  }
  if ( v11 <= 0 )
  {
    ulog_0(3LL, "Failed to start kmodloader instance: %m\n");
  }
  else
  {
    v15 = (__pid_t (__fastcall *)(__pid_t, int *, int))waitpid_0;
    v16 = 1200;
    v17 = (int (__fastcall *)(const struct timespec *, struct timespec *))nanosleep_0;
    ptr.tv_sec = 0LL;
    ptr.tv_nsec = 10000000LL;
    do
    {
      if ( v15(v12, 0LL, 1) > 0 )
        break;
      v18 = v17(&ptr, 0LL);
      sub_23A4(v18);
      --v16;
    }
    while ( v16 );
  }
```

- `fork()`를 통해 자식 프로세스를 생성한다.
- `sub_2154`
    
    ```c
    __int64 __fastcall sub_2154(char *file)
    {
      //(변수 초기화 부분 생략)
      v2 = (int (__fastcall *)(int, int))dup2_0;
      v3 = (void (__fastcall *)(int))close_0;
      v4 = 0;
      v5 = 0LL;
      v12[0] = stdin[0];
      v12[1] = stdout[0];
      v12[2] = stderr_1;
      do
      {
        v6 = v5 != 0;
        if ( *file == 47 )
        {
          v10 = unsetenv_1(file, v6);
        }
        else
        {
          v7 = unsetenv_1("/dev", 2113536);
          v8 = v7;
          if ( v7 < 0 )
            goto LABEL_4;
          v10 = openat_0(v7, file, v6);
          close_0(v8);
        }
        if ( v10 < 0 )
        {
          if ( !strcmp_1(file, "/dev/null") )
            goto LABEL_4;
          v10 = unsetenv_1("/dev/null", v6);
          if ( v10 < 0 )
            goto LABEL_4;
        }
        v11 = v2(v10, v5);
        if ( v10 > 2 )
          v3(v10);
        if ( v11 < 0 )
        {
    LABEL_4:
          v4 = -1;
          ulog_0(3LL, "Failed to redirect %s to %s: %m\n", (const char *)v12[v5], file);
        }
        ++v5;
      }
      while ( v5 != 3 );
      return v4;
    }
    ```
    
    `sub_2154` 함수에서는 `/dev`하위 파일을 열어 표준 입력, 출력, 오류 스트림을 모두 해당 파일로 리다이렉션함으로써, 프로세스의 기본 I/O를 재설정한다. 이 작업을 통해 데몬 프로세스나 초기화 과정에서 표준 입출력을 `/dev/null` 파일로 보낼 수 있다.
    
- 자식 프로세스에서는 미리 준비된 인자(`off_14030`)를 이용하여 준비된 인자를 사용해 `execvp()`로 kmodloader를 실행한다. 실행 실패시에는 로그를 남기고 종료한다.
- 부모 프로세스에서는 `waitpid()`와 `nanosleep()`을 이용해 자식 프로세스의 종료를 기다린다.

```c
uloop_init();
  file[0] = aBinSh[0];
  file[1] = aEtcPreinit;
  file[2] = *(char **)algn_14058;
  v13 = (void (*)(__int64, const char *, ...))ulog;
  ptr = *(struct timespec *)aSbinProcd_0;
  v24 = *(_OWORD *)&aEtcHotplugPrei_0;
  ulog(6LL, (const char *)&loc_2E98);
  qword_14188 = (__int64)sub_2000;
  v14 = fork();
  dword_14190 = v14;
  if ( !v14 )
  {
    execvp((const char *)ptr.tv_sec, (char *const *)&ptr);
    v13(3LL, "Failed to start plugd: %m\n");
    goto LABEL_36;
  }
  if ( v14 <= 0 )
  {
    v13(3LL, "Failed to start new plugd instance: %m\n");
  }
  else
  {
    uloop_process_add(&unk_14170);
    setenv("PREINIT", "1", 1);
    v20 = creat("/tmp/.preinit", 0x180u);
    if ( v20 < 0 )
      v13(3LL, "Failed to create sentinel file: %m\n");
    else
      close(v20);
    qword_141E0 = (__int64)sub_26C0;
    v21 = fork();
    dword_141E8 = v21;
    if ( !v21 )
    {
      execvp(file[0], file);
      ulog(3LL, "Failed to start preinit: %m\n");
      goto LABEL_36;
    }
    if ( v21 <= 0 )
    {
      ulog(3LL, "Failed to start new preinit instance: %m\n");
    }
    else
    {
      uloop_process_add(&unk_141C8);
      if ( (unsigned int)dword_1415C > 3 )
        ulog(5LL, "Launched preinit instance, pid=%d\n", dword_141E8);
    }
  }
  uloop_run_timeout(0xFFFFFFFFLL);
  return 0LL;
}
```

- `uloop_init()`으로 이벤트 루프를 초기화한다.
- 미리 정의된 파일 배열에 `/bin/sh`, `/etc/preinit` 등의 인자들을 설정한다.
- 이어서 또다른 `fork()`를 통해 plugd 프로세스를 실행한다. 자식에서 `execvp()`로 plugd를 실행하고, 실패 시 로그를  남기고 종료한다. 부모는 실패 여부를 확인하고, uloop에 해당 프로세스를 등록한다.
- 환경변수 “PREINIT”을 “1”로 설정하고, `/tmp/.preinit`이라는 “센티넬 파일”을 생성하여 preinit의 실행을 알린다.
- `fork()`를 호출하여 preinit 프로세스를 실행한다. 자식에서는 파일 배열의 첫 번째 항목(`/bin/sh`)을 실행하고, 실패시 로그를 남기고 종료한다. 성공하면 해당 프로세스를 uloop에 등록하고, 필요 시 로그를 출력한다.
- 이후 `uloop_run_timeout()`을 호출하여 무한 대기 상태의 uloop 이벤트 루프를 실행함으로써, 이후의 이벤트를 처리한다.

### 2.2.4. preinit

```bash
#!/bin/sh
# Copyright (C) 2006-2016 OpenWrt.org
# Copyright (C) 2010 Vertical Communications

[ -z "$PREINIT" ] && exec /sbin/init

export PATH="/usr/sbin:/usr/bin:/sbin:/bin"

. /lib/functions.sh
. /lib/functions/preinit.sh
. /lib/functions/system.sh

boot_hook_init preinit_essential
boot_hook_init preinit_main
boot_hook_init failsafe
boot_hook_init initramfs
boot_hook_init preinit_mount_root

for pi_source_file in /lib/preinit/*; do
	. $pi_source_file
done

boot_run_hook preinit_essential

pi_mount_skip_next=false
pi_jffs2_mount_success=false
pi_failsafe_net_message=false

boot_run_hook preinit_main

```

`/etc/preinit`코드이다. 아래와 같은 작업을 진행한다.

- `/sbin/init` 실행한다.
- `boot_hook_init`함수를 사용해 preinit 훅(`preinit_essential`, `preinit_main`, `failsafe`, `initramfs`, `preinit_mount_root`)을 초기화한다.
- `/lib/preinit/` 디렉터리에 있는 모든 스크립트를 순차적으로 실행한다.
    
    ![image.png](img/image1.png?raw=true)
    
    `/lib/preinit/` 디렉터리에 있는 하위 스크립트들의 모습이다.
    
- `preinit_essential` 훅을 실행하여 기본적인 시스템 환경을 구성하고, 이후 몇 가지 부팅 관련 플래그(`pi_mount_skip_next`, `pi_jffs2_mount_success`, `pi_failsafe_net_message`)를 초기화하고, `preinit_main` 훅을 실행하여 나머지 초기화 작업을 진행한다.

### 2.2.5. uloop callback

```c
__int64 sub_26C0()
{
  //(변수 초기화 부분 생략)
  v0 = (const char *)((__int64 (*)(void))sub_2350)();
  file[0] = "/sbin/procd";
  file[1] = 0LL;
  if ( dword_14190 > 0 )
    kill(dword_14190, 9);
  unsetenv("PREINIT");
  unlink("/tmp/.preinit");
  lineptr = 0LL;
  v18 = 0LL;
  v19 = 0LL;
  if ( !chdir("/") )
  {
    v1 = fopen("/tmp/sysupgrade", "r");
    v2 = v1;
    if ( v1 )
    {
      v3 = getdelim_0;
      n = 0LL;
      if ( (getdelim_0(&lineptr, &n, 0, v1) & 0x8000000000000000LL) == 0 )
      {
        n = 0LL;
        if ( (v3(&v18, &n, 0, v2) & 0x8000000000000000LL) == 0 )
        {
          n = 0LL;
          if ( (v3(&v19, &n, 0, v2) & 0x8000000000000000LL) == 0 )
          {
            v4 = fclose(v2);
            v5 = lineptr;
            v6 = v18;
            v7 = v19;
            v8 = (const char *)sub_2350(v4);
            argv = "/sbin/upgraded";
            v23 = 0LL;
            v24 = 0LL;
            v25 = 0LL;
            v9 = chroot(v5);
            v10 = *(FILE **)stderr_0;
            if ( v9 < 0 )
            {
              fputs("Failed to chroot for upgraded exec.\n", *(FILE **)stderr_0);
            }
            else
            {
              v23 = v6;
              v24 = v7;
              if ( v8 )
              {
                sub_22E0(0LL);
                setenv("WDTFD", v8, 1);
              }
              execvp(argv, &argv);
              v12 = (void (__fastcall *)(const char *, FILE *))fputs;
              fputs("Failed to exec upgraded.\n", v10);
              unsetenv("WDTFD");
              sub_22E0(1LL);
              if ( chroot(".") < 0 )
              {
                v12("Failed to reset chroot, exiting.\n", v10);
                exit(1);
              }
            }
            for ( i = (void (__fastcall *)(unsigned int))sleep_0; ; i(1u) )
              ;
          }
        }
      }
      fclose(v2);
      v13 = free_0;
      free(lineptr);
      v13(v18);
      v13(v19);
    }
  }
  if ( (unsigned int)dword_1415C > 1 )
    ulog(5LL, "Exec to real procd now\n");
  if ( v0 )
    setenv("WDTFD", v0, 1);
  v14 = fopen("/tmp/debug_level", "r");
  LODWORD(argv) = 0;
  v15 = v14;
  if ( v14 )
  {
    if ( off_13E28(v14, "%d", (unsigned int)&argv) == -1 )
      ulog(3LL, "failed to read debug level: %m\n");
    fclose(v15);
    unlink("/tmp/debug_level");
    if ( (unsigned int)((_DWORD)argv - 1) <= 3 )
      dword_1415C = (int)argv;
  }
  if ( dword_1415C )
  {
    off_13DF8((char *)&argv, 2uLL, "%d", dword_1415C);
    setenv("DBGLVL", (const char *)&argv, 1);
  }
  return execvp(file[0], file);
}
```

preinit 프로세스가 종료되면, 콜백 포인터로 저장된 `sub_26C0`이 실행된다.
- 이전에 실행된 preinit 프로세스가 있다면 `kill()`을 통해 종료한다. 환경 변수 `PREINIT`을 unset하고, `/tmp/.preinit` 센티넬 파일을 삭제하여 preinit모드를 해제한다.
- 현재 디렉터리를 `/`로 변경하고, `/tmp/sysupgrade`파일을 열어 업그레이드 관련 설정값을 읽어온다. 읽은 값들을 이용해 `chroot`를 실행하고, 성공 시 환경변수 `WDTFD`를 설정한 후 `/sbin/upgraded`를 `execvp()`로 실행한다. 만약 `chroot`나 `execvp()`가 실패한다면 오류 메시지를 출력하고, 무한 루프로 대기한다.
- 업그레이드 모드가 아니면, `/tmp/debug_level` 파일을 통해 디버그 레벨을 읽어와 환경 변수 `DBGLVL`을 설정한다. 그 후, file[0]에 지정된 `/sbin/procd`를 `execvp()`로 실행하여 실제 프로세스 데몬으로 전환한다.

### 2.2.6. /sbin/procd

procd는 다음과 같은 과정으로 동작한다.

```c
__int64 __fastcall sub_5998(int a1, const char **a2)
{
  const char *v4; // x0
  int (__fastcall *v5)(int, char *const *, const char *); // x19
  unsigned int v6; // w21
  const char **v7; // x25
  int v8; // w0
  void (__fastcall *v9)(int, const struct sigaction *, struct sigaction *); // x19
  __int64 v11; // x19

  v4 = getenv("DBGLVL");
  if ( v4 )
  {
    dword_27D60 = atoi(v4);
    unsetenv("DBGLVL");
  }
```

먼저 환경 변수 `DBGLVL`을 읽어 문자열을 정수로 변환하여 전역변수에 저장한 후, 해당 변수를 unset한다.

```c
  v5 = (int (__fastcall *)(int, char *const *, const char *))getopt_0;
  v6 = 1;
  v7 = (const char **)optarg_0;
  while ( 1 )
  {
    v8 = v5(a1, (char *const *)a2, "d:s:h:S");
    if ( v8 == -1 )
      break;
    if ( v8 == 100 )
    {
      dword_27D60 = atoi(*v7);
    }
    else if ( v8 > 100 )
    {
      if ( v8 == 104 )
      {
        v11 = *(_QWORD *)optarg_0;
        unloop_init();
        sub_6500(v11);
        uloop_run_timeout(0xFFFFFFFFLL);
        return 0LL;
      }
      if ( v8 != 115 )
      {
LABEL_13:
        fprintf(
          *(FILE **)stderr_0,
          "Usage: %s [options]\n"
          "Options:\n"
          "\t-s <path>\tPath to ubus socket\n"
          "\t-h <path>\trun as hotplug daemon\n"
          "\t-d <level>\tEnable debug messages\n"
          "\t-S\t\tPrint messages to stdout\n"
          "\n",
          *a2);
        return 1LL;
      }
      qword_28288 = *(_QWORD *)optarg_0;
    }
    else
    {
      v6 = 4;
      if ( v8 != 83 )
        goto LABEL_13;
    }
  }
```

옵션 문자열 “d:s:h:S”에 따라 옵션을 파싱한다.

- -d <level>: 디버그 레벨을 설정한다.
- -h <path>: 옵션 인자를 읽고, uloop를 초기화한 후 `sub_6500()` 함수를 호출하여 해당 모드를 처리한다. listen 상태로 들어가고 함수를 종료한다.
- -s <path>: 옵션 인자로 받은 값을 `qword_28288` 에 저장한다.
- 옵션이 올바르지 않으면 사용법 메시지를 출력한다.

```c
  ulog_open(v6, 24LL, "procd");
  ulog_threshold(8LL);
  setsid();
  unloop_init();
  signal(13, (__sighandler_t)((char *)&dword_0 + 1));
```

`ulog_open()`을 호출해 로깅 시스템을 초기화하고, 로깅 임계값을 설정한다. `setsid()`를 호출해 새로운 세션을 만들고, `uloop_init()`으로 이벤트 루프 관련 초기화를 진행한다.

```c
  if ( getpid() == 1 )
  {
    v9 = (void (__fastcall *)(int, const struct sigaction *, struct sigaction *))sigaction_0;
    sigaction(15, (const struct sigaction *)off_27740, 0LL);
    v9(2, (const struct sigaction *)off_27740, 0LL);
    v9(10, (const struct sigaction *)off_27740, 0LL);
    v9(12, (const struct sigaction *)off_27740, 0LL);
    v9(30, (const struct sigaction *)off_27740, 0LL);
    v9(11, (const struct sigaction *)off_277D8, 0LL);
    v9(7, (const struct sigaction *)off_277D8, 0LL);
    v9(1, (const struct sigaction *)off_27870, 0LL);
    v9(9, (const struct sigaction *)off_27870, 0LL);
    v9(19, (const struct sigaction *)off_27870, 0LL);
    reboot(0);
  }
  if ( getpid() == 1 )
  {
    sub_117AC();
  }
```

PID가 1인 경우에는 아래 과정에 따라 init역할을 수행한다.

먼저 `sigaction()`을 여러 번 호출하여 적절한 핸들러를 설정한다.

- `SIGTERM(15)`, `SIGINT(2)`, `SIGUSR1(10)`, `SIGALRM(12)`, `SIGCONT(30)`에 대한 처리
    
    ```c
    __int64 __fastcall sub_116A8(unsigned int a1)
    {
      __int64 v1; // x0
      const char *v2; // x2
      unsigned int v3; // w20
      __int64 result; // x0
    
      if ( a1 <= 0x1E )
      {
        v1 = 1LL << a1;
        if ( (v1 & 0x40001400) != 0 )
        {
          v2 = "poweroff";
          v3 = 1126301404;
          goto LABEL_5;
        }
        if ( (v1 & 0x8004) != 0 )
        {
          v2 = "reboot";
          v3 = 19088743;
          goto LABEL_5;
        }
      }
      v2 = 0LL;
      v3 = 0;
    LABEL_5:
      result = (__int64)&off_27000;
      if ( dword_27D60 )
        result = ulog(5LL, "Triggering %s\n", v2);
      if ( v3 )
      {
        result = (unsigned int)dword_28324;
        if ( dword_28324 <= 4 )
          return sub_1164C(v3);
      }
      return result;
    }
    ```
    
    입력된 코드 a1을 이용해 종료 동작(재부팅 or 전원 끄기)을 선택하고, 조건에 따라 로그를 남긴 후 실제 종료 명령을 실행하는 하위 함수 `sub_1164C`를 호출한다.
    
    ```c
    void __fastcall sub_1164C(int a1)
    {
      const char *v1; // x0
      const char *v2; // x19
      char *v3; // x0
      __int64 v4; // x20
      int (*v5)(const char *, int, ...); // x21
      int v6; // w0
      char v8[12]; // [xsp-A0h] [xbp-A0h] BYREF
      __int64 v9; // [xsp-70h] [xbp-70h] BYREF
      _QWORD v10[6]; // [xsp-30h] [xbp-30h]
    
      if ( (unsigned int)dword_27D60 > 1 )
        ulog_0(5LL, "Shutting down system with event %x\n", a1);
      dword_28320 = a1;
      dword_28324 = 5;
      strcpy(v8, "/sbin/ubusd");
      v10[0] = Tty0;
      v10[1] = Console;
      v10[2] = qword_271F0;
      v1 = (const char *)sub_5F28(&v9, 20LL);
      v2 = v1;
      if ( v1 )
      {
        v3 = off_26F78(v1, 44);
        if ( v3 )
          *v3 = 0;
        v4 = 0LL;
      }
      else
      {
        v2 = "tty0";
        v4 = 1LL;
      }
      if ( chdir("/dev") )
      {
        ulog(3LL, "failed to change dir to /dev: %m\n");
      }
      else
      {
        v5 = (int (*)(const char *, int, ...))open_0;
        while ( 1 )
        {
          v6 = v5(v2, 0);
          if ( (v6 & 0x80000000) == 0 )
            break;
          v2 = (const char *)v10[v4++];
          if ( !v2 )
            goto LABEL_14;
        }
        close(v6);
    LABEL_14:
        if ( chdir("/") )
          ulog(3LL, "failed to change dir to /: %m\n");
        if ( v2 )
          sub_10B4C(v2);
      }
      ulog(6LL, "- shutdown -\n");
      sub_6A2C("shutdown");
      sync();
    }
    ```
    
    이 함수는 종료 이벤트를 처리할 때, 종료 이벤트 코드와 상태를 기록하고, 올바른 콘솔장치(tty0 or console)를 선택하여 필요한 리다이렉션 또는 설정을 수행한 후, shutdown처리 루틴을 호출하고 파일시스템을 동기화한다.
    
- `SIGSEGV(11)`, `SIGBUS(7)`에 대한 처리
    
    ```c
    void __noreturn sub_11C84()
    {
      void (*v0)(__int64, const char *, ...); // x19
    
      v0 = (void (*)(__int64, const char *, ...))ulog_0;
      ulog_0(3LL, "Rebooting as procd has crashed\n");
      v0(6LL, "reboot\n");
      fflush_0(*(FILE **)stderr_0);
      sync_0();
      sleep_0(2u);
      reboot_0(19088743);
      while ( 1 )
        ;
    }
    ```
    
    이 함수는 `procd`가 크래시한 경우 호출되며, 안전하게 시스템 상태를 동기화하고 재부팅을 시도한다.
    
- `SIGHUP(1)`, `SIGKILL(9)`, `SIGPIPE(19)`에 대한 처리
    
    ```c
    __int64 __fastcall sub_60E4(int a1)
    {
      return ulog_0(3LL, "Got unexpected signal %d\n", a1);
    }
    ```
    
    이 함수는 예상치 못한 시그널을 받은 경우 로그를 출력한다.
    

시그널에 대한 처리를 정의한 이후에는 `sub_117AC()`함수가 실행된다.

```c
void sub_117AC()
{
  //(변수 초기화 부분 생략)
  if ( (unsigned int)dword_27D60 > 3 )
    ulog_0(5LL, "Change state %d -> %d\n", dword_28324, dword_28324 + 1);
  ++dword_28324;
  strcpy(s, "/sbin/ubusd");
  switch ( dword_28324 )
  {
    case 1:
      v0 = ulog_0(6LL, "- early -\n");
      sub_7FC4(v0, v1, v2);
      sub_6500("/etc/hotplug.json");
      file = "udevtrigger";
      v59 = 0LL;
      v3 = umask_0(0);
      v4 = v3;
      if ( !(unsigned __int8)sub_64A4(v3, v5, v6) )
      {
        v7 = (void (__fastcall *)(const char *, int))umount2_0;
        umount2_0("/dev/pts", 2);
        v7("/dev/", 2);
        v8 = (void (__fastcall *)(const char *, const char *, const char *, unsigned __int64, const void *))mount_0;
        mount_0("tmpfs", "/dev", "tmpfs", 2uLL, "mode=0755,size=512K");
        mkdir_0("/dev/pts", 0x1EDu);
        v8("devpts", "/dev/pts", "devpts", 0xAuLL, 0LL);
      }
      symlink_0("/tmp/shm", "/dev/shm");
      umask_0(v4);
      qword_28340 = (__int64)sub_6258;
      v9 = fork_0();
      dword_28348 = v9;
      if ( !v9 )
      {
        execvp_0(file, &file);
        ulog_0(3LL, "Failed to start coldplug: %m\n");
        exit_0(1);
      }
      if ( v9 <= 0 )
      {
        ulog_0(3LL, "Failed to start new coldplug instance: %m\n");
      }
      else
      {
        uloop_process_add_0(algn_28328);
        if ( (unsigned int)dword_27D60 > 3 )
          ulog_0(5LL, "Launched coldplug instance, pid=%d\n", (unsigned int)dword_28348);
      }
      return;
    case 2:
      sub_7FC4(&dword_27D60, (unsigned int)(dword_28324 - 1), (unsigned int)dword_28324);
      sub_10B4C("console");
      v10 = (char *)getpwnam_0("ubus");
      if ( v10 )
      {
        v11 = (void (*)(__int64, const char *, ...))ulog_0;
        ulog_0(6LL, "- ubus -\n");
        mkdir_0(*((const char **)v10 + 4), 0x1EDu);
        if ( chown_0(*((const char **)v10 + 4), *((_DWORD *)v10 + 4), *((_DWORD *)v10 + 5)) )
          v11(6LL, "- ubus - failed to chown(%s)\n", *((const char **)v10 + 4));
      }
      else
      {
        ulog_0(6LL, "- ubus (running as root!) -\n");
      }
      qword_27DB8 = (__int64)&sub_11868;
      dword_27D98 = 50;
      sub_62BC(&dword_27D60, 50LL, v12);
      if ( v10 )
        v10 = "ubus";
      blob_buf_init_0(&qword_27DF0, 0LL);
      sub_67A4((int)&qword_27DF0, (int)"name", "ubus");
      v13 = sub_7DD4(&qword_27DF0, "instances");
      v14 = sub_7DD4(&qword_27DF0, "instance1");
      v15 = blobmsg_open_nested_0(&qword_27DF0, "command", 1LL);
      v16 = strtok_0;
      for ( i = s; ; i = 0LL )
      {
        v18 = v16(i, " ");
        if ( !v18 )
          break;
        sub_67A4((int)&qword_27DF0, 0, v18);
      }
      v19 = (void (__fastcall *)(__int64 *, __int64))blob_nest_end_0;
      blob_nest_end_0(&qword_27DF0, v15);
      v20 = blobmsg_open_nested_0(&qword_27DF0, "respawn", 1LL);
      sub_67A4((int)&qword_27DF0, 0, "3600");
      sub_67A4((int)&qword_27DF0, 0, "1");
      sub_67A4((int)&qword_27DF0, 0, "0");
      v19(&qword_27DF0, v20);
      if ( v10 )
      {
        sub_67A4((int)&qword_27DF0, (int)"user", v10);
        sub_67A4((int)&qword_27DF0, (int)"group", v10);
      }
      v21 = (void (__fastcall *)(__int64 *, __int64))blob_nest_end_0;
      blob_nest_end_0(&qword_27DF0, v14);
      v21(&qword_27DF0, v13);
      sub_F858(0LL, 0LL, 0LL, "add", qword_27DF0);
      return;
    case 3:
      v22 = (void (*)(__int64, const char *, ...))ulog_0;
      ulog_0(6LL, "- init -\n");
      v23 = off_26DC0("/etc/inittab", "r");
      if ( !v23 )
      {
        v22(3LL, "Failed to open %s: %m\n", "/etc/inittab");
        goto LABEL_29;
      }
      regcomp_0(&preg, "([a-zA-Z0-9]*):([a-zA-Z0-9]*):([a-zA-Z0-9]*):(.*)", 1);
      v24 = (char *)malloc_0(0x80uLL);
      v25 = calloc_0(1uLL, 0xC8uLL);
      v54 = (char *(__fastcall *)(char *, int, FILE *))fgets_0;
LABEL_31:
      while ( v54(v24, 128, v23) )
      {
        for ( j = (int)strlen_0(v24); ; --j )
        {
          v28 = (unsigned __int8)v24[j - 1];
          if ( v28 != 32 && (unsigned int)(v28 - 9) > 4 )
            break;
        }
        v24[j] = 0;
        if ( *v24 != 35 && !regexec_0(&preg, v24, 5uLL, (regmatch_t *)&file, 0) )
        {
          if ( (unsigned int)dword_27D60 > 3 )
            ulog_0(5LL, "Parsing inittab - %s\n", v24);
          p_file = &file;
          for ( k = 0LL; k != 4; ++k )
          {
            p_file[1][(_QWORD)v24] = 0;
            v32 = p_file[2];
            p_file += 2;
            v56[k] = &v32[(_QWORD)v24];
          }
          v33 = 0LL;
          v34 = strtok_0;
          for ( m = strtok_0((char *)v56[3], " "); ; m = v34(0LL, " ") )
          {
            v36 = v33;
            v37 = (int)v33 <= 6;
            if ( !m )
              v37 = 0;
            ++v33;
            if ( !v37 )
              break;
            v25[v33 + 2] = m;
          }
          v38 = (const char *)v56[2];
          v25[v36 + 3] = 0LL;
          v39 = (int (__fastcall *)(const char *, const char *))strcmp_0;
          v40 = 0LL;
          v25[2] = v56[0];
          v25[11] = v24;
          do
          {
            if ( !v39((&sysinit)[3 * v40], v38) )
            {
              v25[12] = &(&sysinit)[3 * (int)v40];
              v41 = off_27350[0];
              v25[1] = off_27350[0];
              v42 = malloc_0;
              off_27350[0] = v25;
              *v25 = &off_27348;
              *v41 = v25;
              v24 = (char *)v42(0x80uLL);
              v25 = calloc_0(1uLL, 0xC8uLL);
              goto LABEL_31;
            }
            ++v40;
          }
          while ( v40 != 7 );
          ulog_0(3LL, "Unknown init handler %s\n", v38);
        }
      }
      fclose_0(v23);
      v26 = free_0;
      free_0(v24);
      v26(v25);
      regfree_0(&preg);
LABEL_29:
      sub_6A2C("respawn");
      sub_6A2C("askconsole");
      sub_6A2C("askfirst");
      sub_6A2C("sysinit");
      ulog_open_0(2LL, 24LL, "procd");
      return;
    case 4:
      ulog_0(6LL, "- init complete -\n");
      sub_6A2C("respawnlate");
      sub_6A2C("askconsolelate");
      return;
    case 5:
      file = Tty0;
      v59 = Console;
      v60 = qword_271F0;
      v43 = (const char *)sub_5F28(&preg, 20LL);
      v44 = v43;
      if ( v43 )
      {
        v45 = strchr_0(v43, 44);
        if ( v45 )
          *v45 = 0;
        v46 = 0LL;
      }
      else
      {
        v44 = "tty0";
        v46 = 1LL;
      }
      if ( chdir_0("/dev") )
      {
        ulog_0(3LL, "failed to change dir to /dev: %m\n");
        goto LABEL_63;
      }
      v47 = (int (*)(const char *, int, ...))open_0;
      break;
    case 6:
      signal_0(17, (__sighandler_t)((char *)&dword_0 + 1));
      v49 = ulog_0;
      ulog_0(6LL, "- SIGTERM processes -\n");
      v50 = (void (__fastcall *)(__pid_t, int))kill_0;
      ((void (__fastcall *)(__pid_t, int))kill_0)(-1, 15);
      v51 = sync_0;
      sync_0();
      v52 = (unsigned int (__fastcall *)(unsigned int))sleep_0;
      ((void (__fastcall *)(unsigned int))sleep_0)(1u);
      v49(6LL, "- SIGKILL processes -\n");
      v50(-1, 9);
      v51();
      v53 = v52(1u);
      sub_10A78(v53);
    default:
      ulog_0(3LL, "Unhandled state %d\n", (unsigned int)dword_28324);
      return;
  }
  while ( 1 )
  {
    v48 = v47(v44, 0);
    if ( (v48 & 0x80000000) == 0 )
      break;
    v44 = (&file)[v46++];
    if ( !v44 )
      goto LABEL_67;
  }
  close_0(v48);
LABEL_67:
  if ( chdir_0("/") )
    ulog_0(3LL, "failed to change dir to /: %m\n");
  if ( v44 )
    sub_10B4C(v44);
LABEL_63:
  ulog_0(6LL, "- shutdown -\n");
  sub_6A2C("shutdown");
  sync_0();
}
```

이 함수는 시스템 초기화 과정으로, 여러 단계를 거쳐 시스템의 초기화 상태를 설정하고, 필요한 스로세스를 실행하며, 종료작업을 처리한다. 이 때, `dword_28324`라는 상태 값에 따라 아래와 같이 처리된다.

- case 1
    
    로그 출력을 통해 초기화 작업을 시작한다. `sub_7FC4`와 `sub_6500` 을 실행한다.
    
    ```c
    __int64 sub_7FC4()
    {
      __int64 result; // x0
      __int64 (*v1)(__int64, const char *, ...); // x19
      __int64 v2; // x0
    
      qword_27ED0 = (__int64)sub_72FC;
      result = sub_6100();
      if ( (result & 0x80000000) == 0 )
      {
        v1 = ulog_0;
        v2 = ulog_0(6LL, "- watchdog -\n");
        sub_7F94(v2);
        sub_72FC(&unk_27EB8);
        result = (unsigned int)dword_27D60;
        if ( (unsigned int)dword_27D60 > 3 )
          return v1(5LL, "Opened watchdog with timeout %ds\n", dword_27050);
      }
      return result;
    }
    ```
    
    `sub_7FC4`는 `watchdog` 장치 초기화를 확인한 후, 관련 로그를 남기고 `watchdog`를 구성하는 여러 하위 함수를 호출한다. 디버그 레벨이 높은 경우에는 `watchdog` 타임아웃 정보를 추가로 출력하여 상태를 알려준다.
    
    ```c
    __int64 __fastcall sub_6500(const char *a1)
    {
      int v1; // w0
      int optval; // [xsp+3Ch] [xbp+3Ch] BYREF
      sockaddr addr; // [xsp+40h] [xbp+40h] BYREF
    
      optval = 0x80000;
      *(_DWORD *)addr.sa_data = 0;
      *(_WORD *)&addr.sa_data[4] = 0;
      qword_27E20 = (__int64)strdup_0(a1);
      addr.sa_family = 16;
      *(_DWORD *)&addr.sa_data[6] = -1;
      v1 = socket_0(16, 524290, 15);
      LODWORD(qword_272C0) = v1;
      if ( v1 == -1 )
      {
        ulog_0(3LL, "Failed to open hotplug socket: %m\n");
        goto LABEL_3;
      }
      if ( bind_0(v1, &addr, 0xCu) )
      {
        ulog_0(3LL, "Failed to bind hotplug socket: %m\n");
    LABEL_3:
        exit_0(1);
      }
      if ( setsockopt_0(qword_272C0, 1, 33, &optval, 4u) )
        ulog_0(3LL, "Failed to resize receive buffer: %m\n");
      json_script_init_0(&unk_272C8);
      qword_27E40 = (__int64)sub_6F20;
      return uloop_fd_add_0(&off_272B8, 1LL);
    }
    ```
    
    `sub_6500`함수는 hotplug 이벤트 처리를 위한 네트워크 소켓을 설정하고, 이벤트 루프에 등록한 후 그 결과를 반환한다.
    
    두 함수 실행 이후 `udevtrigger`실행하여 장치 관리를 초기화하고, `chdir("/dev")`로 작업 디렉터리로 변경한 후 `/dev/pts`와 `tmpfs`을 마운트하고, `/dev/shm`의 심볼릭 링크도 생성한다. 다음으로, `fork()`로 새로운 프로세스에서 `coldplug`를 실행한다.
    
- case 2
    
    `sub_7FC4`와 `sub_10B4C`를 실행한다. `sub_7FC4`는 위의 `watchdog` 처리 함수이다.
    
    ```c
    __int64 __fastcall sub_10B4C(const char *a1)
    {
      int (__fastcall *v1)(const char *); // x20
      FILE *(__fastcall *v4)(const char *, const char *, FILE *); // x19
      int (*v5)(int, int, ...); // x19
      int v6; // w0
    
      v1 = chdir_0;
      if ( chdir_0("/dev") )
        return ulog_0(3LL, "failed to set stdio: %m\n");
      v4 = freopen_0;
      if ( !freopen_0(a1, "r", *(FILE **)stdin_0)
        || !v4(a1, "w", *(FILE **)stdout_0)
        || !v4(a1, "w", *(FILE **)stderr_0)
        || v1("/") )
      {
        return ulog_0(3LL, "failed to set stdio: %m\n");
      }
      v5 = fcntl_0;
      v6 = fcntl_0(2, 3);
      return v5(2, 4, v6 | 0x800u);
    }
    ```
    
    `sub_10B4C`는 지정된 파일 `a1`로 표준 입출력(`stdin`, `stdout`, `stderr`)을 리다이렉트하고, 표준 에러 스트림에 논블로킹 모드를 설정하여, 데몬이나 백그라운드 프로세스에서 I/O를 해당 파일로 보내도록 설정하는 역할을 한다. 이를 통해 console을 설정한다.
    
    console 설정 이후에는 `ubus` 관련 작업을 처리한다. `ubus` 사용자 정보를 가져와 `/dev`경로에 대해 권한 설정을 수행하고, 메시지 로그를 출력한다. 이후 `sub_62BC`와 `blobmsg_open_nested` 등을 사용하여 `ubus` 명령을 처리한다.
    
    ```c
    __int64 sub_62BC()
    {
      __int64 result; // x0
    
      off_26ED0(&unk_27DA0, (unsigned int)dword_27D98);
      LODWORD(result) = 2 * dword_27D98;
      if ( 2 * dword_27D98 >= 1001 )
        result = 1000LL;
      else
        result = (unsigned int)result;
      dword_27D98 = result;
      return result;
    }
    ```
    
    (`sub_62BC()`는 timeout을 설정한다.)
    
- case 3
    
    `/etc/inittab`파일을 읽고 파싱하여 부팅 시 초기화할 서비스를 설정한다. 이 과정에서 `regcomp`와 정규식을 사용하여 `inittab` 파일을 구문 분석하고, `blobmsg` 를 통해 각 서비스를 관리한다. 서비스 이름 및 명령을 추출하고, 일치하는 처리기로 초기화 작업을 수행한다. 최종적으로 초기화가 완료되면 `sync()`를 사용하여 파일 시스템을 동기화한다.
    
    `/etc/inittab` 
    
    ```
    ::sysinit:/etc/init.d/rcS S boot
    ::shutdown:/etc/init.d/rcS K shutdown
    ttyS0::respawnlate:/usr/bin/shell
    #ttyS0::respawnlate:/usr/libexec/login.sh
    ```
    
    해당 파일에서 `sysinit`을 찾아서 초기화한다.
    
    ```bash
    #!/bin/sh /etc/rc.common
    # Copyright (C) 2006-2011 OpenWrt.org
    
    START=10
    STOP=90
    
    uci_apply_defaults() {
    	. /lib/functions/system.sh
    
    	cd /etc/uci-defaults || return 0
    	files="$(ls)"
    	[ -z "$files" ] && return 0
    	mkdir -p /tmp/.uci
    	for file in $files; do
    		( . "./$(basename $file)" ) && rm -f "$file"
    	done
    	uci commit
    }
    
    . /lib/upgrade/nand.sh
    get_rootfs_ext_mountpoint_nand(){
    	local ubidev="$( nand_find_ubi "$CI_UBIPART" )"
    	local rootext_ubivol="$( nand_find_volume $ubidev $CI_ROOTEXTPART )"
    	local rootext_ubiblk="ubiblock${rootext_ubivol:3}"
    	[ -z "$rootext_ubivol" ] && return 1
    	ubiblock -c /dev/$rootext_ubivol
    	mkdir -p /mnt/$rootext_ubiblk
    	mount /dev/$rootext_ubiblk /mnt/$rootext_ubiblk && echo /mnt/$rootext_ubiblk && return 0
    	return 1
    }
    
    get_rootfs_ext_mountpoint_nor(){
    	local mtd=`cat /proc/mtd | grep "rootfs_ext" | cut -d ':' -f 1`
    	local mountpoint=`mount | grep "mtdblock${mtd#mtd}" | cut -d ' ' -f 3`
    	[ -z "$mtd" -o -z "$mountpoint" ] && return 1
    	echo $mountpoint
    	return 0
    }
    
    handle_rootfs_ext(){
    	local mountpoint=$(get_rootfs_ext_mountpoint_nand)
    	[ -z "$mountpoint" ] && mountpoint=$(get_rootfs_ext_mountpoint_nor)
    	[ -z "$mountpoint" ] && mountpoint="/etc/customer"
    	[ -d "$mountpoint" ] || return
    	find "$mountpoint" ! -type d | while read file; do
    		link=${file#$mountpoint}
    		if [ ! -L "$link" -o "`readlink -n "$link"`" != "$file" ]; then
    			mkdir -p "`dirname "$link"`"
    			ln -sf "$file" "$link"
    		fi
    	done
    
    	find /overlay/upper/ -type l | while read file; do
    		link=$(readlink -n "$file")
    		[[ "$link" != "/mnt/*" ]] && continue
    		[[ "$link" == "$mountpoint/*" ]] && continue
    		file=${file#/overlay/upper}
    		rm -f "$file"
    		[ -e "/rom$file" ] && cp -fr "/rom$file" "$file"
    	done
    }
    
    boot() {
    	[ -f /proc/mounts ] || /sbin/mount_root
    	handle_rootfs_ext
    	[ -f /proc/jffs2_bbc ] && echo "S" > /proc/jffs2_bbc
    
    	mkdir -p /var/lock
    	chmod 1777 /var/lock
    	mkdir -p /var/log
    	mkdir -p /var/run
    	mkdir -p /var/state
    	mkdir -p /var/tmp
    	mkdir -p /tmp/.uci
    	chmod 0700 /tmp/.uci
    	touch /var/log/wtmp
    	touch /var/log/lastlog
    	mkdir -p /tmp/resolv.conf.d
    	touch /tmp/resolv.conf.d/resolv.conf.auto
    	ln -sf /tmp/resolv.conf.d/resolv.conf.auto /tmp/resolv.conf
    	grep -q debugfs /proc/filesystems && /bin/mount -o noatime -t debugfs debugfs /sys/kernel/debug
    	grep -q bpf /proc/filesystems && /bin/mount -o nosuid,nodev,noexec,noatime,mode=0700 -t bpf bpffs /sys/fs/bpf
    	grep -q pstore /proc/filesystems && /bin/mount -o noatime -t pstore pstore /sys/fs/pstore
    	[ "$FAILSAFE" = "true" ] && touch /tmp/.failsafe
    
    	[ -f /etc/hotplug.d/firmware/12-mtk-wifi-testmode ] && sh /etc/hotplug.d/firmware/12-mtk-wifi-testmode
    
    	/sbin/kmodloader
    
    	[ ! -f /etc/config/wireless ] && {
    		# compat for bcm47xx and mvebu
    		sleep 1
    	}
    
    	/bin/config_generate
    	uci_apply_defaults
    	sync
    	
    	# temporary hack until configd exists
    	/sbin/reload_config
    }
    ```
    
    `/etc/init.d/boot`파일은 시스템 초기화 작업을 수행하는 쉘 스크립트이다. 이 스크립트는 다음과 같은 과정을 거쳐 실행된다.
    
    - 파일 상단에 START=10, STOP=90으로 실행순서를 지정하여 이 스크립트가 다른 스크립트 파일 `/etc/rc.common`과 함께 `procd`에 의해 관리되도록 한다.
    - `/etc/uci-defaults` 디렉터리 내에 있는 모든 스크립트를 순차적으로 실행한다.
        
        ![image.png](img/image2.png?raw=true)
        
        실행되는 스크립트들이다.
        
        ```bash
        #!/bin/sh
        
        commit=0
        
        if [ -z "$(uci -q get uhttpd.main.ubus_prefix)" ]; then
        	uci set uhttpd.main.ubus_prefix=/ubus
        	commit=1
        fi
        
        [ "$(uci -q get uhttpd.main.ubus_socket)" = "/var/run/ubus.sock" ] && {
        	uci set uhttpd.main.ubus_socket='/var/run/ubus/ubus.sock'
        	commit=1
        }
        
        [ "$commit" = 1 ] && uci commit uhttpd && /etc/init.d/uhttpd reload
        
        exit 0
        ```
        
        실행되는 스크립트 중 `00_uhttpd_ubus`스크립트이다. `uhttpd`의 `ubus_prefix`와 `ubus_socket` 설정이 올바른지 확인하고, 필요시 자동으로 수정하여 변경사항을 적용하여 실행되도록 설계되었다.
        
        각 스크립트는 실행 후 삭제되고, uci 설정을 커밋하여 기본 설정값을 시스템에 반영한다.
        
    - `get_rootfs_ext_mountpoint_nand`와 `get_rootfs_ext_mountpoint_nor` 함수는 NAND나 NOR 플래시에서 확장(rootfs_ext) 파티션의 마운트 포인트를 찾는다. 만약 찾지 못하면 기본값으로 `/etc/customer`를 사용한다.
    - 앞서 찾은 마운트 포인트 내의 파일들을 검색하여, 해당 파일들을 루트 파일시스템 내의 대응 경로에 심볼릭 링크로 연결한다. 또한, 오버레이 파일시스템(`/overlay/upper`) 내의 일부 심볼릭 링크들을 확인하여, 필요 시 복사 작업을 수행한다.
    - `/proc/mounts` 파일이 없으면 `/sbin/mount_root`를 호출해 루트 파일시스템을 마운트한다. 확장 루트 파일시스템 영역(handle_rootfs_ext) 처리를 수행한다. 특정 시스템 파일(예: `/proc/jffs2_bbc`)이 존재하면 초기화 명령(여기서는 "S")을 전달한다.
    - 처리가 끝나면 `/var/lock`, `/var/log`, `/var/run`, `/var/state`, `/var/tmp`, `/tmp/.uci` 등 필수 디렉터리를 생성하고 적절한 권한을 설정한다. 또한 로그 파일(`wtmp`, `lastlog`)과 `resolv.conf` 관련 임시 파일들을 생성하고 심볼릭 링크를 설정한다.
    - `debugfs`, `bpf`, `pstore`와 같은 가상파일시스템을 필요에 따라 마운트한다.
    - FAILSAFE 모드가 활성화되어 있으면 failsafe 마커 파일을 생성한다.
    - 특정 핫플러그 스크립트(예: `12-mtk-wifi-testmode`)가 있으면 실행한다.
    - `/sbin/kmodloader`를 실행하여 커널 모듈들을 로드한다.
    - 무선 설정 파일이 없을 경우 약간의 지연(sleep 1)을 주어 호환성을 확보한다.
    - `/bin/config_generate`를 호출해 시스템 설정 파일을 생성한다.
    - 앞서 정의한 uci_apply_defaults 함수를 실행하여 기본 UCI 설정을 적용한다.
    - 마지막으로 sync를 호출하여 디스크 동기화를 수행하고, 임시적으로 `/sbin/reload_config`를 실행하여 설정을 다시 로드한다.
- case 4
    
    초기화 완료 메시지를 로그에 출력하고, `respawnlate`나 `askfirst`를 포함한 서비스(`/usr/bin/shell`)를 호출하여 시스템 초기화를 마무리한다. 이후 `/sbin/procd`는 요청을 기다리고 처리하며, 다른 서비스들을 관리하고, 시스템 상태를 기록하고 상위 프로세스로 반환하는 역할을 하게된다. 
    
- case 5
    
    `tty0`와 같은 콘솔 장치를 설정하고 관련 작업을 계속하기 위해 파일을 `chdir("/dev")`와 함께 변경한다. 변경 후 `chown()`으로 적절한 권한을 설정하고, `/dev/pts`를 마운트한다.
    
- case 6
    
    `SIGTERM`시그널을 모든 프로세스에 보내고 `kill(SIGKILL)`을 통해 강제 종료 작업을 수행한다. 시스템 종료 전에 필요한 동기화 작업을 실행하고, 프로세스 종료 후 `sleep(1)`과 로그 출력을 진행한다.
    
- PID가 1이 아닌 경우(자식 프로세스)
    
    ```c
      else
      {
        qword_27DB8 = (__int64)&sub_11868;
        dword_27D98 = 50;
        sub_62BC();
      }
      uloop_run_timeout(0xFFFFFFFFLL);
      uloop_done();
      return 0LL;
    }
    ```
    
    전역 변수에 `sub_11868`에 대한 포인터와 타임아웃 값(50)을 설정한다.
    
    ```c
    void sub_11868()
    {
      //(변수 초기화 부분 생략)
      v0 = ubus_connect_0(qword_28288);
      qword_28290 = v0;
      if ( v0 )
      {
        *(_QWORD *)(v0 + 160) = sub_6308;
        qword_28280 = v0;
        sub_10788(0LL, 0LL);
        v1 = opendir_0("/etc/hotplug.d");
        if ( v1 )
        {
          v2 = (struct dirent *(__fastcall *)(DIR *))readdir_0;
          v3 = strlen_0;
          while ( 1 )
          {
            v4 = v2(v1);
            if ( !v4 )
              break;
            if ( v4->d_type == 4 && v4->d_name[0] != 46 )
            {
              d_name = v4->d_name;
              v12 = v3(v4->d_name);
              sub_EE1C(v12, d_name);
            }
          }
          closedir_0(v1);
          v5 = inotify_init1_0(526336);
          qword_28350 = (__int64)sub_10898;
          dword_28358 = v5;
          qword_28318 = (__int64)calloc_0(1uLL, 0x1011uLL);
          inotify_add_watch_0(v5, "/etc/hotplug.d", 0x10003C0u);
          uloop_fd_add_0(&qword_28350, 1LL);
        }
        else
        {
          printf_0("failed to initialize hotplug subsystems from %s\n", "/etc/hotplug.d");
        }
        v6 = (void (__fastcall *)(__int64, void *))ubus_add_object_0;
        qword_27F70 = qword_28290;
        ((void (*)(void))ubus_add_object_0)();
        if ( !off_26EC8("/sbin/ujail", (struct stat *)&v13) )
          v6(qword_27F70, &unk_27650);
        v7 = qword_28290;
        v8 = getenv_0("INITRAMFS");
        dword_2827C = v8 != 0LL;
        if ( v8 )
          unsetenv_0("INITRAMFS");
        if ( (unsigned int)ubus_add_object_0(v7, &unk_276C8) )
        {
          v9 = (const char *)ubus_strerror_0();
          ulog_0(3LL, "Failed to add object: %s\n", v9);
        }
        qword_283D8 = (__int64)sub_8550;
        v10 = qword_28290;
        qword_27F60 = (__int64)sub_8798;
        if ( (unsigned int)((__int64 (__fastcall *)(__int64))ubus_register_subscriber_0)(qword_28290) )
          ulog_0(3LL, "failed to register ubus subscriber\n");
        if ( (unsigned int)ubus_register_event_handler_0(v10, &unk_28360, "ubus.object.add") )
          ulog_0(3LL, "failed to add ubus event handler\n");
        if ( (unsigned int)dword_27D60 > 1 )
          ulog_0(5LL, "Connected to ubus, id=%08x\n", *(_DWORD *)(qword_28290 + 144));
        dword_27D98 = 50;
        uloop_fd_add_0(qword_28290 + 80, 9LL);
        if ( dword_28324 == 2 )
          sub_117AC();
      }
      else
      {
        if ( (unsigned int)dword_27D60 > 3 )
          ulog_0(5LL, "Connection to ubus failed\n");
        sub_62BC();
      }
    }
    ```
    
    `sub_11868`함수는 `ubus`와 `hotplug` 서브시스템을 초기화하여 `procd`가 시스템 내 다른 컴포넌트와 이벤트를 주고받을 수 있도록 설정하는 역할을 하며, 성공 시 uloop 이벤트 루프에 관련 파일 디스크립터를 등록하고, 실패 시 대체 경로로 처리하는 역할을 한다.
    
    다음으로, `sub_62BC`를 통해 타임아웃 값을 설정한다.
    
    최종적으로 `uloop_run_timeout()`을 호출하여 무한 타임아웃 상태로 이벤트 루프를 실행하며, 데몬이 지속적으로 이벤트를 처리하도록 합니다. 이벤트 루프가 종료되면 `uloop_done()`을 호출하고, 함수는 0을 반환하며 종료한다.
    

결론적으로, 부팅 과정은 다음과 같다.

`/sbin/init` → `/etc/preinit` → `/sbin/procd` → `/etc/inittab`에서 조회→ `/etc/init.d/boot` → `/etc/uci-defaults/00_uhttpd_ubus`실행
