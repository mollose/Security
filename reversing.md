# 리버싱 핵심 원리

## 목차

<br/><br/>

* Win32 응용 프로그램에서는 API 함수의 파라미터를 스택을 이용하여 전달하는데, VC++ 사용 시 기본 문자열은 유니코드가 되고, 문자열 API 함수들도 유니코드 계열 함수로 변경됨
* Optional Header의 DLL Characteristics 멤버에서 0x0040은 Dynamic Base를 의미(Windows Vista부터 적용된 ASLR 기능 때문). 0x0000으로 패치함으로써 ASLR 기능 해제 가능
* 라이브러리와 헤더의 차이점 : 라이브러리는 기계어로 번역된 바이너리이며, 헤더 파일은 컴파일러가 컴파일 전 알아볼 수 있도록 프로그래밍 언어의 문법에 맞게 작성된 선언들의 집합. 라이브러리를 사용하기 위해서는 해당 라이브러리의 헤더 파일이 있어야 하는데, 컴파일러가 이러한 헤더 파일을 가지고 심볼 네임을 만들어 오브젝트 파일에 넣어주면, 링커가 해당 심볼 네임을 가지고 라이브러리를 뒤져 링크를 하게 됨

<br/><br/>

## 레지스터
극히 소량의 데이터나 처리 중인 중간 결과를 일시적으로 기억해두는 고속의 전용 영역. 아래는 IA-32 레지스터 중 Basic program execution register의 종류들

<br/>

### 32bit 범용 레지스터
* EAX : 주로 산술 연산에 사용. 함수의 리턴값이 저장됨
* EBX : 특정 주소를 저장하는데 사용. 간접 주소 지정자
* ECX : 반복적으로 실행되는 특정 명령에 사용. 반복 카운터
* EDX : 주로 EAX 보조로 사용되며, 일반적인 자료 저장, 입출력에 사용
* 각 레지스터들은 16bit 하위 호환을 위해 몇 개의 구획으로 나뉨(ex. AX(EAX의 하위 16bit), AH(AX의 상위 8bit), AL(AX의 하위 8bit))

<br/>

### 세그먼트 레지스터(16bits) 
IA-32 보호 모드에서 세그먼트란, 메모리를 조각내어 각 조각마다 시작 주소, 범위, 접근 권한 등을 부여해 메모리를 보호하는 기법을 말하며, 페이징 기법과 함께 가상 메모리를 실제 물리 메모리로 변경할 때 사용되기도 함. 세그먼트 메모리는 Segment Descriptor Table(SDT)에 기술되어 있는데, 세그먼트 레지스터는 SDT의 index를 저장
* CS(Code Segment) : 실행될 기계 명령어가 저장된 메모리 주소 지정
* DS(Data Segment) : 프로그램에서 정의된 데이터, 상수, 작업영역의 주소 지정
* SS(Stack Segment) : 프로그램이 임시로 저장할 필요가 있거나 함수 호출에 관련된 정보 저장
* ES, FS, GS(Extra Data Segment)
* 세그먼트 레지스터가 가리키는 세그먼트 디스크립터와 가상 메모리가 조합되어 선형 주소(Linear Address)가 되며, 페이징 기법에 의해 변환 과정을 거치거나, 페이징을 사용하지 않을 시엔, 그대로 물리 주소로 바뀌게 됨

<br/>

### 포인터 레지스터(32bits)
* EIP : 다음 실행될 명령어의 주소가 저장됨(프로그램 카운터. 범용 레지스터들과 달리 직접 값을 변경할 수 없기 때문에, 다른 명령어를 사용해 간접적으로 변경시켜야 함(특정 명령어(JMP, Jcc, CALL, RET 등) 사용 혹은 인터럽트, 예외의 발생)). EIP에 저장된 주소의 명령어를 하나 처리하고, 자동으로 명령어 길이만큼 EIP 길이 증가
* EBP : SS 레지스터와 함께 사용. 스택 프레임의 기준점 주소
* ESP : SS 레지스터와 함께 사용. 스택의 가장 끝 주소 가리킴
* ESP는 스택 메모리에서의 현재 위치를 가리키며, EBP는 함수 호출 시 그 순간의 ESP를 저장하고 있다가 함수가 리턴하기 직전에 ESP에 값을 되돌려줌으로써 스택이 깨지지 않도록 함(Stack Frame 기법)

<br/>

### 인덱스 레지스터(32bits) 
특정 명령어들(LODS, STOS, REP MOVS 등)과 함께 주로 메모리 복사에 사용
* EDI : 목적지(사본 주소)에 대한 값 저장
* ESI : 출발지(원본 주소)에 대한 값 저장

<br/>

### 플래그 레지스터(32bits)
EFLAGS(FLAGS 레지스터의 32bit 확장형으로, 연산 결과 및 시스템 상태와 관련된 플래그값 저장). 일부 비트는 시스템에서 직접 세팅하고 일부 비트는 프로그램에서 사용된 명령의 수행 결과에 따라 세팅됨
* ZF(Zero Flag) : 연산 후 결과값이 0일 시 1로 세팅
* OF(Overflow Flag) : 부호 있는 수의 오버플로 발생, 혹은 MSB 변경 시 1로 세팅
* CF(Carry Flag) : 부호 없는 수의 오버플로 발생 시 1로 세팅

<br/><br/>

## 어셈블리 기본 명령어
* PUSH, POP : 스택에 값을 넣거나 스택에 있는 값을 사용
* MOV : 값을 넣는 역할(ex. MOV eax, 1(eax에 1을 집어넣음))
* MOVZX(Move with Zero-Extend) : 소스 오퍼랜드를 도착점으로 복사 후, 그 값을 16bit 또는 32bit 확장
* LEA : 값을 가져오는 MOV와 달리, 주소를 가져옴(ex. lea eax, dword ptr ds:[esi](esi의 주소값을 eax로 가져옴. ≠ mov eax, dword ptr ds:[esi](esi가 가리키는 값을 eax로 가져옴)))
* ADD, SUB : src에서 dest로 값을 더하거나 빼는 명령
* INT : 인터럽트(예기치 않은 일이 발생 시 작동 중지 예방)을 일으킴
* CALL : 함수를 호출. CALL로 호출된 코드 안에서 RET을 만났을 때 해당 호출 번지의 다음 번지로 되돌아옴(명령어 실행 시 호출 번지의 다음 주소값을 스택에 PUSH)
* INC, DEC : i++, i--와 대응
* AND, OR, XOR : 비트 연산자. dest와 src를 동일한 오퍼랜드로 처리 가능
* NOP : 아무 것도 하지 말라는 명령어(CPU 클럭만 소모)
* TEST : 논리 비교. 두 operand 중에 하나가 0이면 ZF = 1로 세팅(‘AND’ 연산과 동일)
* PUSHAD : EAX ~ EDI 레지스터 값들을 스택에 저장
* SHL, SHR(Shift Left, Right) : Dest 피연산자를 Src 크기만큼 비트 시프트
* SAL, SAR(Shift Arithmetic Left, Right) : 오퍼랜드만큼 비트 시프트. 최상위 비트는 유지하며, 최하위 비트는 0
* ROL, ROR(Rotate Left, RIght) : 회전 명령어. 비트 시프트의 일종으로, 한쪽 끝에서 회전되어 나온 비트는 다른 쪽 끝에서 다시 나타남
* XCHG(Exchange Data) : temp 변수 없이 데이터 교환이 가능
* BSWAP(Byte Swap) : 엔디언 변환을 진행
* LOOP(LOOPD) : ECX가 0이 될 때까지 정해진 레이블로 goto함
* REP : REP 뒤에 오는 스트링 명령을 CX가 0이 될 때까지 반복
* REPE(Repeat until Equal) : ZF가 0이거나 ECX가 0이 될 때까지 반복
* REPNE(Repeat until Not Equal) : 보통 SCAS 명령어와 함께 쓰임. 지정된 데이터 타입별로 문자열을 구분하고, 한 번 구분할 때마다 ECX를 1 차감하는 방법으로 쓰임
* SCAS(Scan String) : 보통 REPE, REPNE와 함께 쓰임. AL 또는 AX 레지스터와 메모리의 데이터를 비교하며, 같은 값이면 ZF를 1로 세팅
* LODS(Load String) : SI의 내용을 AL 또는 AX로 로드
* STOS(Store String) : AL 또는 AX를 ES:DI가 지시하는 메모리에 저장
* MOVS(Move String) : ESI가 가리키고 있는 곳의 값을 EDI가 가리키는 곳으로 복사
* CMP : 주어진 두 개의 operand 비교. SUB와는 달리 operand 값이 변경되지 않고, EFLAGS 레지스터만 변경됨. 두 operand 값이 동일할 때 ZF = 1로 세팅

<br/>

### JMP
* JZ, JE(Jump if Equal) : 조건 분기. ZF = 1이면 점프
* JNE, JNZ(Jump Not Zero) : 결과가 0이 아닐 때 점프
* JLE(Jump if Less or Equal) : <=와 같은 의미를 지님
* JA(Jump if Above) : CMP a, b에서 a가 클 때 점프
* JB(Jump if Below) : CMP a, b에서 b가 클 때 점프

<br/>

### 바이트 오더링
* 빅 엔디언 : 데이터 저장 시, 사람이 보는 방식과 동일하게 저장. 대형 유닉스 서버에 이용되는 RISC 계열 CPU, 네트워크 프로토콜 등에서 빅 엔디언이 사용됨
* 리틀 엔디언 : 저장되는 바이트의 순서를 역순으로 배치하여 저장. 저장된 값의 하위 바이트만 사용할 때, 혹은 산술 연산 시 효율적. 인텔 계열 프로세서에서 사용됨
* 배열의 경우 엔디언 형식과 상관없이 동일한 요소 순서를 지님

<br/>

### 스택
프로세서에서, 함수 내 로컬 변수 임시저장. 함수 호출 시 파라미터 전달. 복귀주소 저장 시에 스택 메모리를 사용. 스택은 높은 주소(아래쪽)에서 낮은 주소(위쪽) 방향으로 값이 추가됨. 스택 포인터의 초기값은 스택 메모리 가장 아래쪽을 가리킴

<br/>

### 16진수 ⇔ 2진수 변환
* 16진수(h) : 1, 2, 3, 4, 5, 6, 7, 8, 9, A, B, C, D, E, F
* 2진수(b) : 0001, 0010, 0011, 0101, 0110, 0111, 1000, 1001, 1010, 1011, 1100, 1101, 1110, 1111

<br/><br/>

## 예제 #1: abex’ crackme #1
* MessageBoxA : 함수 내부에서 ESI = FFFFFFFF로 세팅(Win32 API 호출 시 특정 레지스터 값이 변경되는 경우가 있음. Win32 어셈블리 프로그래밍 시 주의 요망)
* GetDriveTypeA : HDD 타입의 경우, 리턴값(EAX)은 3
* 스택에 파라미터 전달 : 함수 호출 전 PUSH 명령어를 사용하여 필요한 파라미터를 역순으로 스택에 입력하면, 받는 쪽인 함수에서 올바른 순서로 꺼내어 사용

<br/><br/>

## 스택 프레임
EBP(베이스 포인터) 레지스터를 사용하여 스택 내의 로컬 변수, 파라미터, 복귀 주소에 접근하는 기법

