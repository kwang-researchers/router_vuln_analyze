# 4. 분석 환경

## 4.1. MEX605 Firmware Emulation & Debugging

---

관리자 페이지에서의 특정 요청이 어느 엔드포인트와 바이너리로 가는지 파악이 쉬웠던 tew-831dr 공유기와 달리, MEX605 공유기는 요청을 전부 ubus로 처리하여서 특정 서비스에 대한 요청이 어느 파일에서 처리되는지 파악하기 쉽지 않았기에 해당 공유기에 대해서만 에뮬레이션을 진행하여 분석환경을 마련했다.

### 4.1.1. rootfs 파일 시스템 생성

---

에뮬레이션에 필요한 파일 시스템은 `squashfs`지만, QEMU환경에서는 그대로 마운트할 수 없으므로, 펌웨어에서 원본 파일 시스템을 `binwalk -e`로 추출한 `squashfs-root`를 ext4 형식으로 변환해야 한다. ext4로 변환하는 과정은 아래 명령어를 사용했다.

```bash
dd if=/dev/zero of=rootfs.img bs=1M count=1024
mkfs.ext4 rootfs.img
sudo mount -o loop rootfs.img /mnt
sudo cp -a squashfs-root/* /mnt
```

그리고 동적 디버깅을 위해 아래와 같이 호스트에서 gdbserver를 다운로드 후 위에서 루프백 장치를 이용해 마운트 시킨 `rootfs.img`로 복사해주었다. 이때, 에뮬레이션된 바이너리들은 ARM64이기 때문에 `gdbserver-8.3.1-aarch64-le` 바이너리를 다운받아서 `rootfs/bin`에 넣었다.

```bash
wget https://github.com/hugsy/gdb-static/raw/master/gdbserver-8.3.1-aarch64-le
chmod +x gdbserver-8.3.1-aarch64-le

sudo cp gdbserver-8.3.1-aarch64-le /mnt/rootfs/bin/gdbserver
sudo chmod +x /mnt/rootfs/bin/gdbserver
```

### 4.1.2. 커널 빌드

---

펌웨어 내의 Linux kernel ARM64 boot executable Image를 통해 커널 버전을 확인했다.

```bash
strings E8 | grep "Linux version"                                                                             
Linux version 5.4.225 (yun@ubuntu) (gcc version 8.4.0 (OpenWrt GCC 8.4.0 unknown)) #0 SMP Tue Jun 20 02:21:09 2023
```

커널 5.4.225 버전을 다운로드한 후, 해당 디렉터리로 이동하여 빌드할 준비를 한다.

```bash
git clone --depth=1 --branch v5.4.225 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
```

ARM64용 크로스 컴파일러를 설치해준 후, 커널 빌드를 설정하고 빌드한다.

```bash
make ARCH=ARM64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make ARCH=ARM64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

### 4.1.3. QEMU 세팅

---

1. 커널을 빌드한 뒤 생성된 `arch/ARM64/boot/Image`를 -kernel 옵션으로 경로를 지정한다. 압축되지 않은 바이너리 커널이라서 바로 사용이 가능하다.
2. ext4로 변환한 `rootfs.img`를 -drive file로 지정한다. -initrd 옵션을 사용하지 않은 이유는, 파일을 RAM에 임시로 올리는 방식이 필요하지 않기 때문이다.
3. append 옵션으로 console과 root 디렉터리 경로를 지정한다. 이때, RAM이 아니라 vda를 사용하는 이유는 앞서 설명한 것과 같다. 즉, RAM에 임시적으로 파일을 올려서 사용하지 않기 때문이다.
4. 관리자 페이지 접속과 동적 디버깅을 위해 netdev user모드로 설정을 한 후, 포트포워딩을 설정했다.

최종 QEMU 실행 명령어는 다음과 같다.

```bash
qemu-system-aarch64 \
    -machine virt \
    -cpu cortex-a72 \
    -m 1024 \
    -nographic \
    -kernel linux/arch/ARM64/boot/Image \
    -append "console=ttyAMA0 root=/dev/vda rw" \
    -drive file=./_root.extracted/rootfs.img,format=raw,if=virtio \
    -netdev user,id=net0,hostfwd=tcp::2222-:22,hostfwd=tcp::8080-:80,hostfwd=tcp::1234-:1234 \
    -device virtio-net-pci,netdev=net0
