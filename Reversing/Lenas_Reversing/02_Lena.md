# Lena's Reversing for Newbies #02

### 1. 문제

- 문제: Keyfiling the reverseme + assembler 
- [파일다운로드](https://tuts4you.com/download/123/)



### 2. 파일실행

- \#01과 동일 파일

- reverseMe.exe를 실행하면 "Evaluation period out of date. Purchase new license" 메시지박스 출력

  <img src="img/01/01_messagebox.png">

-  \#01과 다르게 \#02에서는 keyfile을 만들어서 "You really did it! Congrats !!!" 메시지박스가 출력되도록 해야함

<img src="img/01/01_success_msg.png">



### 3. OllyDbg로 어셈블리코드 분석

- 상세 분석은 \#01에서 했으므로 생략



#### 1) Keyfile 생성, CreateFileA() 함수를 호출하여 파일 열기

- "Evaluation period out of date. Purchase new license" 메시지박스 출력 전 CreateFileA() 함수를 호출
  - FileName: "Keyfile.dat"
  - Mode: OPEN_EXISTING

<img src="img/02/createfile_before.png">

- reverseMe.exe와 동일 경로에 "Keyfile.dat" 파일을 생성(내용은 없는 채로 생성)

   <img src="img/02/create_keyfile.png">

- CreateFileA() 함수 호출 코드에 브레이크 포인트 설정한 후  한 후 CreateFileA() 함수 호출까지 실행 
- CreateFileA() 함수 호출 후 리턴값인 EAX 레지스터 값은 -0x1이 아닌 값이므로 성공
  - EAX 레지스터 값: 0x00000044
  - 리턴값을 비교하는 코드와 비교 결과에 따라 분기하는 코드에 따라 reverseM.0040109A로 분기

<img src="img/02/createfile_called.png">



#### 2)  reverseM.0040109A로 분기, ReadFile() 함수를 호출하여 파일 내용 읽기

- ReadFile() 함수 호출 코드에 브레이크 포인트 설정한 후  한 후 ReadFile() 함수 호출까지 실행
- ReadFile() 함수 호출 후 리턴값인 EAX 레지스터 값은 0이 아닌 값이므로 함수 호출 성공
  - EAX 레지스터 값: 0x00000001
  - 리턴값을 비교하는 코드와 비교 결과에 따라 분기하는 코드에 따라 reverseM.004010B4로 분기

<img src="img/02/readfile_called.png">



#### 3) reverseM.004010B4로 분기하여 코드 실행

- reverseM.004010B4로 분기하여 어셈블리 코드를 차례대로 실행
- EBX, ESI 레지스터를 0으로 초기화(XOR연산: 비트값이 같으면 0, 다르면 1)
  - XOR EBX,EBX
  - XOR ESI,ESI
- CMP DWORD PTR DS:[0x402173],0x10
  - DS:[0x402173]가 가리키는 값과 0x10(16(dec))와 비교
-  JL SHORT reverseM.004010F7
  - CMP의 두 오퍼랜드 중 앞 오퍼랜드가 0x10보다 작으면 reverseM.004010F7로 분기
  - JL: Jump if Less
- DS:[0x402173]: 0x00000000이므로 비교문 결과 reverseM.004010F7로 분기하게 됨
  -  "Keyfile is not valid. Sorry" 메시지박스가 출력(실패)

<img src="img/01/01_key_invalid_msgbox.png">

- DS:[0x402173]는 ReadFile()  함수의 PByteRead가 가리키는 값임을 알 수 있음

<img src="img/02/readfile_fail.png">

- ReadFile() 함수의 pByteRead 파라미터의 의미

  - [ReadFile함수 msdn 설명](https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-readfile)
  - pByteRead는 lpNumberOfBytesRead 파라미터에 해당
    - 읽어드린 byte 수

- 1) 에서 생성한 Keyfile.dat은 내용이 없는 빈 파일

  - ReadFile()함수를 실행 할 때 pByteRead가 가리키는 값은 0이 됨

- Keyfile.dat파일 내용 길이(byte)를 유추

  - 힌트: CMP DWORD PTR DS:[0x402173],0x10, JL SHORT reverseM.004010F7
  - 최소 16 byte 이상
    -  JL로 되어 있기 때문
  - 최대 70byte 이하
    - ReadFile() 함수의 BytesToRead(46(hex, 70(dec))


#### 4) Keyfile.dat에 16byte 이상 내용을 입력 한 후 어셈블리 코드 다시 분석

- Keyfile.dat을 열어 16byte 입력 후 저장

<img src="img/02/keyfile_modify.png">

- 004010B8   CMP DWORD PTR DS:[0x402173],0x10부분에 브레이크 포인트를 설정
- OllyDbg를 재시작하여 브레이크 포인트까지 코드 실행
- reverseM.004010F7로 분기하지 않음

<img src="img/02/readfile_called_again.png">



#### 5) 반복문 어셈블리 코드 실행

- reverseM.004010F7로 분기하지 않고 코드를 차례대로 실행하면 반복문 어셈블리 코드가 나옴
  - 반복문: reverseM.004010C1 ~ reverseM.004010D1
- 반복문 실행이 끝나고 실행되는 reverseM.004010D3에 브레이크 포인트를 설정하고 브레이크 포인트까지 실행
- CMP ESI,0x8, JL SHORT reverseM.004010F7
  - ESI 레지스터 값을 0x8과 비교, 비교결과 ESI 레지스터 값이  0x8보다 작아 reverseM.004010F7로 분기 될 예정
    -   reverseM.004010F7로 분기 시 "Keyfile is not valid. Sorry" 메시지박스 출력(실패), 프로그램 종료
  - ESI, EBX 레지스터는 3) 에서 XOR연산으로 0x0으로 초기화 이후, 반복문 안에서  INC 연산을 통해 1씩 증가 하고 있음
    - 반복문이 종료된 후 ESI 레지스터 값은 0x0,  EBX 레지스터 값은 0x10

<img src="img/02/cmp_esi.png">

- 반복문 내의 어셈블리 코드 분석이 필요
  - 반복문 내 비교문, 분기문의 의미 파악 필요
  - ESI 레지스터, EBX 레지스터를 증가하는 INC 연산 코드 실행이 언제되는지 분석 필요
- 반복문이 시작되는 reverseM.004010C1에 브레이크 포인트를 설정 한 후 OllyDbg 재시작 후 브레이크 포인트까지 실행