<br/>

### 예시 #1: 스택 프레임의 구조
```
PUSH EBP ; 함수 시작. EBP 사용 전, 기존값을 스택에 저장
MOV EBP, ESP ; 현재의 ESP를 EBP에 저장
; 함수 본체. ESP가 변경되어도 EBP가 변경되지 않으므로 로컬 변수와 파라미터 접근 가능
MOV ESP, EBP ; ESP 정리. 함수 시작 시의 값으로 복원
POP EBP ; 리턴하기 전에, 저장해놓았던 원래 값으로 EBP 복원
RETN ; 함수 종료
```

<br/>

### 예시 #2: 스택 공간 예약
ESP = EBP이고 EBP에 함수 시작 시의 초기 값이 저장되어 있을 때
```
SUB ESP, 8
; ESP 값에서 8을 뺌. 함수의 로컬 변수는 스택에 저장되는데, 로컬 변수
; ‘a’, ‘b’는 long 타입이므로 각각 4byte 크기를 가짐. ESP에서 8을 뺌으로써
; 두 변수에게 필요한 메모리 공간을 확보(예약)한 것
```

* MOV DWORD PTR SS:[EBP – 4], 1과 같이 접근 가능(로컬 변수 접근)
  * DWORD PTR SS:[EBP – 4] ≒ *(DWORD*)(EBP – 4)
  * WORD PTR SS:[EBP – 4] ≒ *(WORD*)(EBP – 4)
  * BYTE PTR SS:[EBP – 4] ≒ *(BYTE*)(EBP – 4)
  * ‘SS’를 명시하는 이유 : 해당 메모리가 어떤 세그먼트에 소속되어 있는지 표시. 32bit Windows OS에서 SS, DS, ES 값은 모두 0이므로 큰 의미는 없음
 
<br/>

* 함수가 리턴하기 직전에 EAX에 어떤 값을 입력하면 그대로 리턴값이 됨
* 파라미터가 두 개인 함수 리턴 후, ADD ESP, 8 : 함수에 넘겨준 두 개의 파라미터가 스택에 그대로 남아있으므로 ESP에 8을 더하여 스택을 정리(이처럼 Caller에서 파라미터를 정리하는 방식을 cdecl, 반대로 Callee에서 파라미터를 정리하는 방식을 stdcall이라고 부름)
* XOR EAX, EAX : main() 함수 리턴값(0) 세팅. 같은 값끼리 XOR 시 0이 되는 특성을 이용
* main 함수 시작 이전과 종료 이후에는, 컴파일러에서 추가한 Stub Code 실행

<br/><br/>

## 예제 #2: abex’ crackme #2

<br/>

### Visual Basic
* VB 전용 엔진 : VB 파일은 MSVBVM60.dll이라는 전용 엔진을 사용(The Thunder Runtime Engine)
  * 메시지 박스 출력 시 MSVBVM60.dll!rtcMsgBox() 함수를 사용하며, 이 함수 내부에서 Win32 API인 user32.dll!MessageBoxW() 함수를 호출
* N(Native) code, P(Pseude) code : VB는 컴파일 옵션에 따라 IA-32 Instruction(일반적인 디버거에서 해석 가능)을 사용하는 N code와 VB 엔진으로 가상 머신을 구현하여 바이트코드를 사용하는 인터프리터 언어 개념의 P code로 컴파일이 가능(P code의 정확한 해석을 위해선 VB 엔진을 분석하여 에뮬레이터를 구현하여야 함)
* Event Handler : VB는 주로 GUI 프로그래밍에 사용되며, Event Driven 방식으로 동작. 따라서 WinMain() 혹은 main()이 아니라 각 event handler에 사용자 코드가 위치
* undocumented 구조체 : VB에서 사용되는 각종 정보들은 내부적으로 구조체 형식으로 저장됨
* 간접 호출 : VB 엔진의 메인 함수인 ThunRTMain() 함수 호출 시 CALL 명령에 의해서 JMP DWORD PTR DS:[4010A0]이 실행되는데, 이것이 VC++, VB 컴파일러에서 자주 사용되는 간접 호출(Indirect Call) 기법
* RT_MainStruct 구조체 : ThunRTMain() 함수의 파라미터로 사용됨. 구조체의 멤버는 또 다른 구조체의 주소들이며, VB 엔진은 이들로부터 실행에 필요한 정보를 얻음
* VB의 문자열은 C++의 string 클래스와 마찬가지로 가변 길이 문자열 타입. 문자열 객체는 16byte 크기 데이터로, 내부에 동적으로 할당한 실제 문자열 버퍼 주소를 가지고 있음
* VB에서 함수(event handler)와 함수 사이는 NOP 명령어들로 채워져 있음
* ThunRTMain() 함수는 프로그램 코드가 아니라, VB 엔진 내부에 위치
* 프로그램 시작 시에는 DLL 파일들을 로드하지 않다가, 필요한 부분에서(해당 함수를 호출했을 경우) DLL 파일들을 로드하여 요청을 처리(메모리 절약의 효과)
  * DLL을 호출했다가 다시 메인 프로그램으로 흐름이 돌아오고 다시 DLL을 호출하는 식의 이러한 방식에는, OllyDbg 같은 일반적인 디버거로 분석하기에 어려운 면이 있음. 따라서 SmartCheck, VB Decompiler 등의 프로그램으로 VB 프로그램의 흐름을 파악할 필요가 있음
* __vbaVarTstEq() : 등록(Registration) 여부를 검증하는 것과 관련된 부분에서 자주 사용

<br/>

### Serial 생성 알고리즘
Name 문자열을 읽은 후, 루프를 돌면서 문자를 암호화(XOR, ADD, SUB 등의 연산 이용)
* __vbaVarForInit(), __vbaVarForNext() : 문자열 객체에서 한 글자씩 참조할 수 있도록 해줌(≒ 연결 리스트의 next 포인터)
* __vbaStrVarVal() : 문자열에서 하나의 문자(UNICODE)를 가져옴
* rtcAnsiValueBstr() : 유니코드 – ASCII 변환에 사용
* __vbaVarAdd() : 두 값을 받아 더한 후, 결과값을 지정된 버퍼에 저장
* rtcHexVarFromVar() : ASCII 값을 유니코드 문자열로 변경
* __vbaVarCat() : 문자열 이어붙이기

<br/>

* MOV EAX, DWORD PTR SS:[EBP – 88] 명령에 의해 EAX 레지스터에 문자열 주소가 세팅될 때, [EBP – 88] 변수가 스트링 객체라는 것을 유추 가능

<br/><br/>

## 함수 호출 규약
함수 호출 이후 ESP(스택 포인터)를 어떻게 정리하는지에 대한 약속
* cdecl : Caller에서 스택을 정리. 가변 길이 파라미터를 전달할 수 있다는 장점
* stdcall : Callee에서 스택을 정리(stdcall 방식으로 컴파일을 원할 때는 함수명 전방에 ‘_stdcall’ 키워드를 붙이면 됨). RETN 8(RETN + POP 8)과 같은 방식으로 스택의 정리가 수행됨. Win32 API에서 다른 언어와의 호환을 위해 사용하는 방식이기도 함
* fastcall : 기본적으로 stdcall 방식과 같지만, 함수 파라미터 일부(2개까지)를 레지스터를 이용해 전달(ECX, EDX 레지스터 사용). 빠른 함수 호출이 가능하나, ECX, EDX 레지스터를 다른 용도로 사용할 필요가 있을 때는 추가저인 오버헤드 필요

<br/><br/>

## 예제 #3: Lena’s Reversing for Newbies
* rtxMsgBox()의 리턴값(EAX)은 1(확인의 의미)이어랴 하며, 1 이외의 값이 지정되면 프로그램이 종료됨
* 각 명령어의 길이는 종류에 따라 다르며, 명령어 수정에 따라 뒤쪽 코드를 침범하거나, 길이가 짧아져 코드가 깨지는 현상이 일어날 수 있음(이 경우 빈 공간은 NOP으로 채워야 함)
* rtcMsgBox의 상위 함수 시작 지점을 찾아 바로 리턴시킴으로써 메시지 박스 제거 가능
* stdcall 방식에서 함수의 파라미터 개수를 확인하는 방법 
  1) 함수의 시작 부분에서 스택에 추가한 리턴 주소값을 확인한 후 이동
  2) Callee에서 스택을 정리하므로, 함수 호출 전과 호출 후의 스택 주소 변화를 확인

<br/><br/>

## PE File Format
* PE Header : PE 파일이 실행되기 위한 모든 정보가 구조체 형식으로 저장되어 있음

<br/>

### PE 파일 형식
OBJ 파일 제외, 모두 실행 가능한 파일 형식
* 실행파일 계열 : EXE, SCR(쉘(Explorer.exe)에서 직접 실행)
* 라이브러리 계열 : DLL, OCX
* 드라이버 계열 : SYS
* 오브젝트 파일 계열 : OBJ

<br/>

### PE 파일의 기본 구조

<br/>

<p align="center">
 <img src="https://github.com/mollose/Security/assets/57161613/28bb5c01-45c0-4578-bb72-72592bd707f4" width="700">
</p><br/>

* offset : HxD 등의 프로그램으로 PE 파일을 열었을 때의 오프셋
* VA : 프로그램 가상 메모리 절대 주소(OllyDbg 등으로 PE 파일을 실행했을 때의 주소)
* RVA : 어느 기준위치에서의 상대 주소(VA = ImageBase + RVA)
* PE Header의 끝 부분과 각 Section의 끝에는 최소 기본 단위에 맞추어 널 패딩이 붙음

<br/>

### DOS Header
DOS 파일에 대한 하위 호환성을 고려, PE Header의 제일 앞부분에는 기존 DOS EXE Header를 확장시킨 IMAGE_DOS_HEADER 구조체가 존재(40h 크기)
* e_magic : DOS Signature, 모든 PE 파일의 시작 부분에 존재(4D5A(ASCII 값 “MZ”))
* e_lfanew : NT Header의 offset을 표시. e_lfanew가 가리키는 위치에 NT Header 구조체가 존재해야 함

<br/>

### DOS Stub
DOS 환경에서 실행되는 코드를 가지는 영역. DOS Stub의 존재 여부는 옵션이며, 크기도 일정하지 않음. 코드와 데이터의 혼합으로 이루어져 있으며, 이를 이용하여 DOS와 Windows 모두에서 실행 가능한 파일을 만들 수도 있음

<br/>

### NT Header
IMAGE_NT_HEADERS 구조체 존재(F8h 크기)
* Signature : PE Signature로서 50450000h(“PE”00) 값을 가짐

#### <ins>File Header</ins>
IMAGE_FILE_HEADER 구조체를 가짐
* Machine : CPU별 고유값으로, winnt.h 파일에 정의되어 있음(32bit Intel 호환칩은 14Ch 값)
* NumberOfSections : 섹션의 개수로, 0보다 커야함
* SizeOfOptionalHeader : IMAGE_OPTIONAL_HEADER32의 크기를 나타냄. C 언어의 구조체이기 때문에 그 크기가 결정되어 있음에도 불구하고, Windows의 PE Loader는 SizeOfOptionalHeader 값을 보고 구조체의 크기를 인식(PE32+ 형식의 파일은 IMAGE_OPTIONAL_HEADER64 구조체를 사용하는데, 그 크기에서 IMAGE_OPTIONAL_HEADER32 구조체와는 차이가 있어, SizeOfOptionalHeader에서 구조체의 크기를 명시해줌)
* Characteristics : 파일의 속성(실행가능 형태인지, 혹은 DLL 파일인지)을 나타내며, bit OR 형식으로 조합되어 있음. winnt.h 파일에 값이 정의되어 있음
* TimeDateStamp : 해당 파일의 빌드 시간. 파일 실행에 영향 X

