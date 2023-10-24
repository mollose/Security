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

</br>

### Modes of Operation
IA-32 프로세서의 mode of operation은 그것이 지원할 기능을 결정. 루트킷 구현에 있어 다음 세 가지 IA-32 모드가 고려 대상
* Real mode : Intel 8086(16비트 레지스터, 16비트 데이터 버스, 20비트 어드레스 버스) / 88 프로세서의 16비트 실행 환경 구현. IA-32 머신 가동 시, real mode에서 작동을 시작(IA-32 머신을 여전히 DOS 부트 디스크로 부팅할 수 있는 이유)
* Protected mode : Windows 7과 같은, 지금의 시스템 소프트웨어를 구동할 수 있는 환경 구현. 머신이 real mode로 구동을 시작한 후, 필수 데이터 구조체들을 셋업한 후에 OS는 프로세스를 protected mode로 전환하여 하드웨어가 제공하는 기능들이 작동할 수 있게 함
* System management mode(SMM) : 주로 펌웨어 내의 특수한 코드를 실행(비상 셧다운, 전원 관리, 시스템 보안 등. SMM을 활용한 루트킷 구현이 공개적으로 논의되어왔음)

</br>

### Real Mode
20비트 주소 공간 사용. real mode에서 한 바이트의 논리 주소는 16비트 segment selector와 16비트 effective address로 구성됨. selector는 64KB 메모리 세그먼트의 베이스 주소값(값을 왼쪽으로 4비트 시프트)을 담고 있으며 effective address는 이 세그먼트의 오프셋으로 사용됨. effective address는 selector에 더해져 해당 바이트의 실제 물리 주소를 나타냄. 이 모드에선 메모리 보호가 지원되지 않으므로 유저 애플리케이션이 내부 OS를 변경하는 것을 방지할 수 없음

* real-mode OS의 대표적 예는 MS-DOS로, 특수한 드라이버 없이 DOS는 20비트 주소 공간으로 제한됨. 첫 640KB(0x00000 ~ 0x9FFFF) 영역은 conventional memory로, 이 공간의 양호한 청크들은 시스템 레벨 코드로 채워짐. 한편 1MB 경계까지의 나머지 영역(0xA0000 ~ 0xFFFFF)은 upper memory area(UMA)로, ROM이나 주변 장치의 RAM 같은 하드웨어에 의해 사용되도록 예약된 공간. UMA 내에는 대개 하드웨어에 의해 사용되지 않는 DOS-accessible RAM 슬롯들이 있는데 이들을 upper memory blocks(UMBs)로 부름. 1MB 상위의 메모리 공간은 extended memory로 불리며, 80386과 같은 프로세서들이 출시되었을 당시 real-mode 프로그램들이 extended memory에 접근 가능하게 하는 DOS extender들이 존재했음
* 변경된 부트 섹터 코드를 통해 OS를 선취하여 루트킷을 메모리로 들여오는 경우 BIOS에 의해 제공되는 서비스에 의존해야 할 수 있는데, BIOS는 real mode에서 작동(커널이 메모리 보호를 통해 그 자신을 격리하기 전에 작업을 쉽게 해치울 수 있게 함)

