# 1. 분해 과정

---

# 1.1. tew-831dr

---

## 1.1.1. 공유기 정보

### 하드웨어 정보

- **CPU**: RTL8812BRH
- **Flash**: winbond 25Q64JVSIQ SPI Nor Flash (64M-bit)

### 소프트웨어 정보

- **UART Baud Rate**: 38400
- **Kernel**: Linux Kernel 3.10.9
- **Busybox**: v1.13.4
- **Firmware Version**: v1.0(601.130.1.1410)
- **Web Server**: Boa v0.94

## 1.1.2. 공유기 커버 분리

![공유기뒷면.png](img/image1.png?raw=true)

![image.png](img/image2.png?raw=true)

하단의 4개의 나사를 제거한 후, 공유기 커버를 분리한다. 분리 후 PCB 기판을 확인할 수 있는데, 해당 기판에서 UART의 4개의 핀(VCC, GND, TX, RX)을 식별할 수 있다.

## 1.1.3. USB to TTL 드라이버 & Tera Term 설치

https://www.silabs.com/developer-tools/usb-to-uart-bridge-vcp-drivers?tab=downloads

Tera Term을 설치하기 전에, USB to TTL 드라이버를 설치해야 한다. 자신이 소유한 USB to TTL 모델의 드라이버를 검색한 후 설치한다. 사용한 USB to TTL은 `AD-USBISP v0.36`이다.

- https://teratermproject.github.io/index-en.html
    
    Tera Term 설치 URL
    

Tera Term은 SSH/Telnet 접속뿐만 아니라, 시리얼 통신할 때도 사용할 수 있는 터미널 유틸리티이다. 그리고 터미널에 출력되는 데이터들을 자동으로 로깅할 수 있는 편리한 기능을 제공한다.

## 1.1.4. USB to TTL 연결 및 Tera Term 실행

![image.png](img/image3.png?raw=true)

USB to TTL과 UART 핀을 연결한다. 이때 GND끼리 동일하게 연결하되, TX와 RX를 교차해서 연결한다. 이때 공유기의 전원은 꺼져있어야 한다.

![image.png](img/image4.png?raw=true)

연결 후 Tera Term을 실행한다. Serial을 선택하며, USB to TTL로 인식된 Port를 선택한 후 OK를 클릭한다.

![image.png](img/image5.png?raw=true)

Tera Term의 상단 옵션 중 Setup→Serial Port를 클릭하면 Baud Rate를 설정할 수 있는 설정 다이얼로그가 표시된다. 이때 Speed를 `38400`으로 설정한 후 New Open을 클릭한다.

![image.png](img/image6.png?raw=true)

![image.png](img/image7.png?raw=true)

자동 로깅을 위해 File→Log 옵션을 클릭한다. 이 옵션에서는 로그 파일의 이름과 경로를 설정할 수 있다. 이름과 경로를 적절히 설정한 후 OK를 클릭하면 로깅이 시작된다.

![image.png](img/image8.png?raw=true)

![image.png](img/image9.png?raw=true)

공유기에 전원을 인가하면 로그가 출력되며 부팅이 진행된다. 부팅이 완료되었다면 File→Stop Logging을 클릭해 로깅을 종료한다.

![image.png](img/image10.png?raw=true)


로깅을 종료하면 지정된 경로에 로그 파일이 생성되고, 부트 로그가 파일에 저장된다.

## 1.1.5. Busybox 설치

![image.png](img/image11.png?raw=true)

UART로 접속한 쉘에서는 일부 명령이 제한되어 있는데, `id`, `dd`, `spi` 등이 그러하다. 이 이유는 해당 쉘에서 사용하는 Busybox에서 일부 명령만을 제공하기 때문이다. 따라서 새로운 Busybox를 설치해 이 제약을 해결하고자 한다.

![image.png](img/image12.png?raw=true)

쉘이 사용하는 Busybox에서는 `wget` 명령을 지원한다. 하지만 read-only 파일시스템인 점, 인터넷과 통신을 못 하는 상태인 점을 고려해야 한다. 따라서 아래의 과정을 거쳐 공유기 내부에 새로운 Busybox를 저장한다.