#### <ins>Optional Header</ins>
IMAGE_OPTIONAL_HEADER32 구조체를 가짐
* Magic : 32bit PE 파일은 10Bh, 64bit PE 파일은 20Bh 값을 가짐
* AddressOfEntryPoint : EP(Entry Point)의 RVA 값을 가짐
* ImageBase : 프로세스의 가상 메모리는 0 ~ FFFFFFFFh 범위의 크기인데, 이때, ImageBase는 PE 파일이 메모리 내에서 매핑되는 시작 주소를 나타냄(EXE, DLL 파일은 user memory 영역인 0 ~ 7FFFFFFFh 범위 내, SYS 파일은 kernel memory 영역인 80000000h ~ FFFFFFFFh 범위 내 위치). 일반적으로 개발도구들이 만드는 EXE 파일의 ImageBase 값은 00400000h, DLL 파일은 01000000h임. PE Loader는 PE 파일 실행 시 프로세스를 생성하고, 파일을 메모리에 매핑시킨 후, EIP 레지스터 값을 ImageBase + AddressOfEntryPoint로 세팅
* SectionAlignment, FileAlignment : 파일에서 섹션의 최소 정렬 단위를 나타내는 것이 FileAlignment, 메모리에서 섹션의 최소 정렬 단위를 나타내는 것이 SectionAlignment
* SizeOfImage : PE 파일이 메모리에 로드되었을 때 가상 메모리에서 PE Image(IMAGE란 PE 구조를 만든 MS에서 만들어 낸 용어로, 보통은 PE 파일이 메모리에 로드된 상태에서 PE 메모리 영역 전체를 지칭)가 차지하는 크기(모든 섹션의 보정된(SectionAlignment 크기로 정렬된) VirtualSize 값 + PE Header 크기)
* SizeOfHeader : PE Header의 전체 크기로, 파일 시작 위치에서 SizeOfHeader offset만큼 떨어진 위치에 첫 번째 섹션이 위치
* Subsystem : Driver File / GUI File / CUI File
* NumberOfRvaAndSizes : 마지막 멤버인 DataDirectory 배열의 크기로서, 구조체 정의에 IMAGE_NUMBEROF_DIRECTORY_ENTRIES(16)이라고 명시되어 있지만, PE Loader는 NumberOfRvaAndSizes 값을 보고 배열의 크기를 인식
* DataDirectory : IMAGE_DATA_DIRECTORY 구조체의 배열로서, 배열의 각 항목마다 정의된 값을 가짐. EXPORT, IMPORT, RESOURCE, TLS Directory 등이 포함됨

#### <ins>Section Header</ins>
각 Section의 속성을 정의. 각 섹션별 IMAGE_SECTION_HEADER 구조체의 배열로 이루어져 있음
* VirtualSize : 메모리에서 섹션이 차지하는 크기
* VirtualAddress : 메모리에서 섹션의 시작 주소(RVA)
* SizeOfRawData : 파일에서 섹션이 차지하는 크기
* PointerToRawData : 파일에서 섹션의 시작 위치
* Characteristics : 섹션의 특징(bit OR)
* Name : 어떠한 명시적 규칙도 없는 참고용 정보

<br/>

### RVA to RAW 
PE 파일이 메모리에 로드되었을 때 각 섹션에서 메모리의 주소(RVA)와 파일 offset을 잘 매핑할 수 있어야 함. 이러한 매핑을 일반적으로 RVA to RAW라고 함. RAW – 섹션의 PointerToRawData = RVA – 섹션의 VirtualAddress. 따라서 RAW = RVA – 섹션의 VirtualAddress + 섹션의 PointerToRawData
* RAW = ABA8h(RVA) - 9000h(VA) + 7C00h(PointerToRawData) = 97A8h로, 파일 offset에 있어서 세 번째 섹션에 위치할 때(RVA는 두 번째 섹션에 위치), 해당 RVA에 대한 RAW 값은 정의할 수 없음. 두 번째 섹션의 VirtualSize 값이 SizeOfRawData 값보다 크기 때문에 발생

<br/>

### DLL(Dynamic Linked Library) 
동적 연결 라이브러리. 라이브러리의 바이너리 코드를 그대로 가져와 프로그램에 삽입시키던 DOS 방식과는 달리, 라이브러리를 별도의 파일로 구성하여 필요할 때마다 불러와 사용. 한 번 로드된 DLL의 코드, 리소스는 Memory Mapping 기술로 여러 프로세스에서 공유해 사용할 수 있도록 함
* Explicit Linking : 프로그램에서 사용하는 순간에 로딩, 사용 후 메모리에서 해제
* Implicit Linking : 프로그램이 시작할 때 같이 로딩, 프로그램 종료 시 메모리에서 해제
  * IAT는 Implicit Linking에 대한 메커니즘 제공
 
<br/>

### IAT(Import Address Table)
프로그램이 어떤 라이브러리에서 어떤 함수를 사용하고 있는지를 기술한 테이블. API 호출 시 함수를 해당 직접 호출하지 않고, IAT 메모리 영역에 접근하여 그 주소값을 가져옴(ex. CALL DWORD PTR DS:[1001104]. 다양한 환경에서 구동되기 때문, DLL Relocation 등의 요인)

#### <ins>IMAGE_IMPORT_DESCRIPTOR</ins>
PE 파일은 자신이 어떤 라이브러리를 Import하고 있는지 IMAGE_IMPORT_DESCRIPTOR 구조체에 명시. 참조하는 Library의 개수만큼의 크기의 구조체 배열 형식으로 존재하며, 배열의 마지막은 NULL 구조체로 끝남
* OriginalFirstThunk : INT(ImportNameTable)의 주소(RVA)
* Name : Library 이름 문자열의 주소(RVA)
* FirstThunk : IAT(Import Address Table)의 주소(RVA)
* INT와 IAT는 longtype 배열이고 NULL로 끝남
* INT에서 각 원소의 값은 IMAGE_IMPORT_BY_NAME 구조체의 주소값(RVA)
* INT와 IAT의 크기는 같아야 함
* IID는 PE Body에 위치하며, PE Header의 IMAGE_OPTIONAL_HEADER32.DataDirectory[1].VirtualAddress에 구조체 배열의 시작 주소가 명시되어 있음
#### <ins>PE Loader가 Import 함수 주소를 IAT에 입력하는 순서</ins>
1) IID Name 멤버를 읽어 라이브러리의 이름 문자열을 얻음
2) 해당 라이브러리를 로딩(LoadLibrary() 사용)
3) IID의 OriginalFirstThunk 멤버를 읽어 INT 주소를 얻음
4) INT에서 배열의 값을 하나씩 읽어 해당 IMAGE_IMPORT_BY_NAME 주소(RVA)를 얻음
5) IMAGE_IMPORT_BY_NAME의 Hint(ordinal) 또는 Name 항목을 이용하여(GetProcAddress() 사용) 해당 함수의 시작 주소를 얻음
6) IID의 FirstThunk 멤버를 읽어 IAT 주소를 얻음
7) 해당 IAT 배열 내에서 해당하는 위치에, 위에서 구한 함수 주소를 저장
8) INT가 끝날 때까지 (4) ~ (7)의 과정 반복
* 일반적인 DLL은 IAT에 실제 주소가 하드코딩 되어있지 않고, INT와 동일한 값을 가지는 경우가 많음
* 일반적인 DLL 파일은 ImageBase가 10000000h로 되어있어 DLL relocation이 발생하지만, Windows 시스템 DLL 파일(kernel32, user32, gdi32 등)은 고유의 ImageBase가 있어서 DLL relocation이 발생하지 않음

<br/>

### EAT(Export Address Table)
라이브러리 파일에서 제공하는 함수를 다른 프로그램에서 가져다 사용할 수 있도록 해주는 메커니즘

#### <ins>IMAGE_EXPORT_DIRECTORY</ins>
라이브러리의 EAT를 설명하는 구조체로, 주로 DLL 파일에 존재. IMAGE_EXPORT_DIRECTORY는 IAT를 설명하는 IID 구조체와는 달리 PE 파일에 단 하나만 존재(IID는 PE 파일이 여러 개의 라이브러리를 동시에 Import 할 수 있다는 특성으로 인해 배열 형태로 존재)
* NumberOfFunctions : 실제 export 함수 개수
* NumberOfNames : export 함수 중 이름을 가지는 함수 개수(≤ NumberOfFunctions)
* AddressOfFunctions : export 함수들의 시작 위치 배열의 주소(배열의 원소 개수 = NumberOfFunctions)
* AddressOfNames : 함수 이름 배열의 주소(배열의 원소 개수 = NumberOfNames)
* AddressOfOrdinals : ordinal 배열의 주소(배열의 원소 개수 = NumberOfNames)
* AddressOfFunctions, AddressOfNames는 각각 4byte RVA 배열의 시작을 가리키며, AddressOfOrdinals는 2byte Ordinal 배열의 시작을 가리킴

#### <ins>GetProcAddress()가 함수 이름을 이용하여 라이브러리에서 함수 주소를 얻는 순서</ins>
1) AddressOfNames 멤버를 이용해 함수 이름 배열로 이동
2) 문자열 주소가 저장된 함수 이름 배열에서 문자열 비교(strcmp)로 원하는 이름을 찾음(이때의 배열 인덱스는 name_index)
3) AddressOfNameOrdinals 멤버를 이용해 Ordinal 배열로 이동
4) Ordinal 배열에서 name_index로 해당 ordinal_index 값을 찾음
5) AddressOfFunctions 멤버를 이용해 함수 주소 배열(EAT)로 이동
6) 함수 주소 배열(EAT)에서 ordinal_index를 인덱스로 하여 함수의 시작 주소(RVA)를 얻음
* export하는 함수의 이름이 존재하지 않거나 index != ordinal인 경우도 있음
* 종종 함수 이름 없이 Ordinal(고유번호)만 공개하는 경우가 있는데, 이 경우엔 Ordinal로 해당 함수의 주소를 찾는 것이 가능(ex. pFunc = GetProcAddress(5);)

<br/>

### 실행 압축 
실행(PE) 파일을 대상으로 파일 내부에 압축해제 코드를 포함하게 하여, 실행되는 순간에 메모리에서 압축을 해제하도록 하는 기술. 실행 압축 파일 역시 PE 파일이며, EP 코드에서 decoding 루틴이 실행되면서 메모리에서 압축을 해제한 후 실행(실행시간이 미세하게 느려짐)
* 패커(Packer) : Run-Time 패커. PE 파일을 실행 압축 파일로 만들어주는 유틸리티
* 프로텍터(Protector) : 실행 압축과 함께, 리버싱을 막기 위한 다양한 기법 추가(원본 파일보다 크기가 커지는 경향). PE 파일 자체와 실행 시의 프로세스 메모리 보호