#### Real Mode 실행 환경
현재의 real-mode 환경은 8086 / 88 프로세서의 기능에 맞춰져 있음. 6개의 세그먼트 레지스터, 4개의 범용 레지스터, 3개의 포인터 레지스터와 두 개의 인덱싱 레지스터, 그리고 FLAGS 레지스터로 구성되며, 이들 모두는 16비트 사이즈. 세그먼트 레지스터(CS, DS, SS, ES)는 segment selectors를 저장하고 8086 / 88 이후의 프로세스들에 존재하는 FS, GS 레지스터 역시 segment selectors 저장. 포인터 레지스터(IP, SP, BP)는 effective addresses를 저장. 범용 레지스터(AX, BX, CX, DX)는 다양한 operand와 주소값을 저장 가능하며 또한 별도의 특수한 목적을 갖고 있음(AX는 산술 연산에서의 Accumulator, BX는 Base Register로 메모리를 간접적으로 지시하기 위한 인덱스로 사용됨. CX는 Counter와 loop index, DX는 Data Register로 AX와 함께 연산에 사용). 인덱싱 레지스터(SI, DI)는 indexed addressing을 구현하기 위해 사용되며 문자열과 산술 연산에도 사용됨. FLAGS 레지스터는 CPU의 상태나 특정 연산 결과를 나타냄. 16비트 중 9비트만이 플래그로 사용되며, trap flag인 TF(bit 8)와 interrupt enable flag인 IF(bit 9) 포함. 만약 TF가 세팅되면 프로세서는 single-step 인터럽트를 각 명령어 이후마다 발생시킴. 만약 IF가 세팅되면 인터럽트를 받아들이는 대로 승인 및 실행. 윈도우는 아직까지도 16비트 머신 코드 디버거(debug.exe)를 내장하고 있음

#### Real Mode 인터럽트
인터럽트의 각 특정 유형마다 그 유형의 인터럽트를 적절한 ISR과 연결시키는 정수 값이 할당됨. 인터럽트가 어떻게 처리될지의 방식은 real mode와 protected mode 각각 다름. real mode에서, 메모리의 첫 1KB(0x00000 ~ 0x003FF)는 interrupt vector table(IVT)이라는 데이터 구조체가 차지(protected mode의 interrupt descriptor table(IDT)과 동일한 역할). IVT와 IDT 모두 ISR의 메모리상 위치를 지정하는 interrupt descriptor 또는 interrupt vector들로 이루어져 있음. real mode에선 IVT는 각 ISR의 논리 주소를 연속적으로 저장해두고 있음. 각 interrupt vector는 interrupt type 0부터 256까지 순서대로 0x00000에서 시작하여 4바이트씩을 차지하고, 각 vector는 ISR의 effective address와 segment selector로 이루어짐(두 값 모두 주소의 낮은 바이트가 앞에 옴(little endian)). MS-DOS에선 BIOS가 interrupt 0 ~ 31을 사용하고 DOS가 32 ~ 63을 사용(DOS system call 인터페이스는 모두 근본적으로 interrupt들로 이뤄져 있음). 나머지(64 ~ 255)는 유저 정의 interrupt

3가지 타입의 인터럽트가 존재
* Hardware interrupt : 외부 디바이스에 의해 예상치 못하게 발생. CLI 명령어로 IF를 클리어함으로써 해제 가능한 maskable interrupt와 해제 불가능한 nonmaskable interrupt로 나뉨
* Software interrupt : INT 명령어를 사용하는 프로그램에 의해 구현됨. INT 명령어는 interrupt vector를 지정하는 하나의 operand를 가짐. INT 명령어 실행 시 TF와 IF를 클리어함. FLAGS, CS, IP 레지스터를 순서대로 스택에 저장한 뒤, 인터럽트 벡터를 통해 ISR의 주소로 이동. IRET 명령어를 만날 때까지 ISR의 코드를 실행. **IRET은 스택에 저장된 값들을 다시 꺼낸 후** INT 명령 이후의 명령어들에 대해 실행 재개
* Exception : 프로세서가 명령어 실행 도중 faults, traps, aborts 등의 에러를 감지했을 때 발생
  * fault : fault 발생 시, 프로세서는 예외를 발생시킨 명령어 앞의 명령어 경계에서 예외를 보고. 따라서 프로그램의 상태는 예외 이전의 상태로 리셋될 수 있고 명령어 재실행 가능. divided by zero(0) 등이 fault에 해당
  * trap : 명령어 재실행 불가. 프로세서는 예외를 발생시킨 명령어 뒤의 경계에서 예외를 보고. breakpoint(3), overflow(4) 등이 fault에 해당
  * abort : abort 발생 시 프로그램은 실행을 재개할 수 없으며 그대로 중단    

