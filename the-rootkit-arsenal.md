# The Rootkit Arsenal

## 목차

<br/><br/>

## Hardware Briefing
프로세서에 의해 메모리가 논리적으로 구성되는데 있어 두 가지 scheme 존재
* Flat Memory Model : physical model과 달리 추상적인 모델로서, 메모리를 인접한 바이트들의 연속으로 나타냄(0으로 시작하고 N(IA-32의 경우 2<sup>32</sup> - 1)으로 끝나는 주소로 표현). 특정 바이트의 주소를 선형 주소로 부르며, 가능한 바이트들의 전체 범위를 선형 주소 공간이라 부름. Flat Memory Model은 물리 메모리와 거의 동일하지만 항상 그러한 것은 아닌데, 예를 들어 전체 메모리 보호 체계 수립 시 선형 주소가 물리 메모리와 전혀 유사하지 않은 전체 주소 변환 과정 중에 사용될 수 있음
* Segmented Memory Model : 메모리는 세그먼트로 불리는, 구분되는 영역들의 관점에서 보여짐. 특정 세그먼트 내에 위치하는 바이트는 논리 주소에 의해 지정됨. 논리 주소(far pointer)는 두 부분으로 나뉘는데, 참조되는 세그먼트를 결정하는 segment selector와 세그먼트 내에서의 바이트의 위치를 명시하는 effective address가 바로 그것. segment selector와 effective address의 실제 내용은 주소 변환 체계마다 다를 수 있음

IA-32 프로세서에는 액세스를 위한 32개의 address line들 외에도, 물리 메모리에 접근하고 이를 업데이트하기 위한 제어 버스와 데이터 버스 존재. 제어(32비트 데이터 청크) 버스는 프로세스가 메모리에 쓰기, 읽기를 원하는지를 나타내며 데이터 버스는 프로세서와 메모리 간 데이터를 주고받는데 사용됨(IA-32 프로세서는 한 번에 4바이트 데이터를 읽고 쓸 수 있음)
* 읽기 동작 : 프로세서의 address line 주소 세팅 ⇒ 제어 버스로 읽기 신호 보냄 ⇒ RAM 칩에서 데이터 버스로 요청된 바이트 반환
* 쓰기 동작 : 프로세서의 address line 주소 세팅 ⇒ 제어 버스로 쓰기 신호 보냄 ⇒ 프로세서에서 데이터 버스로 쓰일 바이트 전송
* Address line은 프로세서를 RAM 칩과 연결하는 선의 집합으로 각 address line은 각 바이트(IA-32 계열 프로세서는 물리 주소를 사용해 물리 메모리상의 각 8비트 바이트에 접근)의 주소의 단일 비트를 명시. IA-32 프로세서는 기본적으로 32개의 address line들을 사용하며 이는 최대 4GB 물리 주소 공간을 나타낼 수 있음. IA-32 프로세서에서 물리 주소 확장(PAE) 기능을 사용하면 52개의 address line까지 사용 가능

<br/>

### Modes of Operation
IA-32 프로세서의 mode of operation은 그것이 지원할 기능을 결정. 루트킷 구현에 있어 다음 세 가지 IA-32 모드가 고려 대상
* Real mode : Intel 8086(16비트 레지스터, 16비트 데이터 버스, 20비트 어드레스 버스) / 88 프로세서의 16비트 실행 환경 구현. IA-32 머신 가동 시, real mode에서 작동을 시작(IA-32 머신을 여전히 DOS 부트 디스크로 부팅할 수 있는 이유)
* Protected mode : Windows 7과 같은, 지금의 시스템 소프트웨어를 구동할 수 있는 환경 구현. 머신이 real mode로 구동을 시작한 후, 필수 데이터 구조체들을 셋업한 후에 OS는 프로세스를 protected mode로 전환하여 하드웨어가 제공하는 기능들이 작동할 수 있게 함
* System management mode(SMM) : 주로 펌웨어 내의 특수한 코드를 실행(비상 셧다운, 전원 관리, 시스템 보안 등. SMM을 활용한 루트킷 구현이 공개적으로 논의되어왔음)

<br/>

### Real Mode
20비트 주소 공간 사용. real mode에서 한 바이트의 논리 주소는 16비트 segment selector와 16비트 effective address로 구성됨. selector는 64KB 메모리 세그먼트의 베이스 주소값(값을 왼쪽으로 4비트 시프트)을 담고 있으며 effective address는 이 세그먼트의 오프셋으로 사용됨. effective address는 selector에 더해져 해당 바이트의 실제 물리 주소를 나타냄. 이 모드에선 메모리 보호가 지원되지 않으므로 유저 애플리케이션이 내부 OS를 변경하는 것을 방지할 수 없음

* real-mode OS의 대표적 예는 MS-DOS로, 특수한 드라이버 없이 DOS는 20비트 주소 공간으로 제한됨. 첫 640KB(0x00000 ~ 0x9FFFF) 영역은 conventional memory로, 이 공간의 양호한 청크들은 시스템 레벨 코드로 채워짐. 한편 1MB 경계까지의 나머지 영역(0xA0000 ~ 0xFFFFF)은 upper memory area(UMA)로, ROM이나 주변 장치의 RAM 같은 하드웨어에 의해 사용되도록 예약된 공간. UMA 내에는 대개 하드웨어에 의해 사용되지 않는 DOS-accessible RAM 슬롯들이 있는데 이들을 upper memory blocks(UMBs)로 부름. 1MB 상위의 메모리 공간은 extended memory로 불리며, 80386과 같은 프로세서들이 출시되었을 당시 real-mode 프로그램들이 extended memory에 접근 가능하게 하는 DOS extender들이 존재했음
* 변경된 부트 섹터 코드를 통해 OS를 선취하여 루트킷을 메모리로 들여오는 경우 BIOS에 의해 제공되는 서비스에 의존해야 할 수 있는데, BIOS는 real mode에서 작동(커널이 메모리 보호를 통해 그 자신을 격리하기 전에 작업을 쉽게 해치울 수 있게 함)

#### <ins>Real Mode 실행 환경</ins>
현재의 real-mode 환경은 8086 / 88 프로세서의 기능에 맞춰져 있음. 6개의 세그먼트 레지스터, 4개의 범용 레지스터, 3개의 포인터 레지스터와 두 개의 인덱싱 레지스터, 그리고 FLAGS 레지스터로 구성되며, 이들 모두는 16비트 사이즈. 세그먼트 레지스터(CS, DS, SS, ES)는 segment selectors를 저장하고 8086 / 88 이후의 프로세스들에 존재하는 FS, GS 레지스터 역시 segment selectors 저장. 포인터 레지스터(IP, SP, BP)는 effective addresses를 저장. 범용 레지스터(AX, BX, CX, DX)는 다양한 operand와 주소값을 저장 가능하며 또한 별도의 특수한 목적을 갖고 있음(AX는 산술 연산에서의 Accumulator, BX는 Base Register로 메모리를 간접적으로 지시하기 위한 인덱스로 사용됨. CX는 Counter와 loop index, DX는 Data Register로 AX와 함께 연산에 사용). 인덱싱 레지스터(SI, DI)는 indexed addressing을 구현하기 위해 사용되며 문자열과 산술 연산에도 사용됨. FLAGS 레지스터는 CPU의 상태나 특정 연산 결과를 나타냄. 16비트 중 9비트만이 플래그로 사용되며, trap flag인 TF(bit 8)와 interrupt enable flag인 IF(bit 9) 포함. 만약 TF가 세팅되면 프로세서는 single-step 인터럽트를 각 명령어 이후마다 발생시킴. 만약 IF가 세팅되면 인터럽트를 받아들이는 대로 승인 및 실행. 윈도우는 아직까지도 16비트 머신 코드 디버거(debug.exe)를 내장하고 있음