<br/>

<p align="center">
 <img src="https://github.com/mollose/Security/assets/57161613/f8b4dda1-d191-4744-9202-81fc4b8187fd" width="700">
</p><br/>

* 섹션 이름 변경(“.text” ⇒ “.UPX0”, “.data” ⇒ “.UPX1”)
* 첫 번째 섹션의 RawDataSize = 0
* EP는 두 번째 섹션에 위치(원본 파일에서는 첫 번째 섹션에 위치)
* UPX 패커의 특징(notepad_upx.exe의 경우) : 첫 번째 섹션의 SizeOfRawData는 0이지만 VirtualSize는 높은 값으로 세팅되어 있음. 이 경우, 압축해제 코드와 원본 코드는 두 번째 섹션에 존재하며, 실실행 시 압축된 코드를 첫 번째 섹션에 해제시킴

<br/>

### 예제 #4: notepad_upx 디버깅
* GetModuleHandleA() : notepad.exe. 프로세스의 ImageBase를 구함
* notepad_upx의 EP는 두 번째 섹션의 끝 부분에 존재하며, notepad의 압축된 원본 코드는 EP 주소 위쪽에 존재함
* EP 진입 이후 PUSHAD 명령으로 레지스터 값을 저장한 뒤, ESI와 EDI 레지스터를 각각 두 번째 섹션 시작 주소와 첫 번째 섹션 시작 주소로 세팅. ESI가 가리키는 버퍼에서, 압축해제된 원본파일의 코드가 저장될 장소인 EDI가 가리키는 위치로 메모리 복사 진행

#### <ins>방대한 코드를 트레이싱할 때의 원칙</ins>
루프를 만나면 그 역할을 알아보고 탈출
* 디코딩 루프 : ESI가 가리키는 두 번째 섹션의 주소에서 차례대로 값을 읽어, 적절한 연산을 거쳐 압축을 해제한 뒤 EDI가 가리키는 첫 번째 섹션에 값을 써줌
* CALL / JMP 복원 루프 : 원본 코드의 CALL / JMP 명령어(opcode E8 / E9)의 destination 주소를 복원하는 코드(EDI 위치를 대상으로 바이트별 비교를 통해 E8 / E9 명령어를 찾음)
* IAT 세팅 루프 : 디코딩을 거친 이후의 두 번째 섹션(UPX1) 영역에는 notepad.exe에서 사용하는 API 이름 문자열이 저장되어 있음. 이 API 문자열을 이용해 GetProcAddress() 함수를 호출하여 시작 주소를 얻은 후, EBX가 가리키는 원본 notepad.exe의 IAT 영역에 API 주소를 입력

최종적으로 JMP 명령어에 의해 OEP(Original Entry Point)로 이동
* UPX 패커의 특징 중 하나는 EP 코드가 PUSHAD, POPAD로 둘러싸여 있다는 것으로, OEP로 가는 JMP 명령어가 POPAD 명령어 직후에 나타난다
  * PUSHAD에서 사용한 스택 주소를 따라가 하드웨어 브레이크 포인트를 설치함으로써, 동일한 주소에 접근하는 POPAD 명령어를 찾을 수 있음

<br/><br/>

## Base Relocation Table
* 프로세스 생성 시 EXE 파일이 가장 먼저 메모리에 로드되므로 EXE에서는 재배치를 고려할 필요가 없었으나, ASLR 기능의 추가로 EXE 파일 이미지 역시 실행 시마다 랜덤한 주소에 로드됨(시스템 DLL 역시 고유한 ImageBase가 있음에도 ASLR의 영향을 받아 시스템 부팅 시마다 위치가 바뀌게 됨)

<br/>

### PE 재배치
프로그램에 하드코딩된 메모리 주소를 현재 로드된 주소에 맞게 변경해주는 작업
1) 프로그램에서 하드코딩된 주소 위치를 찾음
2) 값을 읽은 후, ImageBase만큼 뺌(VA ⇒ RVA)
3) 실제 로딩 주소를 더함(RVA ⇒ VA)
* 이때, PE 파일 내부에 하드코딩 주소들의 offset을 모아놓은 목록이 존재하는데, 이를 Relocation Table이라고 함(PE 파일 빌드 과정에서 제공됨). PE 헤더의 Base Relocation Table 항목을 통해 접근 가능(DataDirectory[5])

<br/>

### IMAGE_BASE_RELOCATION
구조체 배열을 이루며 Base Relocation Table을 구성. 배열의 마지막은 NULL 구조체로 끝남
* VirtualAddress : TypeOffset 배열의 기준 주소(RVA)
* SizeOfBlock : 각 단위 블록의 전체 크기. 기준 주소별로 TypeOffset 배열이 블록을 이루고 있음
* TypeOffset 배열 : 구조체 내에 표시된 주석은 이 구조체 밑으로 WORD 타입의 배열이 따라온다는 것을 의미하며, 배열 항목의 값은 프로그램에 하드코딩된 주소들의 offset
* TypeOffset 값은 Type(4비트)과 Offset(12비트)가 합쳐진 형태
  * 일반적인 Type 값은 3(IMAGE_REL_BASE_HIGHLOW)이고, 64bit용 PE+ 파일에선 A(IMAGE_REL_BASED_DIR64)임(간혹 악성코드 중에선 Type을 0(IMAGE_REL_BASED_ABSOLUTE))으로 수정하여 PE 로더의 재배치 과정을 피하는 경우도 있음)
  * 프로그램에서 하드코딩 주소가 있는 Offset은, 섹션의 VirtualSize + Offset으로 계산됨. Offset을 가리키는 하위 12비트는 최대 1000의 값을 가지는데, 이를 초과하는 값을 표시하기 위해서는 그에 맞게 블록을 하나 더 추가할 필요

<br/><br/>

## 예제 #5: 실행 파일에서 .reloc 섹션 제거하기
* DLL, SYS 파일에선 BaseRelocationTable이 필수요소지만, EXE 파일에선 제거해도 실행에 큰 영향이 없음
1) .reloc 섹션 헤더 정리 : 섹션 헤더 내용을 0으로 덮어씀
2) .reloc 섹션 제거 : HxD의 ‘Delete’로 물리적 제거
3) IMAGE_FILE_HEADER 수정 : NumberOfSection 필드 수정
4) IMAGE_OPTIONAL_HEADER 수정; SizeOfImage 필드 수정

<br/><br/>

## UPack
PE 헤더를 독특한 방식으로 변형하는 기법을 사용하는 실행 압축기

<br/>

### UPack의 PE 헤더 분석
* 헤더 겹쳐 쓰기 : MZ 헤더와 PE 헤더를 교묘하게 겹쳐 씀. e_magic과 e_lfanew를 제외한 나머지 MZ 멤버들은 프로그램 실행에 있어 아무 의미를 가지지 않으므로, 그리고 e_lfanew 값에 따라 IMAGE_NT_HEADERS의 위치가 가변적일 수 있으므로 두 헤더의 겹쳐 쓰기가 가능
* IMAGE_FILE_HEADER.SizeOfOptionalHeader : SizeOfOptionalHeader의 값을 변경하여 헤더 안에 디코딩 코드를 삽입(Optional Header와 Section Header 사이의 공간에 코드 추가)
* IMAGE_OPTIONAL_HEADER.NumberOfRvaAndSizes : NumberOfRvaAndSizes 값을 조작함으로써(10 ⇒ A) IMAGE_DATA_DIRECTORY 구조체 배열의 뒤쪽 6개 원소들을 무시하도록 만들며, 무시된 영역에 자신의 코드를 덮어씀(디코딩 코드)
* IMAGE_SECTION_HEADER : offset to relocations, offset to line numbers, number of relocations, number of line numbers 구조체 멤버들은 실행에 있어 아무런 의미가 없는 멤버 값들

#### <ins>섹션 겹쳐쓰기</ins>
섹션과 헤더를 겹쳐쓰는 것

<br/>

<p align="center">
 <img src="https://github.com/mollose/Security/assets/57161613/5798f030-bb55-40ea-80fa-bf1864ada8a4" width="700">
</p><br/>

* 두 번째 섹션의 크기는 파일의 대부분을 차지할 정도로 큰데, 이곳에 원본 파일이 압축되어 있음
* 메모리에서 첫 번째 섹션의 크기는 14000으로, 이는 원본 파일의 SizeOfImage와 같음. 두 번째 섹션에 압축된 파일 이미지를 메모리 이미지의 첫 번째 섹션에 압축해제(notepad.exe의 원본에는 3개의 섹션이 있지만, 이를 하나의 섹션에 풀어내는 것)

#### <ins>RVA to RAW</ins>
일반적으로 섹션 시작의 PointerToRawData 값은 FileAlignment의 배수가 되어야 하며, 그러한 이유로 첫 번째 섹션의 PointerToRawData(ex. 10)는 PE 로더에서 강제로 배수에 맞추어 인식됨(ex. 0). 그러나 많은 PE 관련 유틸리티에선 첫 번째 섹션의 시작 offset 값 10을 그대로 RVA to RAW 변환 공식에 대입하기 때문에 잘못된 메모리 참조를 겪게 됨

#### <ins>Import Table(IMAGE_IMPORT_DESCRIPTOR array)</ins>
Import Table은 IMAGE_IMPORT_DESCRIPTOR 구조체 배열이며 마지막은 NULL 구조체로 끝나야 하나, 1EE ~ 201 offset에 위치한 첫 번째 구조체의 뒤에는 두 번째 구조체도 NULL 구조체도 오지 않음. 그러나 세 번째 섹션은 200 offset을 기준으로 끝나므로 200 offset 이후에 위치한 데이터들은 세 번째 섹션 메모리에 매핑되지 않고, 메모리에 로딩 시 이러한 나머지 영역은 NULL로 채워지므로, PE 스펙에 어긋나지 않게 됨

#### <ins>IAT(Import Address Table)</ins>
IMAGE_IMPORT_DESCRIPTOR의 Name 멤버의 RVA 값은 Header 영역에 속하며, 이 위치는 DOS 헤더에서 사용되지 않는 공간. 보통은 INT를 따라가 API 이름 문자열을 찾지만, INT의 주소값이 0일 때는 IAT를 따라가도 상관없음. IAT 영역은 동시에 INT의 역할도 하므로, Name Pointer 배열을 이루고 있고 각 RVA 값은 모두 헤더 영역 내에 포함되어 있음
* 이때 LoadLibraryA와 GetProcAddress API를 임포트하는데, 두 함수는 원본 파일의 IAT를 구성할 때 편리하게 사용되기에 일반적으로 패커에서 많이 사용

<br/><br/>

## 예제 #6: UPack 디버깅(OEP 찾기)
* UPack이 NumberOfRvaAndSizes 값을 변경하기 때문에 OllyDbg의 초기 검증 과정에서 에러가 발생하는데, 이 때문에 EP로 가지 못하고 ntdll.dll 영역에서 멈춤
  * 강제로 EP를 설정해주어야 함(New Origin here)
