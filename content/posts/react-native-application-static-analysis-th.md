+++ 
authors = ["bongtrop",]
title = "React Native Application Static Analysis [Thai]"
date = "2021-01-12"
description = "วิธีการแกะการทำงานของ Application ที่ถูกพัฒนามาจาก React Native"
+++

![banner](https://i.imgur.com/cCqu6la.png)

> For english version, please navigate to [https://suam.wtf/posts/react-native-application-static-analysis-th/](https://suam.wtf/posts/react-native-application-static-analysis-th/).

ในปี 2020 ที่ผ่านมา ผมทำงาน pentest เจอแอปที่ถูกพัฒนาด้วย cross-platform framework ต่าง ๆ เยอะขึ้นมาก เช่น [Xamarin](https://dotnet.microsoft.com/apps/xamarin), [React Native](https://reactnative.dev/), หรือ [Flutter](https://flutter.dev/) และผมคิดว่าในปีหลัง ๆ ก็น่าจะเจอบ่อยขึ้นไปอีก อีกทั้งในการทำ reverse engineering หรือการทำ static analysis ของแอปใน framework ต่าง ๆ ก็จะมีวิธีการที่ไม่เหมือนกันเลย ยิ่งเป็น framework ที่ถูกสร้างขึ้นใหม่จะไม่มี material หรือ tool ที่ช่วยในการทำ analysis อยู่เลย ทำให้การทำ static analysis ทำได้ยากมาก รวมไปถึงการ bypass logic หรือ protection ต่าง ๆ ที่ต้องทำการ patch แอป ก็จะแทบเป็นไปไม่ได้เลยถ้าไม่เข้าใจ Internal ของ framework นั้น ๆ

เมื่อก่อน React Native เป็น framework ที่เจอบ่อยมาก และทำ reverse engineering ง่ายกว่าเพื่อน แต่ว่าทาง Facebook ได้เริ่มสร้าง JavaScript Engine ของตัวเองชื่อ [Hermes](https://hermesengine.dev/) ขึ้นมา และเริ่มที่จะนำมาใช้กับ React Native แล้ว ตอนนี้ถูก bundle มาพร้อมใน starter kit ต่าง ๆ แล้ว (แต่ยังไม่ได้ enable เป็น default) ทำให้ developer ที่ต้องการความเร็วแอปเริ่มหันมา enable กันละ โดย React Native แบบเดิมจะทำการฝัง JavaScript source code ที่ถูก minify แล้วมา ทำให้สามารถ beautify และก็อ่านได้เลย ez แต่แบบใหม่ที่ใช้ Hermes ตอน build JavaScript source code จะถูก compile เป็น Hermes bytecode ก่อนและนำมายัดลงแอป

ก็นั้นแหละครับ Factor ความ **ชิบหาย** ครบ ทั้งเป็น VM และมาใหม่ material และ tool ไม่มี ต้องไปอ่าน [Hermes Engine](https://github.com/facebook/hermes) source code ใน Github เพื่อจะได้รู้ว่าจะต้องอ่านอะไร อ่านตรงไหน และควรจะ patch ยังไง จริง ๆ แล้ว AOT snapshot ของ Flutter ก็ไม่ต่างกันมาก  และ Xamarin ก็เปลี่ยนวิธีใหม่เอา DLL ยัดเข้าไปใน Native Lib พวกนี้ก็ยังไม่ค่อยมีคนมาแกะซักเท่าไหร่ วันนี้เลยจะมาแชร์วิธีการทำ static analysis สำหรับ React Native แอปครับ

## React Native แบบไม่ Hermes

ถ้าใครเคยทำ reverse engineering แอปมือถือมาก่อนจะรู้เลยว่าแอปที่เขียนด้วย React Native มันแกะง่ายมาก จะแทบไม่ต่างจากการแฮกเว็บเท่าไหร่เลย โดยตัวอย่างของบทความนี้ผมจะใช้เป็นแอป Android นะครับ เพราะว่าเข้าถึงง่ายกว่า ส่วน iOS แทบจะไม่ต่างกับ Android เลยแค่ที่จัดเก็บ JavaScript ต่างกัน ผมจะแสดงวิธีการแกะและ patch จาก แอปตัวอย่างต่อไปนี้ครับ

> ทดลองทำตามได้นะครับ [ReactNativeReverseingLab.apk](https://github.com/bongtrop/react-native-reversing-lab/releases/download/v1.0/ReactNativeReverseingLab.apk)

เมื่อติดตั้งแอปหลังจากกดปุ่ม + ไปได้ 2 รอบตัวแอปจะแสดงข้อความ `Increase button has already been broken.` และไม่สามารถเพิ่ม `counter` ได้แล้ว ดังรูป

![React Native 1](https://i.imgur.com/ZDIRFSjl.png)

เราจะต้องเพิ่ม counter ไปให้ถึง 10 เพื่อชนะ ดังนั้นต้องทำการ patch แอป ขั้นตอนง่ายมากครับ ดังนี้

**(1)** ทำการ unzip ไฟล์ APK ของแอปเป้าหมายออกมาครับ (อาจจะใช้ apktool ก็ได้ครับ ผมจะสอนทำ manual ก่อนเพื่อจะได้ต่อยอดได้) ผมใช้คำสั่ง unzip ดังต่อไปนี้ครับ

```
(hack) bongtrop@bongtrop-pc:lab/ $ unzip ReactNativeReversingLab.apk -d ReactNativeReversingLab
Archive:  ReactNativeReverseingLab.apk
  inflating: ReactNativeReverseingLab/AndroidManifest.xml
  inflating: ReactNativeReverseingLab/META-INF/CERT.RSA
  inflating: ReactNativeReverseingLab/META-INF/CERT.SF
  inflating: ReactNativeReverseingLab/META-INF/MANIFEST.MF
[...]
```

**(2)** จากนั้นเข้าไปที่ directory `ReactNativeReversingLab/asserts` จะเจอไฟล์ `index.android.bundle` ดังนี้ครับ

```
(hack) bongtrop@bongtrop-pc:lab/ $ cd ReactNativeReverseingLab/assets
(hack) bongtrop@bongtrop-pc:assets/ $ ls -al
total 676
drwxrwxr-x 2 bongtrop bongtrop   4096 Jan 12 19:04 .
drwxrwxr-x 7 bongtrop bongtrop   4096 Jan 12 19:04 ..
-rw-rw-r-- 1 bongtrop bongtrop 680631 Dec 31  1979 index.android.bundle
(hack) bongtrop@bongtrop-pc:assets/ $ cat index.android.bundle
var __BUNDLE_START_TIME__=this.nativePerformanceNow?na[...]
```

**(3)** เมื่อลองเปิดไฟล์ดังกล่าวจะเห็นว่าถูกทำ minify มาทำให้อ่านยาก โดยปกติผมจะทำการ beautify มันก่อนอ่านครับ ผมใช้ tool ชื่อ `js-beautify` สามารถติดตั้งง่าย ๆ ด้วยคำสั่ง `npm install -g js-beautify` รันคำสั่งต่อไปนี้เพื่อ beautify ไฟล์ `index.android.bundle` ครับ

```
bongtrop@bongtrop-pc:assets/ $ js-beautify -r index.android.bundle 
beautified index.android.bundle
```

**(4)** เมื่อทำการ beautify แล้วเปิดไฟล์มาเพื่อแก้ไข JavaScript ได้เลยครับ ผมทำการ search ว่า `Increase button has already been broken` จะเจอว่าอยู่ที่บรรทัดประมาณ 29156 และใกล้ ๆ นั้นจะมี condition ว่า `2 != t.state.counter` อยู่แก้ให้เป็น 1 เลย ดังนี้ครับ

แก้

```js
2 != t.state.counter ? (t.setState({
    counter: t.state.counter + 1
}), 9 == t.state.counter && alert("You win !!")) : alert("Increase button has already been broken.")
```

ให้เป็น

```js
1 ? (t.setState({
    counter: t.state.counter + 1
}), 9 == t.state.counter && alert("You win !!")) : alert("Increase button has already been broken.")
```

**(5)** ทำการ save ไฟล์ และทำการย้อน directory ออกมา 1 ขั้น และทำการลบ signature เดิมของ APK ให้หมดครับ เพื่อจะได้ sign แต่ละไฟล์ใหม่โดยใช้ key ของเราเอง

```
bongtrop@bongtrop-pc:assets/ $ cd ..
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ rm META-INF/CERT.RSA
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ rm META-INF/CERT.SF
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ rm META-INF/MANIFEST.MF
```

**(6)** จากนั้นทำการ zip ไฟล์ทั้งหมดใหม่อีกรอบให้กลับเป็นไฟล์ APK ครับดังนี้

```
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ zip -r ReactNativeReverseingLab.patch.apk *
  adding: AndroidManifest.xml (deflated 66%)
  adding: assets/ (stored 0%)
  adding: assets/index.android.bundle (deflated 81%)
[...]
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ mv ReactNativeReverseingLab.patch.apk ..
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ cd ..
```

**(7)** ทำการ sign APK ใหม่อีกครั้ง

สำหรับใครไม่มี key ให้สร้างขึ้นมาก่อนนะครับ โดยใช้คำสั่งนี้ได้เลย

```
keytool -genkey -v -keystore helloworld.keystore -alias helloworld -keyalg RSA -keysize 2048 -validity 10000
```

ถ้ามีแล้วจะใช้ `apksigner` หรือ `jarsigner` sign APK ได้เลยนะครับ ตัวอย่างผมจะใช้ jarsigner ครับ

```
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore helloworld.keystore ReactNativeReverseingLab.patch.apk helloworld
Enter Passphrase for keystore:
   adding: META-INF/MANIFEST.MF
   adding: META-INF/HELLOWOR.SF
   adding: META-INF/HELLOWOR.RSA
[...]
```

**(8)** ทำการติดตั้งแอปด้วย APK ใหม่ (อย่าลืมลบอันเก่า) และทำการกด + จะสามารถกดไปเรื่อย ๆ จน alert `You win!!` โชว์ขึ้นมาดังรูปครับ

![React Native 2](https://i.imgur.com/9XQCpKnl.png)

สรุปคือถ้าเป็นแบบเดิมจะเป็น JavaScript source code ที่ถูก minify แล้ว เราสามารถเข้าไปอ่าน หรือแก้ไขได้เหมือนกับเขียน JavaScript ทั่วไปเลยครับ จะเห็นว่าไม่ยากเลย ไม่จำเป็นต้องใช้ tool อะไรมาก เห็นไหมครับว่าชีวิต pentester ง่ายมาก ๆ เมื่อเจอ React Native เราสามารถยัด JavaScript debugging tool เข้าไปใน source code และ debug ตัวแอปได้เหมือนเขียนเว็บก็ยังได้

## React Native แบบ Hermes

เข้าเรื่องที่อยากพูดได้ซักที ในการเปิดใช้งาน Hermes ให้ React Native แอป สามารถทำได้ง่ายมากเลยครับ เข้าไปแก้ไฟล์ `android/app/build.gradle` เปลี่ยน `enableHermes: false` เป็น `enableHermes: true`

> สำหรับรายละเอียดเพิ่มเติ่มดูได้จาก [https://reactnative.dev/docs/hermes](https://reactnative.dev/docs/hermes)

หลังจาก build ก็ได้ `index.android.bundle` ที่เป็น Hermes bytecode แล้วครับ ใน Ubuntu 20.04 คำสั่ง `file` จะสามารถ identify format นี้ได้แล้วครับ

```
$ file index.android.bundle
index.android.bundle: Hermes JavaScript bytecode, version 74
```

สำหรับใครที่อยากลองทำตามไปด้วยสามารถไปเล่นได้ที่ [lab.suam.wtf](https://lab.suam.wtf) ที่ข้อ [Hermes Reversing Lab](https://lab.suam.wtf/lab/suam-team/hermes-reversing-lab) ครับ

ถ้าลองตรวจสอบไฟล์ดูจะพบว่ามันอ่านไม่ออกครับ ถ้าเป็น .NET หรือ Java bytecode ทุกคนก็จะสามารถไปหา decompiler มา decompile ได้แล้วใช่ไหมครับ แต่สำหรับของใหม่อย่าง Hermes อย่าได้หวังครับ สิ่งที่เราสามารถทำได้คือใช้คำสั่ง `hbcdump` ที่ทาง Hermes Engine มีให้นะครับ สามารถโหลดได้จาก Github ของทาง [Hermes](https://github.com/facebook/hermes/releases) เลยครับ แต่ที่ต้องระวังคือแต่ละ release ของ Hermes จะรองรับ Hermes bytecode ใน version ที่ต่างกัน จะต้องไปดู Hermes bytecode version ที่ไฟล์ `/include/hermes/BCGen/HBC/BytecodeFileFormat.h` อยู่ที่ประมาณบรรทัดที่ 33 ครับ สำหรับ lab นี้เป็น version 74 ครับจะต้องใช้ [Hermes Release v0.6.0](https://github.com/facebook/hermes/releases/tag/v0.6.0) โหลดมาละใช้ได้เลยครับ จะอยู่ที่ไฟล์ `hbcdump` เลย การใช้งานก็ง่าย ๆ เลยครับ

```
$ ./hbcdump -objdump-disassemble index.android.bundle
hbcdump> dis

d0310a88a868dfb1ee21d12e9011725b1f716875:     file format HBC-74

Disassembly of section .text:

000000000002ca48 <_0>:
0002ca48:       30 44 08 00 00        DeclareGlobalVar        $0x000844
0002ca4d:       30 48 08 00 00        DeclareGlobalVar        $0x000848
[...]
hbcdump> quit
```

จากนั้นถ้าจะแก้อะไรก็ใช้ hex editor ไปแก้ในส่วนที่ต้องการแก้ครับ จะเห็นว่าที่ address 0x0002ca48 จะเหมือนกับ objdump เลย ก็แก้ไปทีละ byte

![React Native 3](https://i.imgur.com/7pWByDtl.png)

จะเห็นว่ามันทำยาก ถึงจะมี tool ของทาง Hermes มาให้ใช้ แต่ก็ไม่ได้แสดงข้อมูลที่ครบและดูง่าย การ patch ก็ต้องมาทำที่ Raw byte จาก hex editor อยู่ดี เหนื่อยสินะครับ ได้เวลาขายของ ผมและเพื่อน ๆ ที่เจอปัญหาการทำ reverse engineering กับ Hermes bytecode ได้ทำ tool ชื่อ [hbctool](https://github.com/bongtrop/hbctool) ขึ้นมาช่วยในการ analyze และ patch Hermes bytecode ครับ อารมจะคล้าย ๆ กับ smali ของ Dalvik (แต่กากกว่าเยอะ) สามารถติดตั้งได้ง่าย ๆ

```
pip install hbctool
```

มาลองไปทีละ step กันเลยครับ เหมือนเดิมครับ เมื่อติดตั้งแอปแล้ว หลังจากกดปุ่ม + ไปได้ 10 รอบตัวแอปจะแสดงข้อความ `Increase button has already been broken.` และไม่สามารถเพิ่ม `counter` ได้แล้ว ดังรูป

![React Native 1](https://i.imgur.com/ZDIRFSjl.png)

เราจะต้องเพิ่ม counter ไปให้ถึง 1337 เพื่อจะได้ flag ครับ เราเลยจะต้องทำการ patch Hermes bytecode เพื่อให้ bypass condition ที่ทำการกันไม่ให้เราเอา flag ครับ จริง ๆ แล้วทำได้หลายวิธีครับ 1 คือทำ static analysis ล้วน ๆ เพื่อหาวิธี decrypt flag ครับ แต่ในครั้งนี้จะสอนวิธีการ patch ตัว Hermes bytecode ครับ โดนขั้นตอนแรก ๆ จะคล้าย ๆ กับด้านบนครับ คือ

**(1)** ทำการ unzip ไฟล์ APK ของแอปเป้าหมายออกมาครับ (อาจจะใช้ apktool ก็ได้ครับ ผมจะสอนทำ manual ก่อนเพื่อจะได้ต่อยอดได้) ผมใช้คำสั่ง unzip ดังต่อไปนี้ครับ

```
(hack) bongtrop@bongtrop-pc:lab/ $ unzip HermesReversingLab.apk -d HermesReversingLab
Archive:  HermesReversingLab.apk
  inflating: HermesReversingLab/AndroidManifest.xml
  inflating: HermesReversingLab/META-INF/CERT.RSA
  inflating: HermesReversingLab/META-INF/CERT.SF
  inflating: HermesReversingLab/META-INF/MANIFEST.MF
[...]
```

**(2)** จากนั้นทำการ disassemble Hermes bytecode ไฟล์ โดยใช้ `hbctool` ครับ

```
(hack) bongtrop@bongtrop-pc:lab/ $ hbctool disasm HermesReversingLab/assets/index.android.bundle HermesReversingLabHASM
[*] Disassemble 'HermesReversingLab/assets/index.android.bundle' to 'HermesReversingLabHASM' path
[*] Hermes Bytecode [ Source Hash: d0310a88a868dfb1ee21d12e9011725b1f716875, HBC Version: 74 ]
[*] Done
```

**(3)** หลังจากที่ disassemble ออกมาเป็น HASM (ผมตั้งชื่อขึ้นมาเองครับ ย่อมาจาก Hermes Assembly) โดยภายใน `HermesReversingLabHASM` จะมีอยู่ 3 ไฟล์ดังนี้ครับ

- `metadata.json`: จะใช้เก็บข้อมูลสำคัญต่าง ๆ ของไฟล์ Hermes bytecode ที่ถูก disassemble มาครับ
- `instruction.hasm`: จะเป็นไฟล์ที่ใช้เก็บ function ต่าง ๆ ที่ถูก disassemble แล้ว เป็น format ที่ผมสร้างขึ้นมาเองครับ ใครมีแนวคิดต่าง ๆ สามารถเสนอได้ครับ สามารถ แก้ instruction ต่าง ๆ ของแอป ได้ที่ไฟล์นี้
- `string.json`: จะเป็น string ต่าง ๆ ที่ถูกใช้ในแอป เหมือนกันสามารถแก้ไข string ต่าง ๆ ได้ที่ไฟล์นี้ครับ

**(4)** ทำการแก้ไข instruction ของแอปผ่านไฟล์ `instruction.hasm` เลยครับ โดยผมได้ทำการแก้ไข ค่าที่ counter จะต้องเพิ่มไปถึงจาก 1336 ให้เป็น 1 แทนครับ จะได้ไม่ติด condition ที่จะเพิ่ม `counter` ได้ไม่เกิน 10 ครับ มีหลาย ๆ วิธีนะครับ อาจจะแก้ไข opcode จาก `JNotGreaterEqual` เป็น `jmp` แบบอื่น หรือเปลี่ยน address ที่จะ `jmp` ไปเป็น alert flag เลยก็ได้ครับ โดยครั้งนี้ผมจะใช้วิธีแรกครับ คือแก้เลข 1336 ที่ บรรทัด 182890 ในไฟล์ `instruction.hasm` ให้เป็น 1 ครับ

จาก

```
[...]
	LoadConstInt        	Reg8:1, Imm32:1336
	JNotGreaterEqual    	Addr8:43, Reg8:2, Reg8:1
[...]
```

ให้เป็น

```
[...]
	LoadConstInt        	Reg8:1, Imm32:1
	JNotGreaterEqual    	Addr8:43, Reg8:2, Reg8:1
[...]
```

**(5)** save ไฟล์และทำการ assemble HASM กลับไปเป็น HBC (Hermes Bytecode) ครับ โดยใช้ `hbctool` เหมือนเดิม

```
(hack) bongtrop@bongtrop-pc:lab/ $ hbctool asm HermesReversingLabHASM HermesReversingLab/assets/index.android.bundle 
[*] Assemble 'HermesReversingLabHASM' to 'HermesReversingLab/assets/index.android.bundle' path
[*] Hermes Bytecode [ Source Hash: d0310a88a868dfb1ee21d12e9011725b1f716875, HBC Version: 74 ]
[*] Done
```

**(6)** ทำการลบ signature ไฟล์ เดิมทิ้งและทำการ zip แอปกลับไปเป็น APK ครับ

```
(hack) bongtrop@bongtrop-pc:lab/ $ cd HermesReversingLab         
(hack) bongtrop@bongtrop-pc:HermesReversingLab/ $ rm META-INF/CERT.RSA
(hack) bongtrop@bongtrop-pc:HermesReversingLab/ $ rm META-INF/CERT.SF       
(hack) bongtrop@bongtrop-pc:HermesReversingLab/ $ rm META-INF/MANIFEST.MF
(hack) bongtrop@bongtrop-pc:HermesReversingLab/ $ zip -r HermesReversingLab.patch.apk *
  adding: AndroidManifest.xml (deflated 66%)              
  adding: assets/ (stored 0%)                           
  adding: assets/index.android.bundle (deflated 48%)
[...]
(hack) bongtrop@bongtrop-pc:HermesReversingLab/ $ mv HermesReversingLab.patch.apk ..
(hack) bongtrop@bongtrop-pc:HermesReversingLab/ $ cd ..
```

**(7)** sign ด้วยวิธีเดิมครับ

```
(hack) bongtrop@bongtrop-pc:lab/ $ jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore helloworld.keystore HermesReversingLab.patch.apk helloworld
Enter Passphrase for keystore:                                                
   adding: META-INF/MANIFEST.MF
   adding: META-INF/HELLOWOR.SF
   adding: META-INF/HELLOWOR.RSA
[...]
```

**(8)** เมื่อทำการติดตั้งและเปิดแอปก็กด + สองครั้งก็จะได้ flag แล้วครับโดยตัวอย่างการทำทั้งหมด ผมใส่ไว้เป็นไฟล์ gif ใน [hbctool](https://github.com/bongtrop/hbctool) แล้วครับ (ตัวอย่างใน gif จะไม่ได้ทำการ bundle และ unbundle manual นะครับ แต่การ patch จะเหมือนกัน)

![hbctool example](https://i.imgur.com/70MBQ2c.gif)

หรือใครอยากดูแบบชัด ๆ ดูเป็น [MP4](https://github.com/bongtrop/hbctool/raw/main/image/hbctool_example.mp4) ได้ครับ

> ผมทำ lab ขึ้นมาเพื่ออยากให้คนที่อ่านสามารถทำตามแต่ละขั้นตอนได้ และเห็นภาพครับ ถ้าใครได้ flag แล้วสามารถ ไป submit ได้ที่ [lab.suam.wtf](https://lab.suam.wtf/) ครับ

## สรุปนะครับ

ผมทำบทความนี้เพื่ออยากให้ มีคนเก่ง ๆ มาสนใจการทำ in-depth analysis ตัว framework ต่าง ๆ และส่งต่อให้ community เพิ่มมากขึ้นครับ เพราะเดี๋ยวนี้ tech มันไปไวมาก จนผมไม่สามารถจะ research ตามได้ทัน ไม่ทันจริง ๆ ยิ่งตอนยังเป็น pentester เรื่องแบบนี้ต้องทำนอกเวลางานเท่านั้นเลยครับ พูดง่าย ๆ คือช่วย pentester ตาดำ ๆ แบบผม ให้ใช้ชีวิตได้อย่างราบลื่นด้วย ด้วยการแบ่งปัญความรู้ด้วยเถอะครับ T^T

อย่างไรก็ตาม ผมหวังว่า tool ที่ผมเขียนและบทความนี้จะเป็นประโยชน์กับ pentester ทุกคนที่ต้องเจอกับเรื่องร้าย ๆ ที่นับวันการทดสอบระบบยิ่งยากขึ้นนะครับ

สรุปเรื่อง technique บ้างละกัน Hermes ก็เป็นอะไรที่จะต้องจับตามองนะครับ เพราะว่าการ enable มันแทบจะไม่มีผลเสียเลย แอปก็ไวขึ้นอย่างเห็นได้ชัด อีกทั้งยังเปิดง่ายด้วยนะ และในอนาคตผมมองว่าถ้า Hermes stable แล้ว React Native แอปทุกแอปน่าจะถูกบังคับเปลี่ยนไปใช้ Hermes ครับ และในอนาคตของอนาคตอีก Facebook น่าจะขยับไปเน้น JIT และ AOT มากขึ้น เหมือนกับที่ Flutter ทำ สำหรับ pentester **ความชิบหาย * 100** แน่นอนครับ