1. PC에서 아래 링크를 통해 Busybox 다운로드

```bash
https://www.Busybox.net/downloads/binaries/1.21.1/Busybox-mipsel # mips는 big endian, mipsel은 little endian
```

2. PC에서 공유기 와이파이에 접속 및 할당된 IP 확인 (`ipconfig`)

```powershell
ipconfig
	... 192.168.10.101
```

3. PC에서 파일 전송 서버 실행

```bash
# Busybox-mipsel 파일이 있는 디렉터리에서 실행
python -m http.server 8000
```

1. 공유기의 쉘에서 `/tmp` 로 이동
2. PC에서 할당된 IP 확인 후, 공유기의 쉘에서 `wget`로 서버 접속 및 Busybox 다운로드

```bash
wget http://192.168.10.101:8000/Busybox-mipsel
```

## 1.1.6. Busybox 실행 권한 부여

다운로드한 Busybox는 기본적으로 실행 권한이 부여되어 있지 않다. 이러면 일반적으로 `chmod` 명령으로 실행 권한을 직접 부여하여 해결하지만, 현재 사용하고 있는 Busybox는 `chmod` 명령을 지원하지 않는다. 따라서 아래의 과정을 통해 이 문제를 해결한다.

1. `/bin/ls`를 `/tmp`에 복사한다. (`ls`를 포함한 실행 권한이 있는 파일 모두 가능)
    
    ```bash
    # /tmp 디렉터리에서 실행
    cp /bin/ls .
    ```
    
2. `Busybox-mipsel`을 `ls`로 복사한다.
    
    ```bash
    cp ./Busybox-mipsel ./ls
    ls -l ./ls # 이름은 ls지만 내용은 Busybox-mipsel. 실행 권한 존재
    ```
    
3. 기존 `Busybox-mipsel` 파일을 지운 후, `ls` 의 이름을 `Busybox-mipsel` 로 변경한다.
    
    ```bash
    rm -f ./Busybox-mipsel
    cp ./ls ./Busybox-mipsel
    rm -f ./ls
    ```
    

1. `Busybox-mipsel`을 실행한다.
    
    ```bash
    ./Busybox-mipsel
    # 기존 Busybox에서 지원하지 않는 id 명령 실행
    ln -s ./id ./Busybox-mipsel
    ./id
    ```
    

해당 Busybox에는 `md`나 `spi`가 없지만, `dd`명령을 지원한다. 따라서 펌웨어를 저장한 플래시 메모리의 장치 파일을 찾는다면, `dd` 명령을 통해 추출할 수 있다.

## 1.1.7. 파일 시스템 추출

```bash
m25p80 spi0.0: found w25q64, expected m25p80
flash vendor: Winbond
m25p80 spi0.0: w25q64 (8192 Kbytes) (29000000 Hz)
5 rtkxxpart partitions found on MTD device m25p80
Creating 5 MTD partitions on "m25p80":
0x000000000000-0x0000002e0000 : "boot+cfg+linux"
0x0000002e0000-0x0000007a0000 : "rootfs"
0x0000007a0000-0x0000007c0000 : "cwmp transfer"
0x0000007c0000-0x0000007e0000 : "cwmp notification"
0x0000007e0000-0x000000800000 : "cwmp cacert"
```

부팅 로그에서는 플래시 메모리의 매핑 정보를 확인할 수 있다. 위 로그에 의하면 MTD partition에 부트로더와 커널, 루트 파일시스템(rootfs), cwmp 파일이 존재한다.

![image.png](img/image13.png?raw=true)

`/dev`에는 `mtdblock0`부터 `mtdblock11`까지 장치 파일이 존재한다. 이들 중  부트로더와 커널은 `mtdblock0`에, rootfs는 `mtdblock1`에 있는 것으로 추측된다. 따라서 아래와 같이 dd 명령을 이용해 `mtdblock0`부터 `mtdblock4`까지의 데이터를 추출해 파일로 저장한다.