* 디코딩 루프 : 첫 번째 섹션 내의 주소를 가리키는 EDI 값에 대해 MOVS, STOS 명령어를 실행하여 압축을 해제한 메모리를 작성. CMP, JB 명령어를 사용해 반복
* IAT 세팅 : UPack이 Import하는 두 개의 함수, LoadLibrary와 GetProcAddress를 이용하여 루프를 돌면서, 원본 notepad의 IAT를 구성(notepad에서 import하는 실제 함수)
* 위의 과정이 끝나면 RETN 명령어를 따라 OEP로 이동

<br/><br/>
## 인라인 패치
인라인 코드 패치는 원하는 코드를 직접 수정하기 어려울 때, 간단히 ‘코드 케이브(Code Cave)’ 패치 코드를 삽입하여 실행함으로써 프로그램을 패치하는 기법

<br/>

<p align="center">
 <img src="https://github.com/mollose/Security/assets/57161613/6e9abfe1-4f64-4d45-81cb-1479903e92f3" width="800">
</p><br/>

* 패치하고자 하는 코드가 암호화된 OEP 영역에 존재한다면, 위치를 알고 있다 하더라도 부호화 문제로 인해 단순한 방법으로는 패치할 수 없음
  * 파일 내에 코드 케이브를 설치한 후, 복호화 과정 이후에 JMP 명령어를 수정하여 코드 케이브로 이동. 즉, 실행될 때마다 프로세스 메모리의 코드를 패치
 
<br/><br/>

## 예제 #7: Patchme 인라인 패치 실습
* 복호화 루프 : XOR BYTE PTR DS:[EBX], 44 명령으로 특정 영역 XOR 복호화
* Checksum 계산 루프 : EDX를 0으로 초기화한 후, 특정 주소영역에서 4byte 단위 순차적으로 값을 읽어 들여 EDX 레지스터에 누적. Checksum 값은 레지스터의 overflow를 무시하며, 최종적으로 EDX에 남은 값을 사용. 이후의 CMP, JE 명령에서 Checksum 값과 31EB8DB0을 비교한 후, 값이 같다면(코드가 변조되지 않았다면) OEP로 이동
* Anti-disassembly 테크닉을 사용하는 파일들에 대해 OllyDbg는 종종 코드를 데이터로 인식하는 실수를 일으킴. 이 경우 ‘Analyse code’(Ctrl + A) 기능을 사용하면 디스어셈블 가능
* 패치하려는 문자열이 이중으로 암호화되어 있고, 프로그램 내에서 문자열 영역에 대해 Checksum 검증을 수행하기에, 인라인 패치 방법을 사용해야 함

<br/>

<p align="center">
 <img src="https://github.com/mollose/Security/assets/57161613/841094b1-7f07-478b-b8b5-99e5733868eb" width="400">
</p><br/>

```
[EP Code]
    [Decoding Code]
        XOR [B] with 44         
        XOR [A] with 7
        XOR [B] with 11
        [A]
            Checksum [B]
            XOR [C] with 17
            JMP OEP
```

* 패치 코드를 설치하는 위치
  * 파일의 빈 영역에 설치. 패치 코드의 크기가 작은 경우 사용
  * 마지막 섹션을 확장 후 설치
  * 새로운 섹션을 추가한 후 설치
 
* 일반적인 PE 파일의 코드 섹션은 쓰기 속성이 없는데, 패커, Cryptor 류의 파일들은 코드 섹션에 쓰기 속성이 존재(프로그램 내에서 복호화 작업을 수행하기 위해서는 반드시 섹션 헤더에 쓰기 속성을 추가하여 쓰기 권한을 얻어야 함)

* 패치 코드 만들기

```
MOV ECX, 0C
MOV ESI, unpackme.004012A8 ; ASCII “ReverseCore“
MOV EDI, unpackme.00401123 ; ASCII “You must patch this NAG!!!“
REP MOVS BYTE PTR ES:[EDI], BYTE PTR DS:[ESI]
MOV ECX, 9
MOV ESI, unpackme.004012B4 ; ASCII “Unpacked“
MOV EDI, unpackme.0040110A ; ASCII “You must unpack me!!!“
REP MOVS BYTE PTR ES:[EDI], BYTE PTR DS:[ESI]
JMP unpackme.0040121E
DB 00
ASCII “ReverseCore“, 0
ASCII “Unpacked“, 0
```

* 패치 코드 실행하기 : JMP OEP(40120E) ⇒ JMP ‘CodeCave’(401280). 그러나 명령어가 위치한 [A] 영역은 XOR 7로 암호화되어 있으므로, 이를 고려하여 Instruction 작성 시 XOR 명령을 수행한 후 사용하여야 함

<br/><br/>

## Windows 메시지 후킹
Windows는 Event Driven 방식으로 동작하는 GUI 제공. 이벤트 발생 시 OS는 미리 정의된 메시지를 해당 응용 프로그램으로 통보. 메시지 훅이란, 이런 메시지를 중간에서 엿보거나 가로채는 것

<br/>

<p align="center">
 <img src="https://github.com/mollose/Security/assets/57161613/a8fb0b89-6798-4467-b78b-1b3067a40491" width="400">
</p><br/>

### 키보드 입력 메시지의 처리 과정
1) 키보드 입력 발생 시 WM_KEYDOWN 메시지가 OS message queue에 추가
2) OS는 어느 응용 프로그램에서 이벤트가 발생했는지 파악해서 OS message queue에서 메시지를 꺼내어 해당 응용 프로그램의 메시지 큐에 추가
3) 응용 프로그램은 자신의 메시지 큐를 모니터링하고 있다가 WM_KEYDOWN의 추가를 확인하고 해당 event handler를 호출
* 키보드 메시지 훅이 설치되었다면, OS 메시지 큐와 응용 프로그램 메시지 큐 사이에 설치된, 훅 체인에 있는 키보드 메시지 훅들이 응용 프로그램에 앞서서 해당 메시지를 볼 수 있음. 키보드 메시지 훅 함수 내에서는 메시지를 단순히 엿보는 것뿐만 아니라 메시지 자체의 변경도 가능하며, 메시지를 가로챔으로 아래로 내려 보내지 않게 할 수도 있음

<br/>

### SetWindowsHookEx()
메시지 훅은 SetWindowsHookEx() API 함수로 구현 가능
> HHOOK SetWindowsHookEx(int idHook, HOOKPROC lpfn, HINSTANCE hMod, DWORD dwThreadId);
* idHook : hook type
* lpfn : hook procedure
* hMod : hook procedure가 속한 DLL 핸들
* dwThreadId : hook을 걸고 싶은 thread의 ID(0을 주고 호출하면 글로벌 훅이 설치됨(모든 프로세스 영향))
* SetWindowsHookEx()를 이용하여 훅을 설치해놓으면, 어떤 메시지가 발생했을 때 운영 체제가 hook procedure를 가진 DLL 파일을 프로세스에 강제로 인젝션하고, 등록된 hook procedure를 호출

1) HookMain.exe는 훅 프로시저(KeyboardProc)가 존재하는 KeyHook.dll을 최초로 로드하고, SetWindowsHookEx()를 사용해 키보드 훅을 설치
2) 다른 프로세스에서 키 입력 이벤트가 발생하면, OS에서 해당 프로세스의 메모리 공간에 KeyHook.dll을 강제로 로드하고 KeyboardProc() 함수 호출

<br/>

### 예시 #3: HookMain.cpp

```cpp
#include “stdio.h“
#include “conio.h“ // getch() 함수를 사용하기 위해
#include “windows.h“
#define DEF_DLL_NAME “KeyHook.dll“
#define DEF_HOOKSTART “HookStart“
#define DEF_HOOKSTOP “HookStop“
typedef void(*PFN_HOOKSTART)();
typedef void(*PFN_HOOKSTOP)();
void main()
{
    HMODULE hDll = NULL;
    PFN_HOOKSTART HookStart = NULL;
    PFN_HOOKSTOP HookStop = NULL;
    char ch = 0;
    hDll = LoadLibraryA(DEF_DLL_NAME);
    HookStart = (PFN_HOOKSTART)GetProcAddress(hDll, DEF_HOOKSTART);
    HookStop = (PFN_HOOKSTOP)GetProcAddress(hDll, DEF_HOOKSTOP);
    HookStart(); // 후킹 시작
    printf(“press ’q’ to quit!\n“);
    while(_getch() != ’q’); // _getch()는 하나의 문자를 입력받으나, 출력하진 않음
    HookStop(); // 후킹 종료
    FreeLibrary(hDll); // KeyHook.dll 언로딩
}
```

<br/>

### 예시 #4: KeyHook.cpp
```cpp
#include “stdio.h“
#include “windows.h“
#define DEF_PROCESS_NAME “notepad.exe“
HINSTANCE g_hInstance = NULL;
HHOOK g_hHook = NULL;
HWND g_hWnd = NULL;
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD dwReason, LPVOID lpvReserved)
{
    switch(dwReason)
    {
    case DLL_PROCESS_ATTACH :
         g_hInstance = hinstDLL;
        break;
    case DLL_PROCESS_DETACH :
         break;
    }
    return TRUE;
}
LRESULT CALLBACK KeyboardProc(int nCode, WPARAM wParam, LPARAM lParam)
{
    char szPath[MAX_PATH] = {0, };
    char* p = NULL;
    // nCode == 0이라면 파라미터들은 keystroke 정보 포함
    if (nCode == 0)
    {
        // bit 31 : 0 = key press, 1 = key release
        if(!(!Param & 0x80000000))
        {
            GetModuleFileNameA(NULL, szPath, MAX_PATH);
            p = strrchr(szPath, ’\\’); // 주어진 문자열에서 뒤에서부터, 문자 검색
            if(!_stricmp(p + 1, DEF_PROCESS_NAME)) // 대소문자 구분 X
                return 1; // 메시지를 넘겨주지 않고 바로 리턴
        }
    }
    return CallNextHookEx(g_hHook, nCode, wParam, lParam);
}
// C++의 경우 오버로딩 등의 이유로, 같은 이름의 함수가 다른 함수일 수 있음
#ifdef __cplusplus
extern “C“{
#endif
__declspec(dllexport) void HookStart() // DLL에서 구현한 함수를 외부로 노출
{
    g_hHook = SetWindowsHookEx(WH_KEYBOARD, KeyboardProc, g_hInstance, 0);
}
__declspec(dllexport) void HookStop()
{
    if(g_hHook)
    {
        UnhookingWindowsHookEx(g_hHook);
        g_hHook = NULL;
    }
}
#ifdef __cplusplus
}
#endif
```

<br/><br/>

## DLL 인젝션
다른 프로세스에게 LoadLibrary() API 함수를 스스로 호출하도록 명령하여, 사용자가 원하는 DLL을 로드하는 것(LoadLibrary() API 함수로 DLL 로딩 시 해당 DLL의 DllMain() 함수가 자동으로 호출됨). 해당 프로세스 메모리에 대해 정당한 접근 권한을 가지게 됨

<br/>

### 원격 스레드 생성(CreateRemoteThread() API 함수) 방식

#### <ins>예시 #5: Myhack.cpp</ins>