#### <ins>Real Mode 인터럽트</ins>
인터럽트의 각 특정 유형마다 그 유형의 인터럽트를 적절한 ISR과 연결시키는 정수 값이 할당됨. 인터럽트가 어떻게 처리될지의 방식은 real mode와 protected mode 각각 다름. real mode에서, 메모리의 첫 1KB(0x00000 ~ 0x003FF)는 interrupt vector table(IVT)이라는 데이터 구조체가 차지(protected mode의 interrupt descriptor table(IDT)과 동일한 역할). IVT와 IDT 모두 ISR의 메모리상 위치를 지정하는 interrupt descriptor 또는 interrupt vector들로 이루어져 있음. real mode에선 IVT는 각 ISR의 논리 주소를 연속적으로 저장해두고 있음. 각 interrupt vector는 interrupt type 0부터 256까지 순서대로 0x00000에서 시작하여 4바이트씩을 차지하고, 각 vector는 ISR의 effective address와 segment selector로 이루어짐(두 값 모두 주소의 낮은 바이트가 앞에 옴(little endian)). MS-DOS에선 BIOS가 interrupt 0 ~ 31을 사용하고 DOS가 32 ~ 63을 사용(DOS system call 인터페이스는 모두 근본적으로 interrupt들로 이뤄져 있음). 나머지(64 ~ 255)는 유저 정의 interrupt

3가지 타입의 인터럽트가 존재

##### Hardware interrupt
외부 디바이스에 의해 예상치 못하게 발생. CLI 명령어로 IF를 클리어함으로써 해제 가능한 maskable interrupt와 해제 불가능한 nonmaskable interrupt로 나뉨

##### Software interrupt
INT 명령어를 사용하는 프로그램에 의해 구현됨. INT 명령어는 interrupt vector를 지정하는 하나의 operand를 가짐. INT 명령어 실행 시 TF와 IF를 클리어함. FLAGS, CS, IP 레지스터를 순서대로 스택에 저장한 뒤, 인터럽트 벡터를 통해 ISR의 주소로 이동. IRET 명령어를 만날 때까지 ISR의 코드를 실행. **IRET은 스택에 저장된 값들을 다시 꺼낸 후** INT 명령 이후의 명령어들에 대해 실행 재개

##### Exception
프로세서가 명령어 실행 도중 faults, traps, aborts 등의 에러를 감지했을 때 발생
  * fault : fault 발생 시, 프로세서는 예외를 발생시킨 명령어 앞의 명령어 경계에서 예외를 보고. 따라서 프로그램의 상태는 예외 이전의 상태로 리셋될 수 있고 명령어 재실행 가능. divided by zero(0) 등이 fault에 해당
  * trap : 명령어 재실행 불가. 프로세서는 예외를 발생시킨 명령어 뒤의 경계에서 예외를 보고. breakpoint(3), overflow(4) 등이 fault에 해당
  * abort : abort 발생 시 프로그램은 실행을 재개할 수 없으며 그대로 중단    

#### <ins>세그먼테이션과 프로그램 제어</ins>
Real mode는 세그먼테이션을 사용하므로 프로그램 제어에 있어 같은 세그먼트 내에서 점프하는지(intrasegment), 한 세그먼트에서 다른 세그먼트로 이동하는지(intersegment)가 명시되어야 함. 점프 명령어들은 near와 far로 분류될 수 있는데 near 점프는 주어진 세그먼트 내에서, far 점프는 세그먼트 간 이동 시 발생(INT / IRET은 far 점프)
* short jump : signed byte 변위를 지님. 이는 IP 레지스터의 현재 값에 더해짐
* near jump : short jump와 유사하나 signed word(little endian) 변위를 지님
* far jump : Far direct jump의 경우 segment selector와 effective address 모두를 명시하는 32비트 operand를 지님(short, near 점프와 달리 목적지의 주소를 명시적으로 지정)

다음과 같은 공격 기법들이 존재함
* Call table hooking : 프로그램의 제어를 얻기 위해 주소 테이블을 변경하는 것
* Direct Kernel Object Manipulation(DKOM) : 시스템 데이터 구조체 변경

<br/>

### Protect Mode
Real mode와 마찬가지로 segment memory 모델의 한 예이지만, 프로세서 홀로 물리 메모리 해상 과정을 수행하지 않고 OS 내의 특수한 테이블의 작용이 이 과정을 돕는다는 차이점이 있음. 이러한 특성이 OS에게는 추가적인 업무 수행의 부담을 지우지만, 특수한 테이블들이 메모리 보호, 페이징 요구 등의 추가적 기능들을 담당하면서 프로세서는 컴퓨팅 수행 그 자체에 집중할 수 있게 됨

#### <ins>Protected Mode 실행 환경</ins>
16비트 세그먼트 레지스터를 제외하고, real mode에 존재하던 레지스터들의 32비트 확장이 이루어짐. 또한 실행 환경 제어를 위한 특정 목적의 레지스터들이 추가됨. 이들엔 5개의 제어 레지스터(CR0 ~ CR4), global descriptor table 레지스터(GDTR), local descriptor table 레지스터(LDTR), interrupt descriptor table 레지스터(IDTR)가 포함됨. Real mode와는 달리, 세그먼트 레지스터들은 segment selector들을 담지만 물리 메모리의 64KB 세그먼트와 연관돼 있진 않음. 대신 이들은 선형 주소 공간 내의 세그먼트를 나타내는, segment descriptor라는 테이블 엔트리를 인덱싱하는 여러 필드로 구성된 바이너리 구조를 저장하고 있음(CS는 현재 실행 코드의 세그먼트의 segment descriptor를, DS ~ GS는 프로그램 데이터 세그먼트의 segment descriptor를, SS는 스택 세그먼트의 segment descriptor를 저장). 6개의 세그먼트 레지스터 중 유일하게 CS 레지스터는 명시적으로 세팅될 수 없으며, 대신 프로그램 제어와 관련된 명령어(JMP, CALL, INT, RET, IRET, SYSENTER, SYSEXIT 등)에 의해 암시적으로 세팅되어야 함