#### 세그먼테이션과 프로그램 제어 
Real mode는 세그먼테이션을 사용하므로 프로그램 제어에 있어 같은 세그먼트 내에서 점프하는지(intrasegment), 한 세그먼트에서 다른 세그먼트로 이동하는지(intersegment)가 명시되어야 함. 점프 명령어들은 near와 far로 분류될 수 있는데 near 점프는 주어진 세그먼트 내에서, far 점프는 세그먼트 간 이동 시 발생(INT / IRET은 far 점프)
* short jump : signed byte 변위를 지님. 이는 IP 레지스터의 현재 값에 더해짐
* near jump : short jump와 유사하나 signed word(little endian) 변위를 지님
* far jump : Far direct jump의 경우 segment selector와 effective address 모두를 명시하는 32비트 operand를 지님(short, near 점프와 달리 목적지의 주소를 명시적으로 지정)

다음과 같은 공격 기법들이 존재함
* Call table hooking : 프로그램의 제어를 얻기 위해 주소 테이블을 변경하는 것
* Direct Kernel Object Manipulation(DKOM) : 시스템 데이터 구조체 변경

</br>

### Protect Mode
Real mode와 마찬가지로 segment memory 모델의 한 예이지만, 프로세서 홀로 물리 메모리 해상 과정을 수행하지 않고 OS 내의 특수한 테이블의 작용이 이 과정을 돕는다는 차이점이 있음. 이러한 특성이 OS에게는 추가적인 업무 수행의 부담을 지우지만, 특수한 테이블들이 메모리 보호, 페이징 요구 등의 추가적 기능들을 담당하면서 프로세서는 컴퓨팅 수행 그 자체에 집중할 수 있게 됨

#### Protected Mode 실행 환경
16비트 세그먼트 레지스터를 제외하고, real mode에 존재하던 레지스터들의 32비트 확장이 이루어짐. 또한 실행 환경 제어를 위한 특정 목적의 레지스터들이 추가됨. 이들엔 5개의 제어 레지스터(CR0 ~ CR4), global descriptor table 레지스터(GDTR), local descriptor table 레지스터(LDTR), interrupt descriptor table 레지스터(IDTR)가 포함됨. Real mode와는 달리, 세그먼트 레지스터들은 segment selector들을 담지만 물리 메모리의 64KB 세그먼트와 연관돼 있진 않음. 대신 이들은 선형 주소 공간 내의 세그먼트를 나타내는, segment descriptor라는 테이블 엔트리를 인덱싱하는 여러 필드로 구성된 바이너리 구조를 저장하고 있음(CS는 현재 실행 코드의 세그먼트의 segment descriptor를, DS ~ GS는 프로그램 데이터 세그먼트의 segment descriptor를, SS는 스택 세그먼트의 segment descriptor를 저장). 6개의 세그먼트 레지스터 중 유일하게 CS 레지스터는 명시적으로 세팅될 수 없으며, 대신 프로그램 제어와 관련된 명령어(JMP, CALL, INT, RET, IRET, SYSENTER, SYSEXIT 등)에 의해 암시적으로 세팅되어야 함