```cpp
#include “windows.h“
// TCHAR형의 사용을 위해 삽입. _UNICODE 정의 시 wchar_t 타입으로,
// _MBCS 정의 시 char 타입으로 자동 형 변환
#include “tchar.h“
#pragma comment(lib, “urlmon.lib“) // 명시적인 라이브러리 링크
#define DEF_URL (L“http://www.naver.com/index.html“)
#define DEF_FILE_NAME (L“index.html“)
HMODULE g_hMod = NULL;
DWORD WINAPI ThreadProc(LPVOID lParam)
{
    TCHAR szPath[_MAX_PATH] = {0, };
    if(!GetModuleFileName(g_hMod, szPath, MAX_PATH))
        return FALSE;
    TCHAR* p = _tcsrchr(szPath, ’\\’); // strrchr에 대응하는 TCHAR용 함수
    if(!p)
        return FALSE;
    _tcscpy_s(p + 1, _MAX_PATH, DEF_FILE_NAME); // strcpy_s의 TCHAR용 함수
    // 인터넷에서 비트 데이터들을 내려 받아 파일로 저장
    URLDownloadToFile(NULL, DEF_URL, szPath, 0, NULL);
    return 0;
}
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
    HANDLE hThread = NULL;
    g_hMod = (HMODULE)hinstDLL;
    switch(fdwReason)
    {
    case DLL_PROCESS_ATTACH :
        OutputDebugString(L“myhack.dll Injection!!!“);
        hThread = CreateThread(NULL, 0, ThreadProc, NULL, 0, NULL);
        // 스레드 참조 카운트를 미리 하나 감소시키고, 이후 함수 반환 시 나머지 감소
        CloseHandle(hThread);
        break;
    }
    return TRUE;
}
```

#### <ins>예시 #6: InjectDLL.cpp</ins>

```cpp
#include “windows.h“
#include “tchar.h“
BOOL InjectDll(DWORD dwPID, LPCTSTR szDllPath)
{
    HANDLE hProcess = NULL;
    hThread = NULL;
    HMODULE hMod = NULL;
    LPVOID pRemoteBuf = NULL;
    DWORD dwBufSize = (DWORD)(_tcslen(szDllPath) + 1) * sizeof(TCHAR);
    LPTHREAD_START_ROUTINE pThreadProc;
    if(!(hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwPID)))
    {
        _tprintf(L“OpenProcess(%d) failed!!! [%d]\n“, dwPID, GetLastError());
        return FALSE;
    }
    pRemoteBuf = VirtualAllocEx // 해당 프로세스 가상 메모리 공간에 할당
        (hProcess, NULL, dwBufSize, MEM_COMMIT, PAGE_READWRITE);
    WriteProcessMemory // 해당 프로세스 메모리 영역에 데이터를 씀
        (hProcess, pRemoteBuf, (LPVOID)szDllPath, dwBufSize, NULL);
    hMod = GetModuleHandle(L“kernel32.dll“);
    pThreadProc = (LPTHREAD_START_ROUTINE)GetProcAddress(hMod, “LoadLibraryW“);
    hThread = CreateRemoteThread // 다른 프로세스 내 스레드 생성
        (hProcess, NULL, 0, pThreadProc, pRemoteBuf, 0, NULL);
    WaitForSingleObject(hThread, INFINITE);
    CloseHandle(hThread);
    CloseHandle(hProcess);
    return TRUE;
}
int _tmain(int argc, TCHAR* argv[]) // main 함수의 TCHAR형
{
    if(argc != 3)
    {
        _tprintf(L“USAGE : %s pid dll_path\n“, argv[0]);
        return 1;
    }
    if(InjectDll((DWORD)_tstol(argv[1]), argv[2]))
        _tprintf(L“InejctDll(\“%s\“) success!!!\n“, argv[2]);
    else
        _tprintf(L“InjectDll(\“%s\“) failed!!!\n“, argv[2]);
    return 0;
}
```

* OpenProcess() : PROCESS_ALL_ACCESS 권한의 프로세스 핸들을 구함
* VirtualAllocEx() : 상대방 프로세스 메모리 공간에 버퍼 할당. 할당된 버퍼 주소는 상대방 프로세스의 메모리 주소이며, 이곳에 로드할 DLL 파일의 경로를 써넣음으로써 대상 프로세스가 인젝션할 DLL의 경로를 알게 함
* WriteProcessMemory() : 앞에서 할당받은 버퍼 주소에 DLL 경로 문자열을 써줌
* LoadLibraryW() 주소 구하기 : Windows 운영 체제에서, kernel32.dll은 프로세스마다 같은 주소에 로드된다는 점 이용. InjectDll.exe 프로세스에 로드된 kernel32.dll의 주소를 그대로 사용
* CreateRemoteThread() : 대상 프로세스에서 원격 스레드를 실행. 첫 번째 파라미터(hProcess)를 제외하곤 일반적인 CreateThread() 함수와 동일(일반적인 스레드 생성 함수에서 ThreadProc() 함수와 그 파라미터를 전달한다면, DLL 인젝션에서는 LoadLibrary() 함수와 라이브러리 경로 주소값을 파라미터로 전달)
* GetModuleFileName() : 대상 프로세스가 자신일 때, 첫 번째 인자는 NULL

<br/>

### 레지스트리 이용(AppInit_DLLs) 방식 
Windows 운영 체제에서 기본으로 제공하는 AppInit_DLLs 레지스트리 항목에 인젝션을 원하는 DLL 경로 문자열을 쓰고, LoadAppInit_DLLs 레지스트리 항목의 값을 1로 변경한 후 재부팅하면, 실행되는 모든 프로세스에 해당 DLL을 인젝션 해줌(User32.dll이 프로세스에 로드될 때, AppInit_DLLs 항목을 읽어서 값이 존재하면 LoadLibrary()를 이용하여 사용자 DLL을 로딩. 따라서user32.dll을 로드하는 프로세스에만 인젝션이 가능하다는 단점이 있음)

#### <ins>예시 #7: myhack2.cpp</ins>

```cpp
#include “windows.h“
#include “tchar.h“
#define DEF_CMD L“C:\\Program Files\\Internet Explorer\\iexplore.exe“
#define DEF_ADDR L“http://www.naver.com“
#define DEF_DST_PROC L“notepad.exe“
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
    TCHAR szCmd[MAX_PATH] = {0, };
    TCHAR szPath[MAX_PATH] = {0, };
    TCHAR* p = NULL;
    STARTUPINFO s = {0, }; // 프로세스 시작 설정
    // 생성하는 프로세스의 정보를 얻기 위해 인자로 전달하는 구조체
    PROCESS_INFORMATION pi = {0, };
    si.cb = sizeof(STARTUPINFO);
    si.dwFlags = STARTF_USESHOWWINDOW;
    si.wShowWindow = SW_HIDE;
    switch(fdwReason)
    {
    case DLL_PROCESS_ATTACH :
        if(!GetModuleFileName(NULL, szPath, MAX_PATH))
            break;
        if(!(p = _tcsrchr(szPath, ’\\’)))
            break;
        if(_tcsicmp(p + 1, DEF_DST_PROC))
            break;
        wsprintf(szCmd, L“%s %s“, DEF_CMD, DEF_ADDR);
        if(!CreateProcess(NULL, (LPTSTR)(LPCTSTR)szCmd, NULL, NULL, FALSE, NORMAL_PRIORITY_CLASS, NULL, NULL, &si, &pi))
            break;
        if(pi.hProcess != NULL)
            CloseHandle(pi.hProcess);
        break;
    }
    return TRUE;
}
```

* notepad 프로세스만을 대상으로 하므로, 코드 내에서 프로세스 모듈명을 비교
* regedit.exe의 HKEY_LOCAL_MACHINE\SOFTWARE\MICROSOFT\Windows NT\Current Version\Windows에서, AppInit_DLLs 값을 myhack2.dll의 경로로, LoadAppInit_DLLs 값을 1로 변경한 뒤 시스템을 재부팅하면, 모든 프로세스에 DLL 인젝션 가능

<br/>

### 메시지 후킹(SetWindowsHookEx() API 함수) 방식
SetWindowsHookEx() API 함수를 이용하여 메시지 훅을 설치하면, OS에서 hook procedure를 담고 있는 DLL을 (창을 가진) 프로세스에게 강제로 인젝션

<br/><br/>

## DLL 이젝션
프로세스에 강제로 삽입한 DLL을 빼내는 기법. 대상 프로세스로 하여금 FreeLibrary() API를 호출하도록 만드는 것
* Windows Kernel Object에게는 ‘참조 카운트’가 있는데, LoadLibrary()는 참조 카운트를 1 증가시키고, FreeLibrary()는 참조 카운트를 1 감소시킴

<br/>

### 예시 #8: EjectDll.cpp