#### <ins>Protected Mode 세그먼테이션</ins>
IA-32 프로세서는 메모리 보호의 구현에 Segmentation과 Paging의 두 가지 기능을 이용(Protected mode에서 Segmentation이 필수적이라면 Paging은 선택적이며, 또한 Paging은 Segmentation 위에서 구현됨). segment selector는 16비트 사이즈이며 effective address는 32비트 사이즈. segment selector는 선형 주소 공간 내의 세그먼트를 나타내는 테이블 엔트리 참조. segment selector는 선형 주소 공간 내의 세그먼트에 대한 세부 사항들을 담고 있는 바이너리 구조를 인덱싱. 이 테이블은 descriptor table로 불리며 이것의 엔트리들은 segment descriptor라고 불림. segment descriptor(64비트)는 선형 주소 공간 내의 세그먼트에 대한 메타데이터(접근 권한, 크기, 32비트 base 선형 주소 등)를 포함. 이때 세그먼트의 32비트 base 선형 주소는 프로세서에 의해 디스크립터에서 추출되어 effective address가 제공하는 오프셋에 더해져 선형 주소를 나타내는데 사용됨(base 선형 주소와 offset 주소는 모두 32비트 값으로, 따라서 protected mode의 선형 주소 공간 크기는 4GB). descriptor table에는 두 종류가 있는데 GDT와 LDT가 바로 그것. GDT를 갖는 것은 필수 사항으로, IA-32상에서 구동되는 OS는 시작 시 반드시 하나의 GDT를 만들어야 함. 전체 시스템 작업 간에 공유될 수 있는 하나의 GDT가 존재할 것(GDT의 첫 엔트리는 항상 비어있으며 null segment descriptor로 불림. 이 descriptor를 참조하는 selector는 null selector로 불림). LDT의 사용은 선택적으로, 하나의 작업이나 관련된 작업들의 그룹에서 사용 가능. 한편 48비트 크기의 GDTR 레지스터는 GDT의 base 주소를 갖고 있으며 하위 16비트는 GDT의 사이즈를, 나머지 32비트는 GDT의 base 선형 주소를 나타냄. LGDT 명령어는 GDTR에 특정 값을 적재하며, SGDT 명령어는 GDTR의 값을 읽음. LGDT 명령어는 privileged 명령어로, OS에 의해서만 실행될 수 있음. segment selector는 3가지 필드로 이루어진 16비트 값. 상위 13비트는 GDT에 대한 인덱스로, 따라서 GDT는 최대 2<sup>13</sup>개의 segment descriptor를 가질 수 있음. 최하위 2비트는 selector의 request privilege level(RPL)을 나타내며 00, 01, 10, 11의 4가지 값을 갖는데, 0이 가장 높은 privilege level을, 3이 가장 낮은 level을 의미. 마지막으로 남은 Bit 2는 1일 때 LDT 내의 descriptor를, 0일 때 GDT 내의 descriptor를 명시. 64비트 구조체인 segment descriptor는 수많은 필드들로 이루어져 있는데 이 중 세 가지인 base address(세그먼트 32비트 base 선형 주소), type, S, DPL 필드에 집중. Descriptor privilege level(DPL)은 참조되는 세그먼트의 privilege level 정의. RPL과 마찬가지로 0 ~ 3의 값을 가짐(Privilege level은 4개의 특권 영역을 나타내는 3개의 동심원으로 표현되기도 함(Ring 0 ~ 3의 4개의 영역). DPL이 0인 세그먼트를 두고 Ring 0 내부에 존재한다고 말하기도 함. OS 커널은 Ring 0에서 실행되며 유저 애플리케이션은 Ring 3에서 실행). type 필드와 S 필드는 함께 사용되어 descriptor의 종류를 결정. S 플래그는 특히 segment descriptor의 두 부류를 결정하는데, S = 1일 때 코드와 데이터 segment descriptor를, S = 0일 때 시스템 segment descriptor를 의미. 코드와 데이터 segment descriptor는 일반 애플리케이션용 세그먼트를 참조. 시스템 segment descriptor는 현재 실행 작업의 privilege level(current privilege level(CPL))보다 높은 레벨의 세그먼트들로 점프하기 위해 사용됨. 한편 type 필드는 세그먼트의 타입(코드 또는 데이터), 접근, 자라나는 방향의 타입을 결정. Segment Limit 필드는 세그먼트의 크기 결정(G 플래그가 0이면 1byte ~ 1MB, 1이면 4KB ~ 4GB)

#### <ins>Protected Mode 페이징</ins>
페이징은 선택적이지만, 페이징이 사용되지 않는다면 선형 주소 공간이 직접적으로 물리 메모리와 연관되는 특성으로 인해 사용 가능한 메모리는 32비트 선형 주소에 따라 4GB로 한정. 페이징을 사용할 때 세그먼테이션으로 만들어진 선형 주소 공간은 고정 크기 저장소인 페이지(4KB, 2MB 또는 4MB)로 나뉨. 이러한 페이지들은 물리 메모리에 매핑되거나 디스크에 저장될 수 있음(만약 프로그램이 참조하는 바이트가 디스크에 저장되어 있는 페이지 내에 위치한다면, 프로세서는 페이지 폴트 예외를 발생시키고 이는 OS에게 물리 메모리로 해당 페이지를 로드할 것을 시그널). 페이지가 로드되는, 물리 메모리 내의 슬롯은 페이지 프레임으로 불림. 페이징 활성화 시 선형 주소는 세 개의 서브 필드를 갖는 구조를 갖게 됨. 최하위 12비트 필드만이 물리 메모리상의 바이트 오프셋을 나타내며, 나머지 두 개의 10비트 필드는 상대 위치를 나타내는 배열 지시자. bits 22 ~ 31의 세 번째 필드는 page directory로 불리는 배열의 엔트리를 명시. 이 엔트리는 page directory entry(PDE)로 불림. page directory의 첫 번째 바이트 물리 주소는 CR3 레지스터(Page Directory Base Register(PDBR))에 저장됨. 인덱스 필드의 10비트 크기로 인해 page directory는 최대 1,024개의 PDE를 저장 가능. 각 PDE는 page table로 불리는 두 번째 배열의 base 물리 주소를 담고 있음. bits 12 ~ 21의 두 번째 필드는 page table의 특정 엔트리(PTE) 명시. page table 역시 최대 1,024개의 PTE들을 담을 수 있음. 각 PTE들은 메모리 내 페이지의 첫 바이트 물리 메모리 주소를 담고 있음. bits 0 ~ 11의 첫 번째 필드는 PTE가 제공하는 물리 base 주소에 더해져 물리 메모리 내의 실제 바이트 주소를 나타냄(논리 주소의 10비트 필드들은 인덱스들을, 마지막 12비트 필드는 실제 바이트 오프셋을 담음)
* PDE, PTE에서 보여지는 20비트 base 주소는 real mode와 같이, 암시적인 0들을 가정함으로써(ex. 0x12345의 경우 실제로 0x12345000) 32비트 주소로 보여질 수 있음. 이러한 주소값은 (암시적 0을 제외하고) page frame number(PFN)로 불리기도 함(페이지 프레임은 특정한 위치이고 페이지는 측정의 단위. PFN은 페이지 프레임에 대한 번호)

#### <ins>주소 확장 페이징</ins>
PAE 페이징 활성화 시(CR4 레지스터의 PAE 플래그를 통해), 세그먼테이션 과정으로 생성된 선형 주소는 4개의 파트로 나뉨. PDE와 PTE 인덱스의 크기가 각각 한 비트 감소한 9비트가 되며 이로서 생겨난 2비트는 주소의 최상위에 위치하여 page directory pointer table(PDPT)의 4개의 엔트리(PDPTE) 중 하나를 가리키게 됨. PAE 페이징은 32비트 선형 주소를 52비트 물리 주소로 매핑(52개보다 적은 address line을 갖는 IA-32 프로세서의 경우 물리 주소 내 관련된 상위 비트들은 0으로 세팅됨). 모든 페이징 과정은 CR3 레지스터로부터 시작되는데 이때의 CR3 레지스터는 PDPT의 물리 base 주소(52비트가 아님) 명시

#### <ins>Table 세부 사항</ins>
PDE와 PTE는 모두 32비트 구조체(덕분에 page directory와 page table 모두 4KB 페이지 크기에 맞게 됨). 메모리 보호의 관점에선 U/S, W 플래그가 핵심적. U/S 플래그는 두 가지 페이지 privilege level(user, supervisor)을 정의. 만일 이 플래그가 클리어되면 해당 PDE / PTE 아래의 페이지들은 supervisor privilege를 갖게 됨. W 플래그의 경우 해당 페이지 또는 페이지 그룹(PDE의 경우)이 읽기 전용인지 혹은 쓰기가 가능한지를 나타냄. W 플래그 세팅 시 페이지 또는 페이지 그룹은 읽기와 쓰기 모두 가능. PAE 페이징 환경에서는 PDPTE, PDE, PTE 모두 64비트 크기를 가짐. 중요한 차이점은 base 물리 주소 필드의 크기가 프로세서가 갖는 address line의 개수에 따라 달라진다는 점

