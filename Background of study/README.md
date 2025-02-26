# 연구 배경

공유기 해킹 사고 시 발생할 피해를 줄이고자 취약점을 찾는 방법론을 연구

# 1. 사전 지식

---
# 1.1. UART(Universal Asynchronous Receiver/Transmitter)

---

UART 란 직렬(Serial) 통신이자 클록(Clock)을 사용하지 않는 비동기(Asynchronous) 프로토콜이다.

## 1.1.1. Pins

UART는 아래 4개의 핀을 제공한다.

- **VCC**: 전원 공급용
- **GND**: 접지
- **TX**: 칩 기준 데이터 송신부
- **RX**: 칩 기준 데이터 수신부

![image.png](img/image1.png?raw=true)

이때 TX와 RX는 위와 같이 데이터를 주고받는 디바이스끼리 교차해서 연결해야 한다.

![image.png](img/image2.png?raw=true)

![image.png](img/image3.png?raw=true)

UART의 각 핀은 왼쪽 그림과 같이 되어 있다. 이를 PC의 USB와 연결하기 위해서는 우측 이미지인 `USB to TTL`이 필요하다.

## 1.1.2. Baud Rate

`Baud Rate`는 시리얼 통신에서 사용하는 데이터의 전송 속도 단위이다.  정확히 초당 전송되는 심볼의 수를 말한다. 만약 심볼 1개가 2bit이고 baud rate가 9600이라면, 초당 19200bit를 전송하는 셈이다.

비동기 통신에서 수신자는 송신자가 얼마나 빠른 속도로 데이터를 전달할 지 모르기 때문에, 그 속도를 미리 약속한 후 통신해야 원활한 송수신이 가능하다. 물론 일부 동기(Synchronous) 통신에서도 baud rate를 미리 정해두고 통신한다(USART).

![image.png](img/image4.png?raw=true)

주로 사용되는 Baud Rate는 위와 같다. 보통 9600~115200의 값을 사용한다. 만약 UART 통신을 시도할 경우, 위 표의 값을 하나씩 설정하다 보면 터미널에 문자가 정상적으로 출력될 것이다.

## 1.1.3. UART 핀 식별

![image.png](img/image5.png?raw=true)

![image.png](img/image6.png?raw=true)

만약 위 이미지와 같이 PCB 기판에서 UART의 각 핀의 역할을 표기하지 않았다면, `멀티미터(Multimeter)`라는 장비로 식별할 수 있다. 핀의 특성에 따라 전압과 전류가 다르므로, 멀티미터를 이용해 전압과 전류를 측정한다면 각 핀을 구분할 수 있다. 이 외의 다른 방법으로는 PCB의 데이터 시트를 확인하는 것이지만, 본문에서는 멀티미터를 이용한 핀의 식별 방법을 기술한다.

### 전압 측정으로 GND 식별하기

멀티미터는 `검정 팁`과 `빨강 팁`이 있다. 먼저 전압을 이용해 GND를 식별할 수 있는데, 그 방법은 4개의 핀 중 두 개씩 짝지어서 전압을 측정하는 것이다. 빨강, 검정의 구분 없이 각 팁을 두 핀에 두어 전압을 측정하다 보면, 3.3V 혹은 5V의 전압이 측정될 수 있다. 이때 검정 팁에 있는 핀이 바로 GND이다. 만약 전압이 -3.3V 혹은 -5V로 측정되었다면, 빨강 팁에 있는 핀이 GND가 된다.

### 전류, 전압 측정으로 VCC, TX, RX 식별하기

나머지 세 개의 핀은 전압과 전류를 모두 측정하여 구분한다. 먼저 식별된 GND 핀에 검정 팁을 두고, 나머지 세 핀에 빨강 팁을 차례로 두어 전압과 전류를 측정 및 기록한다.

|  | VCC | TX | RX |
| --- | --- | --- | --- |
| 전류 | 0mA | 23mA~ | 매우 낮음 |
| 전압 | 3.3V | 3.3V | 0V~3.3V |