```cpp
#include “windows.h”
// 프로세스 리스트와 모듈 리스트를 구하는 데 사용.
// CreateToolhelp32Snapshot으로 프로세스 리스트에 대한 핸들을 얻어온 뒤,
// Process32First, Process32 Next로 프로세스 리스트를 조사할 수 있음.
// 또한 Module32First, Module32Next로 모듈 리스트 역시 조사 가능
#include ”tlhelp32.h”
#include ”tchar.h”
#define DEF_PROC_NAME (L”notepad.exe”)
#define DEF_DLL_NAME (L”myhack.dll”)
DWORD FindProcessID(LPCTSTR szProcessName)
{
    DWORD dwPID = 0xFFFFFFFF;
    HANDLE hSnapShot = INVALID_HANDLE_VALUE;
    // Process32First() 함수의 인자로 전달되어, 프로세스의 정보를 입력받음
    PROCESSENTRY32 pe;
    pe.dwSize = sizeof(PROCESSENTRY32);
    hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPALL, NULL);
    // 스냅샷을 찍는 순간의 프로세스 리스트를 가져옴
    Process32First(hSnapShot, &pe);
    do
    {
        if(!_tcsicmp(szProcessName, (LPCTSTR)pe.szExeFile))
        {
            dwPID = pe.th32ProcessID;
            break;
        }
    } while(Process32Next(hSnapShot, &pe)); // 실패 시 0 반환
    CloseHandle(hSnapShot);
    return dwPID;
}
// 흔히 프로세스의 권한을 얻기 위해 OpenProcess 함수에서 PROCESS_ALL_ACCESS를 사용.
// 그러나 현재 자신의 권한이 낮아서 프로세스 핸들을 열지 못하는 경우가 종종 있음.
// 이 경우 다음과 같은 함수들을 이용하여 자신의 권한을 높일 수 있음
// (스냅샷 등으로 오류 없이 핸들을 가져오기 위한 필수 과정)
BOOL SetPrivilege(LPCTSTR lpszPrivilege, BOOL bEnablePrivilege)
{
    // 접근 토큰(Access token. 로그온 수행 시 필요한 보안정보가 들어있는 객체.
    // 로그온할 때 만들어지며, 프로세스마다 하나의 토큰 사본이 필요한 것으로
    // 사용자의 권한을 식별하는 데 사용)의 권한 정보를 담는 구조체
    TOKEN_PRIVILEGES tp;
    HANDLE hToken;
    // 로컬 단일 식별자. 시스템 시동 시 운영 체제에 부여되는 유일한 식별자
    LUID luid;
    if(!OpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_PREVILEGES | TOKEN_QUERY, &hToken))
    {
        _tprintf(L”OpenProcessToken error : %u\n”, GetLastError());
        return FALSE;
    }
    // 로컬로 지정된 privilege name을 나타내기 위해 시스템에서 사용되는 LUID 검색
    if(!LookupPrivilegeValue(NULL, lpszPrivilege, &luid))
    {
        _tprintf(L”LookupPrivilegeValue error : %u\n”, GetLastError());
        return FALSE;
    }
    // Privileges 배열(LUID_AND_ATTRIBUTES 구조체로 이루어짐)의 크기
    tp.PrivilegeCount = 1;
    tp.Privileges[0].Luid = luid;
    if(bEnablePrivilege)
        tp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;
    else
        tp.Privileges[0].Attributes = 0;
    if(!AdjustTokenPrivileges(hToken, FALSE, &tp, sizeof(TOKEN_PRIVILEGES), (PTOKEN_PRIVILEGES)NULL, (PDWORD)NULL))
    {
        _tprintf(L”AdjustTokenPrivileges error : %u\n”, GetLastError());
        return FALSE;
    }
    if(GetLastError() == ERROR_NOT_ALL_ASSIGNED)
    {
        _tprintf(L”The token does not have the specified privilege.\n”);
        return FALSE;
    }
    return TRUE;}BOOL EjectDll(DWORD dwPID, LPCTSTR szDllName){
    BOOL bMore = FALSE, bFound = FALSE;
    HANDLE hSnapShot, hProcess, hThread;
    HMODULE hModule = NULL;
    // Module32First() 함수의 인자로 전달되어 모듈의 정보를 입력받음
    MODULEENTRY32 me = {sizeof(me)};
    LPTHREAD_START_ROUTINE pThreadProc;
    // PID로 지정한 프로세스의 모듈 정보를 스냅샷
    hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, dwPID);
    bMore = Module32First(hSnapShot, &me);
    // 더 이상 탐색할 모듈이 없을 때 반복구문 탈출
    for(; bMore; bMore = Module32Next(hSnapShot, &me))
    {
        if(!_tcsicmp((LPCTSTR)me.szModule, szDllName) || !_tcsicmp((LPCTSTR)me.szExePath, szDllName))
        {
            bFound = TRUE;
            break;
        }
    }
    if(!bFound)
    {
        CloseHandle(hSnapShot);
        return FALSE;
    }
    if(!(hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwPID)))
    {
        _tprintf(L”OpenProcess(%d) failed!!! [%d]\n”, dwPID, GetLastError());
        return FALSE;
    }
    hModule = GetModuleHandle(L”kernel32.dll”);
    pThreadProc = (LPTHREAD_START_ROUTINE)GetProcAddress(hModule, ”FreeLibrary”);
    hThread = CreateRemoteThread(hProcess, NULL, 0, pThreadProc, me.modBaseAddr, 0, NULL);
    WaitForSingleObject(hThread, INFINITE);
    CloseHandle(hThread);
    CloseHandle(hProcess);
    CloseHandle(hSnapShot);
    return TRUE;
}
int _tmain(int argc, TCHAR* argv[])
{
    DWORD dwPID = 0xFFFFFFFF;
    dwPID = FindProcessID(DEF_PROC_NAME);
    if(dwPID == 0xFFFFFFFF)
    {
        _tprintf(L”There is no %s process!\n”, DEF_PROC_NAME);
        return 1;
    }
    _tprintf(L”PID of \”%s\” is %d\n”, DEF_PROC_NAME, dwPID);
    if(!SetPrivilege(SE_DEBUG_NAME, TRUE))
        return 1;
    if(EjectDll(dwPID, DEF_DLL_NAME))
        _tprintf(L”EjectDll(%d, \”%s\”) success!!!\n”, dwPID, DEF_DLL_NAME);
    else
        _tprintf(L”EjectDll(%d, \”%s\”) failed!!!\n”, dwPID, DEF_DLL_NAME);
    return 0;
}
```

<br/>

### 특권(Privilege) 
로컬 컴퓨터 운영에 관련된 특별한 작업을 할 수 있는 권한. 사용자 계정, 그룹 계정에 개별적으로 부여됨으로써 NT(Windows 커널) 보안모델의 구성요소를 이룸. 특권은 개별 파일의 보안 설정인 보안 설정자보다 우선하며, 관리자만이 특권을 다룰 수 있고, 특권을 받은 사용자는 보안 설정자와 상관없이 모든 파일을 읽을 수 있음. 사용자가 NT 시스템에 로그온 할 때 시스템은 사용자의 특권 목록을 조사해 사용자의 액세스 토큰을 작성하며, 사용자가 특권 동작을 하려고 할 때 시스템이 사용자의 액세스 토큰을 검사해 해당 특권이 있는지 조사해보고, 특권이 없으면 작업을 거부. 또한 특권은 로컬 컴퓨터에만 적용된다는 점에서 지역적
* 특권은 개별적으로 이름을 가지고 있으며 모두 문자열로 되어있음(Winnt.h 참고). 소유권 가져오기 특권, 디바이스 드라이버 설치가 가능한 특권, 백업 특권 등의 종류가 있음. 특권의 이름은 문자열로 정의되어 있으나, 시스템이 특권들을 구분할 때는 LUID라는 특별한 값을 사용. 특권은 로컬 컴퓨터 범위 내에서만 인정되기에, LUID는 시스템마다 다르며 심지어 컴퓨터를 부팅할 때마다 달라짐. 다음 두 함수는 특권의 이름과 LUID를 상호 변환시켜줌
  * LookupPrivilegeName : LUID ⇒ Name
  * LookupPrivilegeValue : Name ⇒ LUID
* 시스템을 재부팅할 때는 ExitWindowsEx라는 함수를 사용하는데, 시스템을 재부팅하기 위해서는 반드시 재부팅 특권이 주어져야 함(관리자로 로그온 하였더라도 이 특권은 디폴트로 주어지지 않음). 관리자는 스스로에게 재부팅 특권을 부여할 수 있으므로 먼저 자신의 액세스 토큰에게 재부팅 특권을 부여한 후, ExitWindowsEx 함수를 호출하면 됨

#### <ins>TOKEN_PRIVILEGES 구조체</ins>
액세스 토큰에 포함되는 특권의 목록은 TOKEN_PRIVILEGES 구조체로 표현됨
* PrivilegeCount : 토큰에 포함되는 특권의 개수
* Privileges : LUID_AND_ATTRIBUTES 구조체의 배열
  * Luid : 특권의 LUID를 나타내는 LUID_AND_ATTRIBUTES 멤버
  * Attribute : SE_PRIVILEGE_ENABLED 플래그 설정 시 특권이 부여됨

#### <ins>AdjustTokenPrivileges() 함수</ins>
특권 목록 작성 후 프로세스의 액세스 토큰에 특권을 부여할 땐 AdjustTokenPrivileges() 함수를 사용
* TokenHandle : 대상 액세스 토큰
* DisableAllPrivilege : TRUE이면 해당 토큰의 모든 특권이 취소됨. FALSE라면 NewState가 지정하는 특권 목록이 액세스 토큰에 부여됨
* 토큰이 지정된 특권을 갖지 못하였을 시, ERROR_NOT_ALL_ASSIGNED 에러 반환

<br/>

* PE 파일에서 직접 임포트하는 DLL의 경우, 프로세스 실행 중에 임의로 이젝션 불가

<br/><br/>

## PE 패치를 이용한 DLL 로딩
실행 파일을 직접 수정하여 DLL을 강제로 로딩
* urlmon.dll에서 제공하는 URLDownloadToFile() API 함수는 wininet.dll의 InternetOpen(), InternetOpenUrl(), InternetReadFile() API 함수만으로도 구현 가능

<br/>

### 예시 #9: DropFile()
다운받은 index.html 파일을 TextView_Patch.exe 프로세스에 드롭

```cpp
BOOL CALLBACK EnumWindowProc(HWND hWnd, LPARAM lParam)
{
    DWORD dwPID = 0;
    GetWindowThreadProcessId(hWnd, &dwPID);
    if(dwPID == (DWORD)lParam)
    {
        g_hWnd = hWnd;
        return FALSE; // EnumWindows 함수 중단
    }
    return TRUE;
}
HWND GetWindowHandleFromPID(DWORD dwPID)
{
    // 화면 상의 모든 윈도우를 거쳐 각 윈도우를 대상으로 콜백 함수 호출.
    // 화면 상의 모든 윈도우를 거쳤거나, 콜백 함수가 FALSE를 반환했을 때 중단
    EnumWindows(EnumWindowsProc, dwPID);
    return g_hWnd;}BOOL DropFile(LPCTSTR wcsFile){
    HWND hWnd = NULL;
    DWORD dwBufSize = 0;
    BYTE* pBuf = NULL;
    DROPFILES* pDrop = NULL;
    char szFile[MAX_PATH] = {0, };
    HANDLE hMem = 0;
    WideCharToMultiByte(CP_ACP, 0, wcsFile, -1, szFile, MAX_PATH, NULL, NULL);
    dwBufSize = sizeof(DROPFILES) + strlen(szFile) + 1;
    if(!(hMem = GlobalAlloc(GMEM_ZEROINIT, dwBufSize)))
    {
        OutputDebufString(L“GlobalAlloc() failed!!!“);
        return FALSE;
    }
    pBuf = (LPBYTE)GlobalLock(hMem);
    pDrop = (DROPFILES*)pBuf;
    pDrop->pFiles = sizeof(DROPFILES);
    strcpy_s((char*)(pBuf + sizeof(DROPFILES)), strlen(szFile) + 1, szFile);
    GlobalUnlock(hMem);
    if(!(hWnd = GetWindowHandleFromPID(GetCurrentProcessId())))
    {
        OutputDebufString(L“GetWndHandleFromPID() failed!!!“);
        return FALSE;
    }
    PostMessage(hWnd, WM_DROPFILES, (WPARAM)pBuf, NULL);
    return TRUE;
}
```

* WM_DROPFILES 메시지와 함께 전달되는 드롭 파일 정보는 DROPFILES 구조체와 드롭 파일 목록으로 이루어짐. 이때, DROPFILES의 pFile 멤버는 현 구조체의 주소값으로부터 드롭 파일 목록까지의 offset을 나타냄
* GlobalAlloc() : 16bit Windows 환경에서는 힙 메모리가 Global Heap과 Local Heap으로 나뉘어져 있었지만, 32bit Windows에선 그렇지 않으므로, 자신의 프로세스 영역의 힙에 그냥 할당하며 다른 프로세스와 힙 메모리 공유(‘Global’) 불가. 그러나 클립보드에서 사용하는 API 함수들이 GlobalAlloc이나 LocalAlloc 함수를 이용해서 할당한 메모리 주소를 요구하는 경우가 있으므로, 아직도 사용되고 있음
* GlobalLock() : GMEM_MOVEABLE 설정 시 시작 주소가 변경될 수 있는 메모리를 사용하므로, GlobalAlloc()은 주소값이 아닌 핸들값을 반환. 이때 GlobalLock 함수는 이러한 핸들값을 포인터로 변환시켜줌

<br/>

* myhack.dll의 익스포트 함수 dummy() : 아무런 기능이 없으나 익스포트하는 이유는, myhack3.dll을 TextView.exe의 임포트 테이블에 추가하도록 하는 형식적인 완전성을 제공하기 위한 것(최소한 하나 이상의 익스포트 함수)

<br/>