```bash
./Busybox-mipsel dd if=/dev/mtdblock0 of=/tmp/dump0.bin bs=1 count=9999999
./Busybox-mipsel dd if=/dev/mtdblock1 of=/tmp/dump1.bin bs=1 count=9999999
./Busybox-mipsel dd if=/dev/mtdblock2 of=/tmp/dump2.bin bs=1 count=9999999
./Busybox-mipsel dd if=/dev/mtdblock3 of=/tmp/dump3.bin bs=1 count=9999999
./Busybox-mipsel dd if=/dev/mtdblock4 of=/tmp/dump4.bin bs=1 count=9999999
```

![image.png](img/image14.png?raw=true)

위 명령을 통해 `dump0.bin`부터 `dump4.bin`까지 생성한 후, `cat` 명령을 통해 이들을 합쳐 `dump_result.bin`을 생성한다.

```python
# get_dump_server.py
import socket
import hashlib

host = '0.0.0.0'
port = 8000
dump_value = b''

serv_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
serv_sock.bind((host, port))
serv_sock.listen(5)

clnt_sock, clnt_addr = serv_sock.accept()
print("connected:", clnt_addr)

while len(dump_value) < 4980736:
	dump_value += clnt_sock.recv(4096)

print(len(dump_value))
print(hashlib.md5(dump_value).hexdigest())

f = open('dump.bin', 'wb')
f.write(dump_value)
f.close()
```

생성한 `dump_result.bin`을 PC로 가져오기 위해 TCP 서버를 실행한다. 위 코드는 공유기로부터 파일 데이터를 수신하는 파이썬 스크립트이다.

위 스크립트를 PC에서 실행한 후, 공유기의 쉘에서 아래의 명령을 수행한다.

```python
cat dump_result.bin| ./Busybox-mipsel nc 192.168.10.101 8000
```

![image.png](img/image15.png?raw=true)

![image.png](img/image16.png?raw=true)

위의 첫 번째 이미지는 공유기에 있는 `dump_result.bin`의 해시를, 두 번째 이미지는 수신한 파일의 해시를 보인다. 이 둘의 해시가 동일한 것으로 보아 성공적으로 파일을 전송했음을 확인할 수 있다.

![image.png](img/image17.png?raw=true)

PC에서 성공적으로 데이터를 수신하면, `binwalk`를 이용해 펌웨어에서 커널, 파일 시스템을 추출한다.

![image.png](img/image18.png?raw=true)

![image.png](img/image19.png?raw=true)

파일 시스템(squashfs)은 정상적으로 추출되었으나, 커널은 `dd` 명령을 통해 직접 카빙하여 추출하였다. binwalk에 의하면 tew-831dr은 `Linux Kernel 3.10.9` 을 사용한다.


# 1.2. MEX605

## 1.2.1. 공유기 정보

---

### 하드웨어 정보

- **CPU**: **MediaTek MT7981** (ARM Cortex-A53, 1300MHz)
- **Flash**: GigaDevice NAND Flash

### 소프트웨어 정보

---

- **UART Baud Rate**: 1125000
- **Kernel**: Linux Kernel 5.4.225
- **Busybox**: v1.33.2
- **Firmware Version**: v2.00.11
- **Web Server**: uhttpd

## 1.2.2. 공유기 커버 분리

---

![image.png](img/image20.png?raw=true)

![image.png](img/image21.png?raw=true)

하단의 4개의 나사를 제거한 후, 공유기 커버를 분리한다. 분리 후 PCB 기판을 확인할 수 있는데, 해당 기판에서 UART의 4개의 핀(VCC, GND, TX, RX)을 식별할 수 있다.

## 1.2.3. 부팅 로그 출력

---

부팅 로그 출력을 위해서 USB to TTL과 Tera Term을 이용하였고, 과정은 2.1.4. USB to TTL 연결 및 Tera Term 실행 과 동일하므로 생략한다.

## 1.2.4. binwalk 파일 추출

---

MEX605공유기 같은 경우, 펌웨어 파일을 netis공식 홈페이지에서 다운 받을 수 있고, 암호화되어 있지 않았기에 바로 binwalk로 추출 가능하다.

![image.png](img/image22.png?raw=true)

binwalk -Mre를 이용하여 다운로드한 펌웨어를 추출하였다. 

![image.png](img/image23.png?raw=true)

압축 해제된 파일 시스템, 커널 등을 확인하였다.
