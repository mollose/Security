# The Rootkit Arsenal

## 목차

</br>

## Hardware Briefing
프로세서에 의해 메모리가 논리적으로 구성되는데 있어 두 가지 scheme 존재
* Flat Memory Model : physical model과 달리 추상적인 모델로서, 메모리를 인접한 바이트들의 연속으로 나타냄(0으로 시작하고 N(IA-32의 경우 2<sup>32</sup> - 1)으로 끝나는 주소로 표현). 특정 바이트의 주소를 선형 주소로 부르며, 가능한 바이트들의 전체 범위를 선형 주소 공간이라 부름. Flat Memory Model은 물리 메모리와 거의 동일하지만 항상 그러한 것은 아닌데, 예를 들어 전체 메모리 보호 체계 수립 시 선형 주소가 물리 메모리와 전혀 유사하지 않은 전체 주소 변환 과정 중에 사용될 수 있음
* Segmented Memory Model : 메모리는 세그먼트로 불리는, 구분되는 영역들의 관점에서 보여짐. 특정 세그먼트 내에 위치하는 바이트는 논리 주소에 의해 지정됨. 논리 주소(far pointer)는 두 부분으로 나뉘는데, 참조되는 세그먼트를 결정하는 segment selector와 세그먼트 내에서의 바이트의 위치를 명시하는 effective address가 바로 그것. segment selector와 effective address의 실제 내용은 주소 변환 체계마다 다를 수 있음

IA-32 프로세서에는 액세스를 위한 32개의 address line들 외에도, 물리 메모리에 접근하고 이를 업데이트하기 위한 제어 버스와 데이터 버스 존재. 제어(32비트 데이터 청크) 버스는 프로세스가 메모리에 쓰기, 읽기를 원하는지를 나타내며 데이터 버스는 프로세서와 메모리 간 데이터를 주고받는데 사용됨(IA-32 프로세서는 한 번에 4바이트 데이터를 읽고 쓸 수 있음)
* 읽기 동작 : 프로세서의 address line 주소 세팅 ⇒ 제어 버스로 읽기 신호 보냄 ⇒ RAM 칩에서 데이터 버스로 요청된 바이트 반환
* 쓰기 동작 : 프로세서의 address line 주소 세팅 ⇒ 제어 버스로 쓰기 신호 보냄 ⇒ 프로세서에서 데이터 버스로 쓰일 바이트 전송
* Address line은 프로세서를 RAM 칩과 연결하는 선의 집합으로 각 address line은 각 바이트(IA-32 계열 프로세서는 물리 주소를 사용해 물리 메모리상의 각 8비트 바이트에 접근)의 주소의 단일 비트를 명시. IA-32 프로세서는 기본적으로 32개의 address line들을 사용하며 이는 최대 4GB 물리 주소 공간을 나타낼 수 있음. IA-32 프로세서에서 물리 주소 확장(PAE) 기능을 사용하면 52개의 address line까지 사용 가능

### Modes of Operation
IA-32 프로세서의 mode of operation은 그것이 지원할 기능을 결정. 루트킷 구현에 있어 다음 세 가지 IA-32 모드가 고려 대상
* Real mode : Intel 8086(16비트 레지스터, 16비트 데이터 버스, 20비트 어드레스 버스) / 88 프로세서의 16비트 실행 환경 구현. IA-32 머신 가동 시, real mode에서 작동을 시작(IA-32 머신을 여전히 DOS 부트 디스크로 부팅할 수 있는 이유)
* Protected mode : Windows 7과 같은, 지금의 시스템 소프트웨어를 구동할 수 있는 환경 구현. 머신이 real mode로 구동을 시작한 후, 필수 데이터 구조체들을 셋업한 후에 OS는 프로세스를 protected mode로 전환하여 하드웨어가 제공하는 기능들이 작동할 수 있게 함
* System management mode(SMM) : 주로 펌웨어 내의 특수한 코드를 실행(비상 셧다운, 전원 관리, 시스템 보안 등. SMM을 활용한 루트킷 구현이 공개적으로 논의되어왔음)

### Real Mode
20비트 주소 공간 사용. real mode에서 한 바이트의 논리 주소는 16비트 segment selector와 16비트 effective address로 구성됨. selector는 64KB 메모리 세그먼트의 베이스 주소값(값을 왼쪽으로 4비트 시프트)을 담고 있으며 effective address는 이 세그먼트의 오프셋으로 사용됨. effective address는 selector에 더해져 해당 바이트의 실제 물리 주소를 나타냄. 이 모드에선 메모리 보호가 지원되지 않으므로 유저 애플리케이션이 내부 OS를 변경하는 것을 방지할 수 없음

* real-mode OS의 대표적 예는 MS-DOS로, 특수한 드라이버 없이 DOS는 20비트 주소 공간으로 제한됨. 첫 640KB(0x00000 ~ 0x9FFFF) 영역은 conventional memory로, 이 공간의 양호한 청크들은 시스템 레벨 코드로 채워짐. 한편 1MB 경계까지의 나머지 영역(0xA0000 ~ 0xFFFFF)은 upper memory area(UMA)로, ROM이나 주변 장치의 RAM 같은 하드웨어에 의해 사용되도록 예약된 공간. UMA 내에는 대개 하드웨어에 의해 사용되지 않는 DOS-accessible RAM 슬롯들이 있는데 이들을 upper memory blocks(UMBs)로 부름. 1MB 상위의 메모리 공간은 extended memory로 불리며, 80386과 같은 프로세서들이 출시되었을 당시 real-mode 프로그램들이 extended memory에 접근 가능하게 하는 DOS extender들이 존재했음
* 변경된 부트 섹터 코드를 통해 OS를 선취하여 루트킷을 메모리로 들여오는 경우 BIOS에 의해 제공되는 서비스에 의존해야 할 수 있는데, BIOS는 real mode에서 작동(커널이 메모리 보호를 통해 그 자신을 격리하기 전에 작업을 쉽게 해치울 수 있게 함)