### 패치 사전조사
TextView.exe의 IDT 주소를 확인했을 때, IDT 끝 부분에 다른 데이터가 존재하기 때문에, myhack3.dll을 위한 IID 구조체를 덧붙일 공간이 없음을 알 수 있음

<br/>

### IDT 이동
IDT 전체를 다른 넓은 위치로 옮긴 후 새로운 IID를 추가해야 함. .rdata 섹션의 파일 크기는 2E00이지만 메모리상에서의 크기가 2C56이므로, 실제 프로그램에서 매핑되는 데이터의 크기는 2C56 뿐임. 나머지 사용하지 않는 1AA 크기의 영역에 IDT를 재구성하는 것은 문제되지 않음
* IMPORT Table의 RVA 값 변경 : IMAGE_OPTIONAL_HEADER의 IMPORT Table 구조체 멤버에서, RVA 값을 새로운 IDT의 RVA 값으로, Size 값은 기존의 64에 14를 더한 값인 78로 변경
* BOUND IMPORT TABLE 제거 : myhack3.dll을 정상적으로 임포트하기 위해선 BOUND IMPORT TABLE에도 정보를 추가해야 하지만, 다행히 이 테이블은 옵션으로서 반드시 존재할 필요가 없기 때문에 작업의 편의를 위해 제거(0으로 변경)
* 새로운 IDT 생성 : 기존 IDT를 전부 복사하여 새로운 위치에 덮어쓰기. myhack3.dll을 위한 IID를 구성하여 새로 생성된 IDT의 끝에 추가
* Name, INT, IAT 세팅 : 작업 편의에 따라 빈 공간 선택. IAT 값은 INT와 같은 값을 가져도, 다른 값을 가져도 좋음
* IAT 섹션의 Characteristics 변경 : IAT는 PE 로더에 의해 메모리에 로드될 때, 실제 함수 주소로 덮어써지기 때문에 해당 섹션은 반드시 WRITE 속성을 가지고 있어야 함(IMAGE_SCN_MEM_WRITE(80000000))
  * Data Directory 배열 중 ‘IMPORT Address Table’에 명시된 주소영역 내에 IAT가 존재한다면, 그 섹션에는 쓰기 속성이 없어도 됨
 
<br/><br/>

## Code 인젝션
상대방 프로세스에 독립 실행 코드를 삽입한 후 실행하는 기법

<br/>

<p align="center">
 <img src="https://github.com/mollose/Security/assets/57161613/b9311f3e-710f-4765-9150-0d709888745b" width="500">
</p><br/>

* 인젝션 대상이 되는 target.exe 프로세스에 코드와 데이터를 삽입. 코드의 형식은 스레드 프로시저 형식으로, 코드에서 사용되는 데이터는 스레드의 파라미터로 전달

<br/>

* 코드 인젝션은 DLL 인젝션에 비해 고려해야 할 사항이 더 많으나, 메모리를 조금만 차지하고 흔적을 쉽게 남기지 않으며, 별도의 DLL 파일이 필요 없다는 장점이 있음

<br/>

### 예시 #10: ThreadProc()
```cpp
typedef struct _THREAD_PARAM
{
    FARPROC pFunc[2]; // FARPROC : typedef int(FAR WINAPI* FARPROC)();
    char szBuf[4][128];
} THREAD_PARAM, *PTHREAD_PARAM;
typedef HMODULE(WINAPI* PFLOADLIBRARYA)(LPCSTR lpLibFileName);
typedef FARPROC(WINAPI* PFGETPROCADDRESS)(HMODULE hModule, LPCSTR lpProcName);
typedef int(WINAPI* PFMESSAGEBOXA)(HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType);
DWORD WINAPI ThreadProc(LPVOID lParam)
{
    PTHREAD_PARAM pParam = (PTHREAD_PARAM)lParam;
    HMODULE hMod = NULL;
    FARPROC pFunc = NULL;
    hMod = ((PFLOADLIBRARYA)pParam->pFunc[0])(pParam->szBuf[0]);
    pFunc = (FARPROC)((PFGETPROCADDRESS)pParam->pFunc[1])(hMod, pParam->szBuf[1]);
    ((PFMESSAGEBOXA)pFunc)(NULL, pParam->szBuf[2], pParam->szBuf[3], MB_OK);
    return 0;
}
```

* ThreadProc() 함수는 API를 직접 호출하지 않으며, 또한 문자열도 직접 정의해서 사용하지 않음. 전부 스레드 파라미터로 넘어온 THREAD_PARAM 구조체에서 가져다 사용하고 있는데, 그 이유는 일반적인 프로그램의 방식대로 MessageBoxA() 함수를 호출한다면 정상적으로 실행되지 않으며, 이는 코드에서 사용되는 데이터가 상대방 프로세스에 없기 때문이라는 것. 따라서 코드와 함께, 문자열과 API 함수의 주소를 같이 인젝션하여 참조가 정확히 이루어지도록 함
* 디버거로 살펴볼 때, 모든 중요한 데이터는 스레드 파라미터인 lParam([EBP + 8])으로 받아서 사용한다는 것을 알 수 있으며, 일반 프로그램의, 하드코딩된 주소의 데이터를 직접 참조하는 방식과는 차이가 있음

<br/>

* 상대방 프로세스에 data와 code를 각각 메모리 할당하고 인젝션 하는데, 함수를 대상 프로세스 메모리에 써넣을 시 함수의 크기를 인자로 전달해야 함. 이 경우 인젝션할 함수 바로 다음에 오는 함수의 주소값에서 인젝션할 함수의 주소값을 빼줌으로써 함수의 크기를 구할 수 있음

<br/>

* ThreadProc 함수 데이터를 인젝션할 때, VirtualAllocEx() 함수의 flProtect 파라미터를 PAGE_EXECUTE_READWRITE로 설정. 해당 페이지는 읽고 쓰는 것 이외에도 실행까지 가능

<br/><br/>

## 어셈블리 언어를 이용한 Code 인젝션
* OllyDbg의 New origin here은 단순히 EIP만 바꿔버리는 것이므로, 직접 디버깅을 해서 그 주소로 가는 것과는 다름. 따라서 레지스터와 스택의 내용은 바뀌지 않음

<br/>

### 예시 #11: CodeInjection2.cpp

```cpp
typedef struct _THREAD_PARAM
{
    FARPROC pFunc[2];
} THREAD_PARAM, *PTHREAD_PARAM;
// OllyDbg에서 어셈블리 코드로 작성한 뒤 Copy-To file로 가져온 ThreadProc() 함수
BYTE g_InjectionCode[] = {0x55, 0x8B, 0xEC, 0x8B, 0x75 … };
/*
55		PUSH EBP
8BEC		MOV EBP, ESP
; 파라미터로 전달된 THREAD_PARAM 구조체를 가져옴
8B75 08		MOV ESI, DWORD PTR SS:[EBP + 8]
68 6C6C0000	PUSH 6C6C ; “\0\0ll“
68 33322E64	PUSH 642E3233 ; “d.23“
68 75736572	PUSH 72657375 ; “resu“
54		PUSH ESP
FF16		CALL DWORD PTR DS:[ESI] ; LoadLibraryA() 호출
68 6F784100	PUSH 41786F ; “\0Axo“
68 61676542	PUSH 42656761 ; “Bega“
68 4D657373	PUSH 7373654D ; “sseM“
54		PUSH ESP
50		PUSH EAX
FF56 04		CALL DWORD PTR DS:[ESI + 4] ; GetProcAddress() 호출
6A 00		PUSH 0
; 리턴 주소를 PUSH하고 해당 주소로 점프하는 CALL 명령어의 특성을 활용하여,
; 함수 파라미터인 문자열 주소를 스택에 넣고 그 다음 코드 명령어로 이동
E8 0C000000	CALL 0040103F
ASCII		“ReverseCore“
E8 14000000	CALL 00401058
ASCII		“www.reversecore.com“
6A 00		PUSH 0
FFD0		CALL EAX ; MessageBoxA() 호출
33C0		XOR EAX, EAX
8BE5		MOV ESP, EBP
5D		POP EBP
C3		RETN
*/
BOOL InjectCode(DWORD dwPID)
{
    /* 생략 : pRemoteBuf[0] 할당 후 함수 포인터 파라미터 구조체를 써넣음 */
    pRemoteBuf[1] = VirtualAllocEx(hProcess, NULL, sizeof(g_InjectionCode), MEM_COMMIT, PAGE_EXECUTE_READWRITE);
    WriteProcessMemory(hProcess, pRemoteBuf[1], (LPVOID)&g_InjectionCode, sizeof(g_InjectionCode), NULL);
    hThread = CreateRemoteThread(hProcess, NULL, 0, (LPTHREAD_START_ROUTINE)pRemoteBuf[1], pRemotrBuf[0], 0, NULL);
    /* 생략 */
}
```

<br/><br/>

## API 후킹
Win32 API를 후킹하는 기술
* 후킹 : 정보를 가로채고 실행 흐름을 변경하며, 원래와는 다른 기능을 제공하게 하는 기술
* API : Windows OS에서는 사용자 애플리케이션이 시스템 자원(프로세스, 스레드, 메모리, 파일, 네트워크, 비디오, 사운드 등)을 사용하고자 할 때 직접 접근할 수 있는 방법이 없으며, 애플리케이션은 Win32 API를 이용해 커널에게 자원을 요청해야 함

<br/>

<p align="center">
 <img src="https://github.com/mollose/Security/assets/57161613/9290da43-66f4-46df-87ec-69471bd206f9" width="700">
</p><br/>

* 모든 프로세스에는 기본적으로 kernel32.dll이 로드되며, kernel32.dll은 ntdll.dll을 로딩(특정 시스템 프로세스(smss.exe)는 kernel32.dll을 로드하지 않음). ntdll.dll의 역할이 바로 유저 모드 애플리케이션의 코드에서 발생하는 시스템 자원에 대한 접근을 커널모드에 요청하는 것. 일반적인 시스템 자원을 이용하는 API는 kernel32.dll과 ntdll.dll을 타고 내려가다 결국 SYSENTER 명령을 통해 커널모드로 진입하게 됨

<br/>

* API 후킹의 이점 : API 호출 전후에 사용자의 훅 코드를 실행할 수 있으며, API에서 넘어온 파라미터 혹은 API 함수의 리턴값을 엿보거나 조작할 수 있고, API 호출 자체를 취소시키거나 사용자 코드로 실행 흐름 변경 가능

<br/>

<p align="center">
 <img src="https://github.com/mollose/Security/assets/57161613/4d27aed7-d1de-478d-95e8-23a0a2e327fd" width="700">
</p><br/>

### 테크 맵
API 후킹의 모든 기술적 범주를 포함

<br/>

<p align="center">
 <img src="https://github.com/mollose/Security/assets/57161613/cdb30e54-e80c-40b0-8a85-d50a51d2a8d3" width="700">
</p><br/>

#### <ins>Method</ins>
API 후킹 방식은 작업 대상이 ‘파일’(static)인지, 혹은 프로세스 메모리인지(dynamic)에 따라 달라짐. static 방식의 API 후킹은 프로그램 실행 전 최초 한 번만 후킹되며 Unhook이 불가능하고, 매우 특수한 상황에서만 사용됨 