#### Protected Mode 세그먼테이션
IA-32 프로세서는 메모리 보호의 구현에 Segmentation과 Paging의 두 가지 기능을 이용(Protected mode에서 Segmentation이 필수적이라면 Paging은 선택적이며, 또한 Paging은 Segmentation 위에서 구현됨). segment selector는 16비트 사이즈이며 effective address는 32비트 사이즈. segment selector는 선형 주소 공간 내의 세그먼트를 나타내는 테이블 엔트리 참조. segment selector는 선형 주소 공간 내의 세그먼트에 대한 세부 사항들을 담고 있는 바이너리 구조를 인덱싱. 이 테이블은 descriptor table로 불리며 이것의 엔트리들은 segment descriptor라고 불림. segment descriptor(64비트)는 선형 주소 공간 내의 세그먼트에 대한 메타데이터(접근 권한, 크기, 32비트 base 선형 주소 등)를 포함. 이때 세그먼트의 32비트 base 선형 주소는 프로세서에 의해 디스크립터에서 추출되어 effective address가 제공하는 오프셋에 더해져 선형 주소를 나타내는데 사용됨(base 선형 주소와 offset 주소는 모두 32비트 값으로, 따라서 protected mode의 선형 주소 공간 크기는 4GB). descriptor table에는 두 종류가 있는데 GDT와 LDT가 바로 그것. GDT를 갖는 것은 필수 사항으로, IA-32상에서 구동되는 OS는 시작 시 반드시 하나의 GDT를 만들어야 함. 전체 시스템 작업 간에 공유될 수 있는 하나의 GDT가 존재할 것(GDT의 첫 엔트리는 항상 비어있으며 null segment descriptor로 불림. 이 descriptor를 참조하는 selector는 null selector로 불림). LDT의 사용은 선택적으로, 하나의 작업이나 관련된 작업들의 그룹에서 사용 가능. 한편 48비트 크기의 GDTR 레지스터는 GDT의 base 주소를 갖고 있으며 하위 16비트는 GDT의 사이즈를, 나머지 32비트는 GDT의 base 선형 주소를 나타냄. LGDT 명령어는 GDTR에 특정 값을 적재하며, SGDT 명령어는 GDTR의 값을 읽음. LGDT 명령어는 privileged 명령어로, OS에 의해서만 실행될 수 있음. segment selector는 3가지 필드로 이루어진 16비트 값. 상위 13비트는 GDT에 대한 인덱스로, 따라서 GDT는 최대 2<sup>13</sup>개의 segment descriptor를 가질 수 있음. 최하위 2비트는 selector의 request privilege level(RPL)을 나타내며 00, 01, 10, 11의 4가지 값을 갖는데, 0이 가장 높은 privilege level을, 3이 가장 낮은 level을 의미. 마지막으로 남은 Bit 2는 1일 때 LDT 내의 descriptor를, 0일 때 GDT 내의 descriptor를 명시. 64비트 구조체인 segment descriptor는 수많은 필드들로 이루어져 있는데 이 중 세 가지인 base address(세그먼트 32비트 base 선형 주소), type, S, DPL 필드에 집중. Descriptor privilege level(DPL)은 참조되는 세그먼트의 privilege level 정의. RPL과 마찬가지로 0 ~ 3의 값을 가짐(Privilege level은 4개의 특권 영역을 나타내는 3개의 동심원으로 표현되기도 함(Ring 0 ~ 3의 4개의 영역). DPL이 0인 세그먼트를 두고 Ring 0 내부에 존재한다고 말하기도 함. OS 커널은 Ring 0에서 실행되며 유저 애플리케이션은 Ring 3에서 실행). type 필드와 S 필드는 함께 사용되어 descriptor의 종류를 결정. S 플래그는 특히 segment descriptor의 두 부류를 결정하는데, S = 1일 때 코드와 데이터 segment descriptor를, S = 0일 때 시스템 segment descriptor를 의미. 코드와 데이터 segment descriptor는 일반 애플리케이션용 세그먼트를 참조. 시스템 segment descriptor는 현재 실행 작업의 privilege level(current privilege level(CPL))보다 높은 레벨의 세그먼트들로 점프하기 위해 사용됨. 한편 type 필드는 세그먼트의 타입(코드 또는 데이터), 접근, 자라나는 방향의 타입을 결정. Segment Limit 필드는 세그먼트의 크기 결정(G 플래그가 0이면 1byte ~ 1MB, 1이면 4KB ~ 4GB)
