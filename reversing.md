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
 <img src="https://github.com/mollose/Security/assets/57161613/6e9abfe1-4f64-4d45-81cb-1479903e92f3" width="700">
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
 <img src="https://github.com/mollose/Security/assets/57161613/841094b1-7f07-478b-b8b5-99e5733868eb" width="700">
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
 <img src="https://github.com/mollose/Security/assets/57161613/a8fb0b89-6798-4467-b78b-1b3067a40491" width="700">
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