#### <ins>제어 레지스터 세부 사항</ins>
CR3 레지스터는 page directory의 첫 바이트 물리 주소를 가지며, 각 프로세스마다 고유의 CR3 복사값을 지님으로써 이 값을 커널이 갖는 스케줄링 context로 유지하고, 따라서 두 프로세스가 동일한 선형 주소를 갖더라도 서로 다른 물리 주소에 매핑되는 것이 가능해짐. 이것은 프로세스마다 고유의 page directory를 가진 결과. 이는 메모리 보호의 비교적 덜 분명하게 보이는 한 측면. 한편 CR0 레지스터 역시 주목할 만한데, CR0의 16번째 비트는 WP(Write Protection) 플래그. **WP 세팅 시 supervisor-level 코드가 읽기 전용 user-level 메모리 페이지 내에 쓰는 것을 허용하지 않음**. 이 메커니즘은 전통적으로 UNIX 시스템에서 사용되는 프로세스 생성의 copy-on-write 방식(forking과 같이)을 용이하게 하는 반면, 루트킷이 특정 시스템 데이터 구조체를 수정할 수 없다는 것을 의미하기에 위험할 수 있음
* CR4 : PSE(세팅 시 큰 페이지 사이즈 사용(2 또는 4MB)), PAE(세팅 시 PAE 활성화)
* CR3 : page directory base 주소(bits 12 ~ 31, PAE 환경에서는 PDPT의 물리 base 주소를 나타내는 27비트(5 ~ 31) 값을 가짐. 52비트 물리 주소를 나타내기 위해 암시적으로 뒤에 0이 붙게 됨)
* CR2 : 페이지 폴트를 발생시킨 선형 주소
* CR1 : 예약된 공간
* CR0 : PG(세팅 시 페이징 활성화), PE(세팅 시 protected mode 활성화. real mode로부터 전환 시 OS에 의해 세팅됨)

<br/>

### 메모리 보호 구현
분석 전, 어떠한 메모리 보호도 적용되지 않은 환경을 상상해볼 수 있을 것. 페이징이 비활성화된 두 개의 Ring 0 세그먼트가 전체 데이터(0x00000000 ~ 0xFFFFFFFF) 영역에 걸쳐 있어 모든 프로세스가 Ring 0상에서 실행되고 모든 메모리에 액세스 가능한 환경

#### <ins>세그먼테이션을 통한 보호</ins>
현재의 OS가 실제로 메모리를 관리하는 방식은 아님. IA-32 플랫폼상 세그먼트를 이용한 보호는 주소 해상 과정에서 한계 검사, 세그먼트 타입 검사, privilege-level 검사, 제한된 명령어 검사 등을 수행할 것. 이러한 검사들은 메모리 접근 사이클 시작 전에 수행될 것이며 위반이 발생할 때 일반 보호 예외가 프로세서에 의해 일어날 것. 게다가 주소 해상 과정과 함께 수행되기에 검사로 인한 성능 저하는 발생하지 않음

#### <ins>한계 검사</ins>
한계 검사는 segment descriptor의 20비트 limit 필드를 사용해 프로그램이 해당 위치에 존재하지 않은 메모리에 접근할 수 없게 함. 프로세서는 또한 GDTR의 size limit 필드를 사용해 segment selector가 GDT 밖의 엔트리에 접근하지 못하게 방지

#### <ins>타입 검사</ins>
타입 검사는 segment descriptor의 S 플래그와 type 필드를 사용하여, 프로그램이 적절하지 못한 방식으로 메모리 세그먼트 접근을 시도하는 것을 방지(ex. CS 레지스터는 코드 세그먼트에 대한 selector로만 로드될 수 있음. far call 또는 far jump는 다른 코드 세그먼트 segment descriptor 또는 call gate에만 접근 가능). 또한 만약에 프로그램이 첫 GDT 엔트리를 가리키는 selector로 CS 또는 SS 세그먼트 레지스터 로드를 시도한다면, 일반 보호 예외가 발생할 것

#### <ins>특권 검사</ins>
Privilege-level 검사는 4개의 privilege level에 기반. 특권 검사는 바깥 링에 위치하는 프로세스가 안쪽 링에 존재하는 세그먼트에 고의로 접근하는 것을 막는 것. 특권 검사 적용을 위해 CPL, RPL, DPL의 세 가지 지표 사용. Current Privilege Level<strong>(CPL)은 기본적으로, 실행 중인 프로세스의 CS, SS 레지스터 내에 저장된 selector들의 RPL</strong> 값. 프로그램의 CPL은 일반적으로 현재 코드 세그먼트의 privilege level. far jump나 far call의 실행에 따라 CPL은 바뀔 수 있음. 프로세서의 세그먼트 레지스터 중 하나에 segment descriptor와 연관된 segment selector가 로드될 때 Privilege-level 검사 수행. 이는 프로그램이 다른 코드 세그먼트 내의 데이터에 접근하려 하거나 또는 세그먼트 간 점프를 통해 프로그램 제어를 옮길 때 발생. 만약 프로세서가 Privilege-level 위반을 식별하면 일반 보호 예외 발생. 다른 데이터 세그먼트 내의 데이터에 접근할 때 데이터 세그먼트의 selector가 반드시 스택 세그먼트(SS) 레지스터나 데이터 세그먼트 레지스터(DS, ES, FS, GS)에 로드되어야 함. 프로그램이 다른 코드 세그먼트로 점프하는 제어 변경을 시도할 때 목적지 코드 세그먼트의 segment selector가 CS 레지스터에 반드시 로드되어야 함. CS 레지스터는 명시적으로 변경될 수 없으며 JMP, CALL, RET, INT, IRET, SYSENTER, SYSEXIT 등의 명령어에 의해 암시적으로만 변경 가능. 다른 세그먼트 내의 **데이터 접근 시 프로세서는 DPL이 RPL과 CPL 둘보다 크거나 같음**을 보장하기 위해 검사 진행(DPL은 프로그램이나 태스크가 세그먼트에 액세스를 허용해야 하는 가장 낮은 권한 레벨(숫자상으로 가장 높은)을 나타냄). 만약 조건이 성립한다면 프로세서는 해당 데이터 세그먼트의 segment selector를 데이터 세그먼트 레지스터에 로드할 것. 다른 세그먼트 내의 데이터에 **접근하려는 프로세스는 해당 데이터 세그먼트의 segment selector의 RPL 값을 제어**. SS 레지스터에 새로운 스택 세그먼트의 segment selector를 로드할 때, **스택 세그먼트의 DPL과 segment selector의 RPL은 모두 CPL**과 일치해야 함. nonconforming 코드 세그먼트(NCCS)는 보다 낮은 특권(보다 높은 CPL)에서 실행하는 프로그램에 의해 접근될 수 없는 코드 세그먼트로, **nonconforming 코드 세그먼트로 제어를 옮길 때 호출 루틴의 CPL은 목적지 세그먼트의 DPL**과 같아야 함. 목적지 세그먼트와 연관된 segment selector의 **RPL은 CPL보다 작거나 같아**야 함. **conforming 코드 세그먼트(CCS)로 제어를 옮길 때 호출 루틴의 CPL은 목적지 세그먼트의 DPL보다 크거나 같아**야 함. 이 경우 목적지 세그먼트의 segment selector의 **RPL 값은 검사되지 않음**
* 프로세서는 프로그램 제어가 다른 권한 레벨을 가진 코드 세그먼트로 전이될 때 CPL을 변경
* DPL은 세그먼트나 게이트가 액세스되고 있는 타입에 따라 다르게 해석됨
* 프로그램이나 태스크가 세그먼트에 접근하기 위한 충분한 권한을 갖더라도 만약 RPL이 그렇지 못하다면 접근은 거부됨. RPL이 **CPL보다 낮은 권한 레벨(숫자상 높은)인 경우 RPL은 CPL을 오버라이드**. RPL은 만약 프로그램 자체가 세그먼트에 대한 액세스 권한을 갖고 있지 않다면 프로그램 대신 privileged code가 세그먼트에 접근하지 않는 것을 보장
* 코드 세그먼트 직접 접근에 대한 일반적인 규칙은 세그먼트와 동일한 특권하에서만 그럴 수 있다는 것(NCCS의 사용). 보다 낮은 특권의 애플리케이션에서도 코드 세그먼트에 접근 가능하도록(호출자의 **특권을 상승시키지 않으면서) CCS** 등장. 따라서 코드 세그먼트에 대한 직접 접근은 결코 현재 특권을 변경하지 않음(스택 변경도 없음). **CPL의 변경을 위해 call gate** 등장