VCC는 전압이 3.3V 가까이 측정되지만, 전류는 0mA로 측정된다. TX도 마찬가지로 3.3V의 전압이 측정되나, 전류가 23mA 이상의 값이 측정된다. 그 이유는 장치의 부팅 시 부팅 로그를 전송하기 때문이다. 반면에 RX는 전류가 0mA에 가깝게 측정되고, 전압은 장치의 풀업/풀다운 저항에 따라 3.3V 또는 0V로 측정된다.

# 1.2. Firmware & Binwalk

---

## 1.2.1. Firmware와 추출 방법

`펌웨어(Firmware)`는 공유기, 냉장고, 스마트폰 등 임베디드 시스템 내에서 동작하는 소프트웨어를 말한다. 보통 펌웨어는 부트로더, 커널, 파일시스템을 포함하였으며, 펌웨어는 보통 플래시 메모리에 저장된다.

장치에서 제공하는 서비스를 분석하기 위해서는 서비스 바이너리를 얻어야 한다. 그런데 이 바이너리는 펌웨어에 포함되어 있으므로, 결국 펌웨어를 추출해야 원활한 분석이 가능하다. 아래는 펌웨어를 추출하는 대표적인 세 가지 방법이다.

- 디버깅 포트 활용: UART를 통한 디버깅 및 쉘 획득. 제조사에 의해 디버깅 포트가 막힐 수도 있음
- 공개 펌웨어 조사: 온라인에서 얻을 수 있는 펌웨어 조사
- 플래시 덤프: 플래시 메모리를 직접 분리 후 플래시 메모리의 프로토콜(SPI/MMC/UFS)을 이용해 데이터 추출

## 1.2.2. binwalk 소개

`binwalk`는 펌웨어를 분석하는 유틸리티로, 펌웨어가 포함하고 있는 바이너리, 이미지, 파일시스템 등 파일을 분석 및 추출해 준다. github에서 오픈소스로 제공되고 있으며, Debian 기준 apt 패키지로도 설치할 수 있다.

```bash
# apt를 통해 설치
sudo apt install binwalk
```

```bash
# 직접 빌드
sudo apt install curl git
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
. $HOME/.cargo/env
git clone https://github.com/ReFirmLabs/binwalk
sudo ./binwalk/dependencies/ubuntu.sh
cd binwalk
cargo build --release
```

## 1.2.3. binwalk 옵션

### -A: Firwmare Architecture

![image.png](img/image7.png?raw=true)

`-A` 옵션은 머신 코드의 아키텍처를 분석한다. 위 이미지에서는 MEX605의 아키텍처가 ARMEB(ARM Big Endian)임을 알 수 있다.

### -B: File Signature

![image.png](img/image8.png?raw=true)

`-B` 옵션은 바이너리에 포함된 파일의 파일 시그니처를 분석한다. 이 옵션은 기본으로 활성화되어 있으므로 사용하지 않아도 자동 적용된다.

### -e: Extraction

![image.png](img/image9.png?raw=true)

`-e` 옵션은 바이너리에서 식별된 파일을 추출한다.  위 이미지는 바이너리에 포함돼 있던 tar 아카이브 파일을 카빙한 후 압축 해제한 것이다.

### -Mre: Recursive Extraction

![image.png](img/image12.png?raw=true)

![image.png](img/image10.png?raw=true)

`-Mre` 옵션에서 `-M` 옵션(Matryoshka)은 파일 추출 시 추출된 파일에 대해 한 번 더 카빙(carving)한다. `-e` 옵션과 달리, kernel과 root, root_ext도 카빙된 것을 확인할 수 있다.

### -E: Entropy

![image.png](img/image11.png?raw=true)

`-E` 옵션은 바이너리의 엔트로피(Entropy)를 계산하여 데이터의 무작위성을 그래프로 표시한다. 이 값을 통해 해당 바이너리의 압축 또는 암호화 여부를 확인할 수 있다. 암호화되어 있지 않은 평문(ASCII)이나 데이터들은 엔트로피가 낮지만, 압축 및 암호화된 데이터의 엔트로피는 매우 크다.