```

### 4.1.4. 쉘 내 환경 구축

---

위의 명령어로 QEMU를 실행한 후, 다음 화면이 나올 때 아래와 같이 f와 엔터를 눌러 failsafe. 즉, 복구모드에 진입한다.

![image.png](img/image1.png?raw=true)

![image.png](img/image2.png?raw=true)

복구모드에 진입한 모습. 복구모드는 init부팅 도중 진입하여 수동으로 설정하는 방식이다. 최소한의 환경에서 관리자 페이지에 접속해 동적 디버깅을 수행하는 것이 목표이므로 복구모드에서 웹 서버를 실행하고 관리자 페이지에 접속하도록 한다.

**4.1.4.1. 인터넷 연결 설정**

1. dhcp할당을 위해 ip실행파일 권한 문제를 해결한다.
    
    ```bash
    rm -f /sbin/ip
    ln -s /bin/busybox /sbin/ip
    chmod +x /sbin/ip
    ```
    
    위와 같이 기존 ip실행 파일을 지운 뒤, busybox로 심볼릭링크가 걸린 ip실행파일을 새로 만들고 권한을 부여한다.
    

1. 복구모드에 진입했을 때 ip가 할당이 안 되어있었기 때문에 dhcp를 통해서 ip를 자동할당 받았다.
    
    ```bash
    /bin/busybox udhcpc -i eth0
    ```
    
    ![image.png](img/image3.png?raw=true)
    
    eth0이 up된 상태이고, ip는 10.0.2.15로 설정된 모습이다. 기본 게이트웨이도 올바르게 설정되었다.
    
2. 마지막으로 웹 서버 스크립트 실행을 위한 설정을 해준다. procd에 문제가 있어 다시 실행하고, MEX605의 웹 서버는 ubus를 통해 요청을 주고받기 때문에 ubusd역시 실행을 시켜준다. 대략적인 통신 과정은 4.1.5.1에서 서술하였다.
    
    ```bash
    killall procd
    /sbin/procd &
    /sbin/ubusd &
    ```
    

### 4.1.5. 웹 서버 실행

---

MEX605공유기는 openWRT 운영체제와 uhttpd경량 웹 서버를 사용한다.

**4.1.5.1. MEX605공유기 구조**

- 운영체제
    
    MEX605는 운영체제로 openWRT를 사용한다. openWRT란 임베디드 장치, 특히 무선 공유기(router)용으로 설계된 오픈 소스 리눅스 기반 운영체제로, 완전한 운영체제를 포함하고 있기 때문에 부트로더, 커널, 파일 시스템은 물론 전체 운영체제 기능을 가진다는 특징이 있다.
    

- 웹 서버
    
    openWRT는 내부적으로 경량 웹 서버인 uhttpd를 사용한다. HTTP, CGI, Lua등을 지원하며 ubus를 통해 통신을 한다. 웹 서버와 통신을 하기 위해서는 ubus라는 중앙 허브 역할을 하는 micro bus를 거쳐야 한다. 아래는 대략적인 웹 서버의 통신 과정이다.
    
    ![1.png](img/image4.png?raw=true)
    
    펌웨어 내에 uhttpd관련 파일은 3개가 있으며, 다음과 같다.
    
    - `./usr/sbin/uhttpd`는 웹 서버 실행 파일(바이너리)로, `/etc/init.d/uhttpd` 스크립트가 이 파일을 이용해서 웹 서버를 시작/중지한다.
    - `./etc/config/uhttpd`는 uhttpd의 환경 설정을 저장하는 파일로, 어떤 포트에서 listen을 하고 있는 지 등 설정들이 담겨있다.
    - `./etc/init.d/uhttpd`는 openWRT의 procd 시스템을 통해 uhttpd를 관리하는 스크립트로, config파일을 로드하면서 웹 서버 실행 파일을 실행하는 등 웹 서버를 시작/중지한다.

**4.1.5.2. 웹 서버 시작 스크립트 실행**

위에서 해당 스크립트(`/etc/init.d/uhttpd`)실행 준비를 모두 했기 때문에 웹 서버를 실행해 준 후, 잘 실행되었는지 확인해준다.

```bash
/etc/init.d/uhttpd start
```

![image.png](img/image26.png?raw=true)

해당 스크립트가 웹 서버 실행파일을 실행한 것을 확인했고, 22, 443, 23355, 80포트에서 listen하고 있다.

### 4.1.6. 관리자 페이지 접속

---

eth0의 ip를 확인해보면

```bash
/ # /bin/busybox ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global eth0
       valid_lft forever preferred_lft forever