#### <ins>제한 명령어 검사</ins>
프로그램이 보다 낮은 CPL 값으로 제한된 명령어를 사용하려 하지 않음을 검증. LGDT, LLDT(LDTR 레지스터 로드), MOV(제어 레지스터로 값을 옮김), HLT(프로세서 중지), WRMSR(모델 특정 레지스터(MSR)에 쓰기 작업 수행) 등은 CPL이 0(가장 높은 privilege level)일 때만 실행되도록 제한된 명령어

#### <ins>Gate Descriptor</ins>
Gate descriptor들은 프로그램이 다른 privilege level로 코드 세그먼트에 접근할 수 있는 방법 제공. Gate descriptor는 system descriptor라는 점에서 특별(segment descriptor의 S 플래그가 클리어된 상태). 이러한 게이트들은 16비트 또는 32비트가 될 수 있음. 만약 코드 세그먼트 간 점프로 인해 스택 변경이 이루어져야 한다면, 새로운 스택에 푸시되어야 할 값이 16비트 푸시 또는 32비트 푸시로 축적될지를 결정(gate descriptor들의 종류는 segment descriptor의 type 필드로 식별됨)

##### Call-gate descriptor
(16비트의 경우 0100, 32비트의 경우 1100) GDT 내에 상주. 몇몇 차이점을 제외하고 segment descriptor의 구성과 매우 유사. 예를 들어 32비트 base 선형 주소 대신 16비트 segment selector와 32비트 오프셋 주소를 저장. Call-gate descriptor 내의 segment selector는 GDT 내의 코드 세그먼트 descriptor를 참조. 오프셋 주소는 해당 코드 세그먼트 descriptor에 더해져 목적지 코드 세그먼트 내의 루틴의 선형 주소를 나타냄. 이때, 원래 논리 주소의 effective address는 쓰이지 않음(segment selector만 call gate를 가리키는데 쓰임). 프로그램이 call gate를 이용하여 새로운 코드 세그먼트로 점프할 때 충족되어야 할 조건 첫째로, 프로그램의 CPL과 segment selector의 RPL은 반드시 call-gate descriptor의 DPL보다 작거나 같아야 하며, 둘째로 프로그램의 CPL은 목적지 코드 세그먼트의 DPL보다 크거나 같아야 함

##### Interrupt-gate descriptor와 Trap-gate descriptor
(Interrupt-gate 16비트의 경우 0100, 32비트의 경우 1110. Trap-gate 16비트의 경우 0111, 32비트의 경우 1111) Call-gate descriptor와 유사하게 동작. 대신 이들은 interrupt descriptor table(IDT) 내에 상주. Interrupt-gate와 Trap-gate descriptor 모두 segment selector와 effective address를 저장하고 있으며, segment selector는 GDT 내의 코드 segment descriptor를 명시하고 effective address는 코드 세그먼트의 base 주소에 더해져 선형 주소 공간 내의 interrupt / trap handling 루틴을 가리킴. interrupt-gate descriptor와 trap-gate descriptor의 유일한 차이점은 프로세서가 EFLAGS 레지스터 내의 IF 플래그를 조작하는 방식에 있음. interrupt-gate descriptor를 통해 interrupt handling routine에 접근할 때 프로세서는 IF를 클리어하지만, trap-gate의 경우엔 반대로 IF 변경을 요구하지 않음. handling 루틴을 불러일으키는 프로그램의 CPL은 interrupt 또는 trap gate의 DPL보다 작거나 같아야 하는데 이러한 조건은 소프트웨어에 의해 handling 루틴이 호출되는 경우(INT 등)에만 유지됨. 또한 call gate와 마찬가지로 handling 루틴의 코드 세그먼트를 가리키는 segment descriptor의 DPL은 프로그램의 CPL보다 작거나 같아야 함

#### <ins>Protected-Mode Interrupt Table</ins>
real mode와 달리 protected mode에선 IDT가 IVT를 대신함. IDT는 64비트 gate descriptor들의 배열을 저장하고 있음(interrupt-gate, trap-gate, task-gate descriptors). IVT와는 달리 IDT는 선형 주소 공간 내 어디에나 상주 가능. IDT 32비트 base 주소가 48비트 IDTR 레지스터의 bits 16 ~ 47에 위치하며, IDT의 크기 제한은 bits 0 ~ 15에 위치. LIDT 명령은 IDTR 값의 세팅에 사용될 수 있고 SIDT 명령은 IDTR 레지스터 값을 읽는데 사용됨. 크기 제한은 IDT의 base 주소에서 테이블의 마지막 엔트리까지의 바이트 오프셋으로, N개의 엔트리를 갖는 IDT는 (8(N – 1))의 제한을 가질 것. 만약 크기 제한을 넘는 벡터가 참조될 때 프로세서는 일반 보호 예외를 발생시킴. protected mode에서 벡터 0 ~ 31은 머신 특정적인 예외와 인터럽트를 위해 IA-32 프로세서에 의해 예약되어 있음. 나머지는 유저 정의 인터럽트에 사용 가능

#### <ins>페이징을 통한 보호</ins>
IA-32 프로세서에서 제공하는 페이징 기능 역시 메모리 보호 구현에 사용 가능. 세그먼테이션을 통한 메모리 보호와 마찬가지로 페이지 층위의 검사도 메모리 사이클 초기화 이전에 수행. 페이지 층위 검사는 주소 해상 과정과 동시에 이루어지므로 성능 저하는 발생하지 않음. 메모리 층위 검사 위반 발생 시 페이지 폴트 예외가 프로세서에 의해 일어남. protected mode 역시 segmented memory 모델의 한 예이기에 IA-32 프로세서에서 세그먼테이션은 필수적. 반면 페이징은 선택적. 페이징이 활성화된 상태라 하더라도 CR0 레지스터의 WP 플래그를 클리어하고 각 PDE와 PTE의 R/W, U/S 플래그를 세팅함으로써 페이징을 통한 보호를 비활성화 가능(이는 모든 메모리 페이지를 쓰기 가능 상태로 만들고 유저 privilege level을 갖게 하며, supervisor level 코드가 읽기 전용 user level 페이지에 쓰는 것을 허용). 만약 세그먼테이션과 페이징이 모두 메모리 보호에 사용된다면 **세그먼트 층위의 검사가 먼저 수행되고, 이어서 페이지 검사**가 수행됨. 세그먼트 층위 위반은 일반 보호 예외를, 페이지 층위 위반은 페이지 폴트 예외를 발생시킴. 게다가 **세그먼트 층위 보호 설정은 페이지 층위 보호 설정에 의해 오버라이드되지 않음**(ex. 코드 세그먼트 내의 페이지와 연관된 페이지 테이블 내의 R/W 비트 세팅은 페이지를 쓰기 가능 상태로 만들지 못함). 페이징 활성화 상태라면 프로세서가 수행 가능한 두 가지 타입의 검사가 수행되는데 User / Supervisor 모드 검사와 Page-type 검사가 바로 그것. 선형 주소에 대한 모든 접근은 supervisor-mode 접근이거나 user-mode 접근. CPL이 3보다 작을 때 이루어지는 모든 접근은 supervisor-mode 접근이며, CPL = 3일 때 접근은 일반적으로 user-mode 접근. 하지만 선형 주소로 시스템 데이터 구조체를 암시적으로 접근하는 몇몇 동작들은 CPL과 상관없이 supervisor-mode 접근을 수행. segment descriptor를 로드하기 위한 GDT나 LDT 접근, 인터럽트나 예외 발생 시 IDT 접근, 태스크 전환이나 CPL 변경의 일환으로 수행되는 태스크 상태 세그먼트(TSS) 접근이 그 예. 다음은 페이징이 접근 권한을 결정하는데 있어서의 세부 사항들임

##### supervisor-mode 접근
* 데이터 읽기 : 데이터는 변환을 통해 어느 선형 주소에서든 읽혀질 수 있음
* 데이터 쓰기 : 데이터는 변환을 통해 어느 선형 주소에서든 쓰일 수 있으나 CR0 레지스터의 WP 플래그가 1일 땐, 변환을 제어하는 모든 페이징 구조체 엔트리의 R/W 플래그가 1인 경우에만 가능
* 명령어 패치 : 명령어는 변환을 통해 어느 선형 주소에서든 패치될 수 있으나 CR4 레지스터의 SMEP 플래그가 1일 땐, 변환을 제어하는 페이징 구조체 엔트리 중 최소 하나의 U/S 플래그가 0인 경우에만 가능

##### user-mode 접근
* 데이터 읽기 : 데이터는 변환을 제어하는 모든 페이징 구조체 엔트리의 U/S 플래그가 1인 경우, 변환을 통해 어느 선형 주소에서든 읽혀질 수 있음
* 데이터 쓰기 : 데이터는 변환을 제어하는 모든 페이징 구조체 엔트리의 R/W 플래그와 U/S 플래그가 모두 1인 경우, 변환을 통해 어느 선형 주소에서든 읽혀질 수 있음
* 명령어 패치 : 명령어는 변환을 제어하는 모든 페이징 구조체 엔트리의 U/S 플래그가 1인 경우, 변환을 통해 어느 선형 주소에서든 패치될 수 있음

페이징을 통한 보호와 관련된 기타 사항들
* 프로세서는 존재하는 경우, 메모리상 페이징 구조체 대신 TLB와 페이징 구조체 캐시에 기반을 두어 접근 권한을 결정
* 세그먼테이션은 필수 사항이지만, 세그먼트 층위 보호의 영향을 최소화하고 페이지 층위 기능들에 주로 의존하는 것도 가능. 특히 5개의 엔트리로 이루어진 GDT를 갖는 flat 세그먼테이션 모델의 경우 하나의 null descriptor와 각각 두 개의 코드와 데이터 descriptor 셋을 통해 구현 가능. 하나의 code와 data descriptor 셋은 0의 DPL 값을, 다른 셋의 경우 3의 값을 갖게 하고 모든 descriptor에 대해 0x00000000 주소에서 시작해서 전체 선형 주소 공간에 걸쳐 있도록 세팅하여, 모두가 동일한 공간을 공유하게 하고 사실상 세그먼테이션이 일어나지 않도록 만듦. 페이징은 세그먼테이션에 비해 보다 나은 granularity level의 기능들을 제공하며 완전히 선택적이라는 장점이 있음

결국 이 모든 내용은 운영 체제가 만들고 특수한 데이터 구조체로 채우는 몇몇 인덱스 테이블들로 귀결됨. 이러한 데이터 구조체들은 메모리의 레이아웃과 메모리 접근에 대해 프로세서가 검사할 때 이용하는 규칙들을 정의. 만약 규칙이 위반된다면 프로세서는 예외를 던지고 OS에 의해 정의된 이벤트 handling 루틴을 호출할 것

<br/><br/>

## System Briefing
Windows와 Intel 사이의 매핑은 일대일 대응이 아님. Windows는 HAL을 통해 칩셋 의존성을 격리(Windows NT는 처음에 MIPS와 Intel 하드웨어 플랫폼 둘 다에서 구동되도록 구현됨)

<br/>

### Windows상의 물리 메모리
msinfo32 커맨드를 통해 실제 물리 메모리의 전체 공간을 사용할 수 있는 것은 아니라는 것을 알 수 있음. 윈도우가 디바이스 메모리를 위해 공간을 예약해두기 때문

#### <ins>윈도우가 물리 주소 확장을 사용하는 방법</ins>
윈도우가 접근 가능한 물리 메모리의 크기는 윈도우의 크기, 내부 하드웨어 플랫폼, 윈도우가 구성된 방식에 따라 달라짐(IA-32상의 Windows 7은 최대 4GB의 메모리에만 접근 가능하기에 PAE 활성화는 어떤 추가 메모리도 벌어주지 못함(64비트 OS의 경우 기본적으로 주소값에 32비트 이상 사용 가능하기에 PAE를 사용할 이유가 없음). 그럼에도 PAE를 사용하는 이유는 DEP 같은 추가적인 기능 지원을 위한 것)

#### <ins>페이지, 페이지 프레임, PFN</ins>
페이지는 선형 주소 공간 내의 연속적인 영역으로, 한 페이지와 관련된 어떠한 구체적이고 물리적인 위치도 존재하지 않음. 반면 페이지 프레임은 페이지가 RAM에 상주할 때 페이지가 저장되는 물리 메모리상 구체적인 위치. 이 위치의 물리 주소는 PFN으로 나타내어질 수 있음(페이지가 4KB이고 PAE가 비활성화된 환경의 경우 PFN은 20비트 값. 이 값은 최하위에 12비트 크기 0들을 가정함으로써 32비트 물리 주소 표현. 이 말은 페이지들은 항상 4KB 경계에 정렬된다는 의미)

#### <ins>윈도우상의 세그먼테이션과 페이징</ins>
OS와 유저 애플리케이션 간 경계는 하드웨어 기반 메커니즘에 크게 의존. 윈도우는 세그먼테이션보다 페이징에 더 의존하는 경향이 있음. 세그먼트 privilege 파라미터들을 통한 4개의 링 모델은 supervisor level(커널모드) 또는 user level(유저모드) 둘 중 하나에서 코드가 구동되는 두 개의 링 모델로 단순화됨. 이 구분은 시스템의 PDE와 PTE들 내의 U/S 비트에 기반을 둠

##### 세그먼테이션
윈도우는 GDT에서 전체 선형 주소 공간(0x00000000 ~ 0xFFFFFFFF)을 코드와 데이터 세그먼트로 정의하는 4개의 descriptor를 갖는데, 이들은 또한 같은 영역에서 링 0와 링 3의 세그먼트를 동시에 규정. 본질적으로 세그먼테이션은 없게 됨. 이는 페이징을 통해 보안이 구현되는 것을 허용함(윈도우가 Intel 하드웨어가 제공하는 모든 기능을 사용하는 것은 아님)

##### 페이징
윈도우에서 각 프로세스는 자신만의 페이지 디렉터리를 가짐. 연관된 CR3값은 KPROCESS 구조체 내의 DirectoryTableBase 필드 내에 저장됨(!process 출력의 DirBase). 선형 주소 0x7FFFFFFF에서 0x80000000으로 넘어갈 때 PDE 엔트리의 U/S 플래그에서 전환이 일어나는데, 이것이 바로 윈도우가 두 개의 링의 메모리 보호를 구현하는 방식

##### 선형 – 물리 주소 변환
윈도우 메모리 관리자는 0xC0000000에서 시작하는 선형 주소 공간 내에 페이지 테이블들을 로드(한편, 페이지 디렉터리들의 경우 x86 시스템에선 0xC0300000, PAE 활성화 시 0xC0600000)
> PTE 선형 주소 = 페이지 테이블 시작 주소 + 페이지 디렉터리 인덱스 * 페이지 테이블당 크기(바이트) + 페이지 테이블 인덱스 * PTE 크기(바이트)

#### <ins>유저 영역과 커널 영역</ins>
윈도우는 물리 메모리를 시뮬레이션하기 위해 디스크 공간 사용. 윈도우에서 각 프로세스는 고유의 CR3 레지스터 값을 가지며 고유의 가상 주소 공간을 가짐

##### 4GB 튜닝(4GT)
BCD의 increaseuserva 옵션을 활성화하여 IMAGE_FILE_LARGE_ADDRESS_AWARE 플래그가 세팅된 애플리케이션에 한해 3GB의 유저 메모리 허용