```

다음과 같지만, QEMU를 처음 부팅 할 때 hostfwd=tcp::8080-:80과 같이 포트 포워딩을 설정하였기 때문에, 로컬에서 접속 시 

```bash
127.0.0.1:8080
```

으로 접속이 가능하다.

![image.png](img/image6.png?raw=true)

![image.png](img/image7.png?raw=true)

![image.png](img/image8.png?raw=true)

![image.png](img/image9.png?raw=true)

### 4.1.7. 부분 에뮬레이션

---

부분 에뮬레이션은 커널을 사용하지 않고 파일 시스템만 마운트한 후, chroot를 이용해 격리된 공간을 생성하고, 그 안에서 관리자 페이지에 접근할 수 있도록 설정하여 웹 페이지에 접속하는 방식이다. 파일 시스템은 이전에 ext4로 변환해 놓은 `rootfs.img`를 사용한다.

1. 우선 임시적으로 rootfs.img를 마운트 시킬 디렉터리를 /mnt/rootfs라는 이름으로 만든다.
    
    ```bash
    mkdir /mnt/rootfs
    ```
    
2. 만든 디렉터리에 rootfs.img를 마운트 시킨다.
    
    ```bash
    sudo mount -o loop rootfs.img /mnt/rootfs
    ```
    
3. x86 환경에서 ARM 바이너리를 실행하려면, qemu-aarch64-static을 `/mnt/rootfs`로 옮겨야 한다.
    
    ```bash
    sudo cp /usr/bin/qemu-aarch64-static /mnt/rootfs/usr/bin/
    ```
    
4. proc, sys, dev등 필요한 파일을 마운트한다.
    
    ```bash
    sudo mount -t proc /proc /mnt/rootfs/proc
    sudo mount --rbind /sys /mnt/rootfs/sys
    sudo mount --rbind /dev /mnt/rootfs/dev
    sudo mount --rbind /run /mnt/rootfs/run
    ```
    
    ARM 바이너리들이 proc, sys폴더의 정보에 의존하기 때문에 chroot 환경에서도 접근 가능해야 하므로 같이 마운트 시켜준다. 추가로 기존 시스템의 dev, sys내용을 rootfs내부에서 그대로 사용해주기 위해서 rbind옵션을 주었다.
    
5. chroot를 이용하여 rootfs내부로 진입한다.
    
    ```bash
    sudo chroot /mnt/rootfs /bin/sh
    ```
    
    ![image.png](img/image10.png?raw=true)
    

**4.1.7.1. chroot 내 관리자 페이지 접속**

chroot 환경에서도 관리자 페이지에 접속하려면 동일하게 uhttpd웹 서버를 실행해야 한다. 따라서 4.1.5.2와 같은 방법으로 uhttpd실행 파일을 직접 실행 시킨 후, 관리자 페이지 접속을 시도한다.

접속 시 브라우저에서 ipconfig로 확인한 현재 부분 에뮬레이션 환경의 ip인 172.31.120.64로 접속한다.

![image.png](img/image11.png?raw=true)

![image.png](img/image12.png?raw=true)

관리자 페이지에 접속이 가능하다.

### 4.1.8. 동적 디버깅

---

동적 디버깅 과정은 [starlink analyze 프로젝트](https://github.com/Starlink-Research/Starlink-Research/tree/master/V4%20DISHY#3-%EC%97%90%E)를 참고하여 제작했다.

동적 디버깅을 하는 방법은 에뮬레이터 시작 시 부팅된 바이너리에 attach하여 디버깅을 하는 방법과 바이너리를 처음부터 디버깅 하는 방법이 있는데, 5.1.6. 관리자 페이지 접속 부분 이후부터 진행했기 때문에 부팅된 바이너리에 attach하여 디버깅하는 방식으로 진행했다. 

```bash
qemu-system-aarch64 \
    -machine virt \
    -cpu cortex-a72 \
    -m 1024 \
    -nographic \
    -kernel linux/arch/ARM64/boot/Image \
    -append "console=ttyAMA0 root=/dev/vda rw" \
    -drive file=./_root.extracted/rootfs.img,format=raw,if=virtio \
    -netdev user,id=net0,hostfwd=tcp::2222-:22,hostfwd=tcp::8080-:80,hostfwd=tcp::1234-:1234 \
    -device virtio-net-pci,netdev=net0