##### To Each His Own
같은 커널 주소 공간의 공유는 각 프로그램의 supervisor-level PDE들을 같은 시스템 페이지 테이블 셋에 매핑함으로써 구현

##### Jumping the Fence
커널의 주소 공간은 보호받고 있어서 애플리케이션 스레드는 커널 영역에 접근하기 위해 시스템 콜 게이트를 통과해야 하지만, 스레드는 SYSENTER 명령어를 통해서 유저 영역과 커널 영역을 넘나들 수 있음. 이는 유저 애플리케이션이 자의적으로 커널 영역을 넘나들고 privileged 코드를 충동적으로 실행하는 것을 방지

##### 커널 영역 동적 할당
Windows Vista와 Windows Server 2008 이후 커널은 변화하는 요구의 수용을 위해 내부 구조체들을 동적으로 할당하고 재배치할 수 있게 됨

##### 주소 윈도잉 확장(AWE)
AWE는 32비트 유저 애플리케이션이 64GB까지의 물리 메모리에 접근하는 것을 허용. 이 기능은 winbase.h에 선언된 API를 기반으로 하고 있으며 이는 이 API를 사용한 프로그램이 선형 주소 공간에 의해 주어진 제한보다 큰 크기의 물리 메모리 범위에 접근하는 것을 허용하는 방식. AWE API 루틴으로는 VirtualAlloc(), VirtualAllocEx(), AllocateUserPhysicalPages(), MapUserPhysicalPages(), MapUserPhysicalPagesScatter(), FreeUserPhysicalPages()가 있음. AWE는 애플리케이션의 선형 주소 공간 내에 고정 크기 영역(window)들을 할당하고 물리 메모리 내의 더욱 큰 고정 크기 window들에 매핑한다는 점에서 주소 윈도잉 확장으로 불림. AWE를 통해 할당된 메모리는 페이징되지 않음(non-paged). AWE는 엄밀히 Microsoft의 기술이지만(또한 PAE 없이도 사용될 수 있음), 만약 애플리케이션이 4GB 경계 이상의 물리 메모리에 접근하는데 AWE API를 사용한다면 PAE의 활성화는 필요할 것. 또한 AWE 루틴을 호출하는 애플리케이션을 실행하는 유저는 ‘Lock Pages in Memory’ 특권 필요

4GT는 선형 주소 공간을 재분할해 유저 애플리케이션이 더욱 큰 영역을 가질 수 있게 하나, 만약 애플리케이션이 3GB보다 큰 영역을 요구할 경우 AWE를 사용할 수 있음. AWE를 사용하는 애플리케이션에서 4GB 이상의 물리 메모리를 요구하면 PAE의 활성화가 필요할 수 있음

#### <ins>유저모드와 커널모드</ins>
유저 영역은 유저모드에서 실행되는 코드를 포함하고 있음. 유저모드에서 실행되는 코드는 커널 영역 내의 어떤 것도 접근할 수 없으며 하드웨어와 직접 통신하는 것도, privileged 명령어를 실행하는 것도 불가능. 커널 영역 내의 코드는 커널모드에서 실행되며 유저모드의 코드에서 할 수 없는 모든 것이 가능

##### 커널모드 컴포넌트
HAL의 경우 interrupt controller들을 처리하기도 함(cf. halacpi.dll, halmacpi.dll). 파일의 PE 헤더를 조사하며 각 커널 모듈 간 import 관계 확인 가능

##### 유저모드 컴포넌트
환경 서브시스템은 유저모드에서 동작하는 바이너리 셋으로 특정 환경 및 API를 사용하도록 작성된 애플리케이션이 작동할 수 있도록 함(다른 운영 체제에서 동작하는 프로그램이 눈에 띄는 변화 없이 서브시스템 위에서 실행될 수 있게 함). Windows 서브시스템은 Csrss.exe, Win32k.sys, 서브시스템 API를 구현하는 유저모드 DLL로 이루어짐. Windows 서브시스템이 유저 애플리케이션에 노출하는 인터페이스(Windows API)는 Win32 API와 많이 닮아 있음

#### <ins>기타 메모리 보호 기능들</ins>
유저모드에서 애플리케이션을 실행하는 것 외에도 윈도우가 프로그램의 주소 공간의 무결성을 보장하기 위해 구현하는 기능들이 존재

##### 데이터 실행 방지(DEP)
DEP는 Software-enforced와 Hardware-enforced 유형으로 나뉨. Software-enforced DEP는 Microsoft가 포함시킨, 보다 약한 구현. Hardware-enforced DEP는 IA32-EFER 머신 특정 레지스터의 NXE 플래그의 세팅을 요구. 비트가 세팅되어 있고 그리고 PAE 활성화 상태라면 PDE와 PTE의 64번째 비트가 XD라는 특수한 플래그로 변경됨. 만약 IA32_EFER = 1이고 XD = 1이라면 현재 참조되는 페이지로부터 꺼내지는 명령어들은 허용되지 않음. DEP는 BCD의 nx 정책 설정 세팅을 통해 부팅 시점에 구성됨. 특정 프로세스의 DEP와 연관된 엔트리들은 관련 KPROCESS 구조체 내의 KEXECUTE_OPTIONS 구조체에 위치해 있음(DEP 활성화 시 ExecuteDisable 필드가 1로 세팅되며 반대로 비활성화 시 ExecuteEnable 필드가 1로 세팅됨. DEP 구성이 프로세스에 의해 동적으로 변경될 수 없는 구성인 경우 Permanent 필드가 1로 세팅됨. Process Explorer의 DEP 프로세스 상태에 의하면 ‘DEP(permanent)’는 모듈이 시스템 바이너리라 DEP가 활성화된 경우, ‘DEP’는 DEP가 현재 정책 또는 opted in으로 인해 활성된 경우, ‘Empty’는 현재 정책 또는 opted out으로 인해 비활성화된 경우)

##### 주소 공간 레이아웃 랜덤화(ASLR)
ASLR과 DEP는 사용될 때 가장 효과적

##### /GS 컴파일러 옵션
/GS 컴파일러 옵션은 software-enforced DEP로 여겨질 수 있음. security cookie라는 특별한 값을 스택 프레임에 추가해 버퍼 오버플로를 방지. 만약 함수 반환 전의 스택 검사 시 security cookie가 변경되었다면 프로그램 제어는 sechook.c에 정의된 특수한 루틴들로 옮겨갈 것. 대부분의 구현은 루틴의 프롤로그와 에필로그에서 이루어짐. 스택 프레임 셋업 후 할당되는 추가 공간들은 인자 및 지역 변수들을 스택상의 함수 버퍼 아래(below, 낮은 주소값)로 복사될 수 있게 함으로써 버퍼 오버플로 발생 시 조작으로부터 그것들을 보호할 수 있음. 이러한 행위를 variable reordering이라 하며 컴파일러는 이러한 변수 복사들을 tv(temporary variables) 접두어로 나타냄. 공간들이 할당된 후 쿠키값은 스택 프레임 포인터(EBP) 값과 XOR 연산을 거치고 스택의 __$ArrayPad$로 이름 붙여진 지역 변수에 저장됨. __$ArrayPad$ 값은 프레임 내에서 지역 변수로 존재하는 버퍼보다 높은 주소에 위치하므로 오버플로 시 덮어써지게 될 것. 루틴 종료 시, 쿠키값을 대상으로 __security_check_cookie 루틴이 호출되어 쿠키값이 오버플로에 의해 변경되진 않았는지 점검 수행. 만약 변경되었다면 __security_error_handler()를 단계적으로 호출하는, report_failure() 함수 호출. 일반적으로 Visual Studio 컴파일러는 스택 프레임 보호가 필요한 루틴을 결정하는데 heuristic한 알고리즘을 사용하나, strict_gs_check pragma를 사용해 컴파일러가 로컬 변수를 조작하는 모든 루틴에 GS 쿠키를 삽입하도록 할 수 있음

##### /SAFESEH 링커 옵션
만약 /SAFESEH 옵션이 IA-32 시스템에서 명시된다면 링커는 바이너리의 헤더에 모듈의 유효한 예외 핸들러의 목록을 갖는 특수한 테이블을 삽입. 실행 중에 예외 발생 시, 예외 처리에 책임이 있는 ntdll.dll 내의 코드는 현재 스택에 위치한 예외 핸들러 레코드가 테이블에 명시된 핸들러 중 하나인지를 확인할 것

#### <ins>Native API</ins>
전통적으로 UNIX와 같은 OS는 항상 명확하게 정의된 well-documented system call 셋을 포함해왔음. 반면 윈도우의 경우 Native API라는 system call 인터페이스를 가지며 Windows API 뒤에 그것들을 감추어왔음. 이러한 방식 덕분에 만약 system call 인터페이스 업데이트를 포함하는 시스템 패치가 이루어지더라도 개발자는 이에서 소외되지 않을 수 있는데, 그들의 코드는 Windows API에 의존하고 있기 때문

##### The IVT Grows Up
MS-DOS 같은 real-mode OS에서 IVT는 가장 주된 시스템 레벨 구조체였으며 커널로 통하는 정식 입구였음. 모든 DOS system call은 소프트웨어로부터 발생한 인터럽트에 의해 호출될 수 있었음(함수 코드가 AH 레지스터에 놓인 상황에서의 INT 0x21 명령어를 통해). 그러나 IVT가 IDT로 재탄생하며, 중심적인 커널 입구 구조체로서의 기능을 잃게 되었음

##### IDT 세부 사항
윈도우 시작 시, 구동되는 프로세서의 종류를 검사하고 system call 호출들을 그에 맞게 조정(펜티엄 II 이전 프로세서의 경우 INT 0x2E 명령어가 system call에 사용되고, IA-32 프로세서의 경우 SYSENTER 명령어를 사용하여 IDT를 커널 영역으로의 점프의 의무에서 해방). 가장 현대의 윈도우 구성은 하드웨어 발생 신호를 처리하고 프로세서 예외에 응답하는 데만 IDT를 사용. Intel의 스펙에 따르면 IDT는 최대 256개의 8바이트 descriptor를 유지할 수 있음(IDT의 베이스 주소와 크기는 IDTR, IDTL 레지스터를 덤프하여 알 수 있음). !idt 명령 사용 시, 콘솔로 스트림된 254개의 엔트리 중 1/4 이하가 의미 있는 루틴을 참조. 대다수의 엔트리가 KiUnexpectedInterrupt 루틴을 참조하고 있음. 이러한 KiUnexpectedInterrupt 루틴들은 메모리상에서 연속적으로 정렬되어 있으며 그들은 모두 KiEndUnexpectedRange 함수를 호출하는 것으로 끝이 남. 또한 이전의 프로세서들을 위한 커널 영역 점프 기능은 여전히 IDT 엔트리 0x2E에 상주하고 있음(KiSystemService. 내부적으로는 KiFastCallEntry의 중간 부분을 호출하고 있음). 이는 시스템 서비스 디스패처로, Native API 루틴의 주소를 지정하고 이를 호출하는데 그것에 상속된(유저모드로부터) 정보를 사용

##### 인터럽트 기반 system call
INT 0x2E가 system call 호출에 사용될 때 system service number(또는 dispatch ID)가 EAX 레지스터에 세팅됨(real-mode를 연상). 또한 호출자의 인자 목록의 주소가 EDX 레지스터에 세팅되며 최종적으로 인터럽트가 이루어짐

##### SYSENTER 명령어
SYSENTER가 호출되기 전 3개의 64비트 machine-specific register(MSR)들이 입력되어 프로세서가 점프해야 할 목적지와 커널모드 스택의 위치를 알도록 해야 함(유저모드 스택의 정보를 복사해야 하는 경우에). 이러한 MSR들은 RDMSR과 WRMSR 명령어로 조작될 수 있음
* IA32_SYSENTER_CS(주소값 0x174) : 커널모드 코드와 스택 segment selector 명시
* IA32_SYSENTER_ESP(주소값 0x175) : 커널모드 스택 포인터의 위치를 명시
* IA32_SYSENTER_EIP(주소값 0x176) : 커널모드 코드의 EP를 명시
* IA32_SYSENTER_CS와 IA32_SYSENTER_EIP 레지스터 덤프 시 그들이 KiFastCallEntry 루틴을 명시하고 있음을 알 수 있음. IA32_SYSENTER_CS에 저장된 segment selector는 전체 주소 영역에 걸쳐 있는 Ring 0 코드 세그먼트와 연관되어 있음. 그리고 IA32_SYSENTER_EIP는 KiFastCallEntry 루틴의 완전한 32비트 선형 주소를 오프셋으로서 담고 있음
* INT 0x2E의 경우와 마찬가지로, SYSENTER 명령어 실행 전에 system service number가 EAX 레지스터에 위치해 있어야 하며 EDX 레지스터는 호출자의 인자 목록을 가리키는 것이 됨

##### 시스템 서비스 디스패치 테이블
system service number는 32비트 값으로, 처음 12비트는 어떤 시스템 서비스가 최종적으로 호출될 것인지를, Bits 12, 13은 4개의 가능한 service descriptor table 중 하나를 명시. 비록 4개의 descriptor table이 가능하지만 커널 영역 내에서 가시적인 심볼을 갖는 테이블은 KeServiceDescriptorTable(Ntoskrnl.exe에 의해 익스포트됨)과 KeServiceDescriptorTableShadow(익스큐티브 경계 내에서만 보여짐)뿐. Bits 12, 13이 00이라면(system service number는 0x0000 ~ 0x0FFF에 걸쳐 있음) KeServiceDescriptorTable이 사용되며, 만약 bits 12, 13이 01이라면(system service number는 0x1000 ~ 0x1FFF에 걸쳐 있음) KeServiceDescriptorTableShadow가 사용됨(0x2000 ~ 0x2FFF, 0x3000 ~ 0x3FFF는 service descriptor table을 위해 할당된 것으로 보이지 않음). 두 service descriptor table은 시스템 서비스 테이블(SST)이라는 하위 구조체들을 포함하고 있음. SST는 다음의 4개의 필드로 이루어진 구조체

* serviceTable : 선형 주소들로 이루어진 배열의 첫 요소의 포인터를 담고 있는데 선형 주소들은 커널 영역 내 루틴의 EP들이며, 이러한 선형 주소들을 담고 있는 배열을 시스템 서비스 디스패치 테이블(SSDT)이라 부름. SSDT는 정신적으로 real-mode의 IVT와 같음
* field2 : 윈도우 free build에서는 사용되지 않는 필드
* nEntries : SSDT 배열 내의 엔트리 수를 명시
* argumentTable : 바이트들로 이루어진 배열의 첫 요소의 포인터를 담고 있는데 배열 내의 각 바이트는 연관된 SSDT 루틴 호출 시 함수 인자를 위해 할당하는 공간의 바이트 크기를 나타냄. 이러한 배열은 시스템 서비스 파라미터 테이블(SSPT)로 불림
<br/>

<img src="https://github.com/mollose/Security/assets/57161613/c1e28417-3ab0-4662-88a1-55cf1759f406" width="800"><br/>

* KeServiceDescriptorTable의 첫 16바이트는 윈도우 Native API를 위한 SSDT를 나타내는 SST(Windows 7의 경우 이는 401개의 루틴들로 구성(nEntries = 0x191)). KeServiceDescriptorTableShadow의 첫 32바이트는 두 개의 SST를 포함하고 있음. 첫 번째 SST는 KeServiceDescriptorTable의 SST의 복사본, 두 번째 SST는 Win32k.sys 커널모드 드라이버에 의해 구현된 USER와 GDI 루틴들을 위한 SSDT를 나타냄