```
QEMU 실행 시 user 모드로 설정했기 때문에, gdbserver를 사용 시 포트포워딩을 추가로 1234→1234로 설정해주어야 한다.

![image.png](img/image27.png?raw=true)

우선 ps명령어로 확인한 결과, 동적 디버깅을 진행할 uhttpd 바이너리는 242 PID를 가지고 있기 때문에, gdbserver를 열고 localhost:1234에서 원격 디버깅을 위해 대기하도록 설정한 후, PID가 242인 uhttpd에 attach 하여 디버깅을 준비하였다.

```bash
root@(none):/# /bin/gdbserver localhost:1234 --attach 242
Attached; pid = 242
Listening on port 1234
```

```bash
host pc
gdb-multiarch
pwndbg> target remote 127.0.0.1:1234
Remote debugging using 127.0.0.1:1234
Reading /usr/sbin/uhttpd from remote target...
```

호스트에서는 포트포워딩 된 포트 1234를 통하여 원격 디버깅을 수행하도록 접속한다.

![image.png](img/image13.png?raw=true)

위와 같이 디버깅을 위한 준비가 완료되었다. 하지만 심볼이 없고 stripped되어 있다. 심볼 복구 작업과 부분 에뮬레이션 환경에서 동적 디버깅을 통한 분석은 이후 후속 연구과제로 남아있다.

## 4.2. 포트 포워딩

---

### 포트 포워딩의 목적

공유기의 관리자 페이지를 분석하기 위해서는 우선 공유기와 동일한 네트워크에 연결되어 있어야 한다. 하지만 분석하는 날마다 매번 공유기를 갖고 다녀야 하고, 실제로 만나서만 공유기 관리자 페이지 분석이 가능하다는 불편함이 존재한다. 따라서 포트 포워딩을 통해 인터넷상에서 관리자 페이지에 접속할 수 있도록 한다.

### 4.2.1. tew-831dr 관리자 페이지 포트 포워딩

### 포트 포워딩 설계

![image.png](img/image15.png?raw=true)

포트 포워딩을 했을 경우 관리자 페이지를 접속하는 패킷의 흐름은 위와 같다. 먼저 인터넷과 연결된 AP에 패킷이 전달된다. 이때 이 AP에 tew-831dr의 포트 포워딩 설정이 되어 있다면, 해당 패킷을 tew-831dr에 전달할 수 있다. 예를 들어, AP에 8888번 포트로 패킷이 수신되었다면, 이 패킷을 tew-831dr의 80번 포트로 전송하는 것이다.

### tew-831dr 동작 모드 설정

![image.png](img/image16.png?raw=true)

tew-831dr은 기본적으로 `Router` 모드로 동작한다. 하지만 상위 AP와의 연결을 위해 `Access Point (AP)` 모드로 변경한 후, `Save & Apply`를 클릭한다.

![image.png](img/image17.png?raw=true)

tew-831dr 재시작 이후 와이파이에 접속하면, 할당된 IP가 tew-831dr이 아닌 상위 계층 AP의 네트워크 영역임을 확인할 수 있다.

### 포트포워딩 설정

![image.png](img/image18.png?raw=true)

포트포워딩을 하기 전에, 먼저 tew-831dr에 할당된 IP를 확인한다. 확인 방법은 tew-831dr이 연결된 AP의 관리자 페이지에서 확인하는 것이다.

![image.png](img/image19.png?raw=true)

그다음 AP에 접속한 단말에서 tew-831dr의 관리자 페이지에 접속할 수 있는지 확인한다. 이를 사전에 확인해 두지 않으면, 포트 포워딩을 제대로 설정했음에도 정상 접근이 되지 않는 상황이 발생하기 때문이다.

![image.png](img/image20.png?raw=true)

포트 포워딩을 하는 메뉴는 공유기 제조사마다 천차만별이지만, 방법은 대부분 동일하다. 위 이미지는 ipTIME의 포트 포워딩 설정 페이지다. 포트 포워딩은 외부에서 `Public IP:9000`으로 접속 시 `tew-831dr IP:80`으로 접속할 수 있도록 설정한다.

- 내부 IP 주소: tew-831dr에 할당된 IP
- 외부 포트: 9000~9000
- 내부 포트: 80~80

위와 같이 포트 포워딩을 설정한 후, 공인 IP와 9000번 포트를 통해 접속을 시도한다.

![image.png](img/image21.png?raw=true)

위와 같이 정상 접속되어 ID와 패스워드를 요구하는 것을 확인할 수 있다.

### 4.2.2. MEX605 관리자 페이지 포트 포워딩

![ll.jpg](img/image22.jpg?raw=true)

사용 중인 공유기(AP)의 LAN 3번 포트와 연결한다.

![image.png](img/image23.png?raw=true)

연결 후 MEX605 공유기 관리자 페이지로 접속하여 고급 > 원격 접속 > ACL 정책에서 그림과 같이 설정하여 원격 접속을 모두 허용한다.

![image.png](img/image24.png?raw=true)

그 다음 사용중인 공유기의 관리자 페이지에서 그림과 같이 포트포워딩을 설정한다.

내부에서 실행할 포트는 8080번으로 설정하였고, 외부에서 접속하는 포트는 80번으로 설정한다. 

사용 IP 목록에서 공유기 LAN3 포트와 연결된 MAX605의 ip 주소를 찾을 수 있다(192.168.1.23)

설정이 완료된 후 [WhatIsMyIP](https://www.whatismyip.com/)에서 로컬 ip주소를 확인했고,

확인한 `<공인 ip주소>:<설정한 외부포트>`로 접근하면 외부에서도 성공적으로 관리자 페이지에 접속할 수 있다.

![image.png](img/image25.png?raw=true)
