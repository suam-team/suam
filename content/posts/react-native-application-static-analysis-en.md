+++ 
authors = ["bongtrop",]
title = "React Native Application Static Analysis [English]"
date = "2021-01-20"
description = "The reverse engineer of react native application"
+++

![banner](https://i.imgur.com/cCqu6la.png)

> สำหรับเวอร์ชันภาษาไทย อ่านได้ที่ [https://suam.wtf/posts/react-native-application-static-analysis-th/](https://suam.wtf/posts/react-native-application-static-analysis-th/)

In the previous year (2020), I performed penetration testing on many mobile applications, and many of them are implemented with cross-platform frameworks e.g., [Xamarin](https://dotnet.microsoft.com/apps/xamarin), [React Native](https://reactnative.dev/), or [Flutter](https://flutter.dev/). I think in the upcoming years, an increasing amount of mobile applications will be developed with these frameworks.

For each framework, different reverse engineering and static analysis methods are used. Moreover, for the newer frameworks, reverse engineering materials and tools are lacking. As a result, it is very hard to perform static analysis and patch these applications if you don't understand the internals of these frameworks.

React Native application was one of the most common types found, and they had been the easiest to perform reverse engineering. However, Facebook has implemented their own JavaScript Engine named [Hermes](https://hermesengine.dev/). Some React Native applications have already migrated to the Hermes engine. Additionally, the Hermes engine has also been bundled in many React Native starter kits, so enabling it is very easy.

Without the Hermes engine, the React Native JavaScript source code is just minified before being stored in the application. Thus, to do reverse engineering, only beautifying is needed before reading or patching it. But with the Hermes engine, JavaScript source code is compiled to the Hermes bytecode before being stored in the application.

Hermes is a new custom JavaScript engine. Therefore, the only way to understand and know how to patch it is by reading the [Hermes engine source code](https://github.com/facebook/hermes) from Github. This is not very different from the AOT snapshot of Flutter, or .NET DLL bundled in the native lib of Xamarin. When I found them in pentest projects, they always made me frustrated.

## React Native without Hermes engine

If you used to perform reverse engineering on the mobile application, you must know that it is very easy. It is hardly different from web hacking. In the following examples, the Android application will be used because it is easier to create and set up labs. Besides, for React Native, Android and iOS are hardly different.

> You can follow this tutorial using this application ([ReactNativeReverseingLab.apk](https://github.com/bongtrop/react-native-reversing-lab/releases/download/v1.0/ReactNativeReverseingLab.apk)).

After installing and launching the application, tap on the + button twice. The `Increase button has already been broken.` message will be shown and you will not be able to increase the `counter` anymore.

![React Native 1](https://i.imgur.com/ZDIRFSjl.png)

The goal of this lab is to increase the counter to 10. Therefore, application patching. It is very simple as follows:

**(1)**  Unzip the APK file (you can also use `apktool` in this step).

```
(hack) bongtrop@bongtrop-pc:lab/ $ unzip ReactNativeReversingLab.apk -d ReactNativeReversingLab
Archive:  ReactNativeReverseingLab.apk
  inflating: ReactNativeReverseingLab/AndroidManifest.xml
  inflating: ReactNativeReverseingLab/META-INF/CERT.RSA
  inflating: ReactNativeReverseingLab/META-INF/CERT.SF
  inflating: ReactNativeReverseingLab/META-INF/MANIFEST.MF
[...]
```

**(2)** Navigate to `ReactNativeReversingLab/asserts`. The `index.android.bundle` will be found.

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

**(3)**  The JavaScript source code in `index.android.bundle` file is minified. You can beautify it by using the `js-beautify` tool. It can be installed by executing `npm install -g js-beautify` command.

```
bongtrop@bongtrop-pc:assets/ $ js-beautify -r index.android.bundle 
beautified index.android.bundle
```

**(4)** After beautifying the JavaScript source code, search `Increase button has already been broken` text (line 29156). Then, change the `2 != t.state.counter` condition to `1`.

From:

```js
2 != t.state.counter ? (t.setState({
    counter: t.state.counter + 1
}), 9 == t.state.counter && alert("You win !!")) : alert("Increase button has already been broken.")
```

To:

```js
1 ? (t.setState({
    counter: t.state.counter + 1
}), 9 == t.state.counter && alert("You win !!")) : alert("Increase button has already been broken.")
```

**(5)** Save `index.android.bundle`. Next, navigate to the parent directory and remove all old signatures in the `META-INF` directory.

```
bongtrop@bongtrop-pc:assets/ $ cd ..
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ rm META-INF/CERT.RSA
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ rm META-INF/CERT.SF
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ rm META-INF/MANIFEST.MF
```

**(6)** Zip all files into an APK.

```
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ zip -r ReactNativeReverseingLab.patch.apk *
  adding: AndroidManifest.xml (deflated 66%)
  adding: assets/ (stored 0%)
  adding: assets/index.android.bundle (deflated 81%)
[...]
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ mv ReactNativeReverseingLab.patch.apk ..
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ cd ..
```

**(7)** Sign the APK.

If you don't have a key, generate it with the following command.

```
keytool -genkey -v -keystore helloworld.keystore -alias helloworld -keyalg RSA -keysize 2048 -validity 10000
```

Then, sign the APK with `apksigner` or `jarsigner`.

```
bongtrop@bongtrop-pc:ReactNativeReverseingLab/ $ jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore helloworld.keystore ReactNativeReverseingLab.patch.apk helloworld
Enter Passphrase for keystore:
   adding: META-INF/MANIFEST.MF
   adding: META-INF/HELLOWOR.SF
   adding: META-INF/HELLOWOR.RSA
[...]
```

**(8)** Finally, install the new APK and tap on + button. The `You win!!` alert will be pop up as shown below.

![React Native 2](https://i.imgur.com/9XQCpKnl.png)

In a nutshell, without Hermes, you can beautify the JavaScript source code then easily analyze and patch it. As a result, I really love to do reverse engineering on React Native without Hermes. We can also inject a JavaScript debugging tool into the application and debug it with the browser's dev tools.

## React Native with Hermes

We are finally reaching the main point. To enable the Hermes engine for React Native application, you can easily edit the `android/app/build.gradle` file to change `enableHermes: false` to `enableHermes: true`.

> For more information, please visit: [https://reactnative.dev/docs/hermes](https://reactnative.dev/docs/hermes)

After building the application, the `index.android.bundle` in Hermes bytecode format will be created. In Ubuntu 20.04, the `file` command can identify this format.

```
$ file index.android.bundle
index.android.bundle: Hermes JavaScript bytecode, version 74
```

If you want to perform the reverse engineering on the Hermes bytecode by yourself, you can test in our lab ([Hermes Reversing Lab](https://lab.suam.wtf/lab/suam-team/hermes-reversing-lab)).

By investigating the `index.android.bundle` file, you will discover that it cannot be read. If it is .NET or Java bytecodes, you can use the decompiler tool from the internet to decompile it. But, for a new engine like Hermes, there is no hope.

A built-in tool like `hbcdump` can help you analyze the application. You can download it from [Hermes Github repo](https://github.com/facebook/hermes/releases). Please note that you must use the correct version of Hermes bytecode to disassemble the application. The version of Hermes bytecode can be found in `/include/hermes/BCGen/HBC/BytecodeFileFormat.h` file around line 33. For the [Hermes Reversing Lab](https://lab.suam.wtf/lab/suam-team/hermes-reversing-lab), it was built with Hermes bytecode version 74. Please use `hbcdump` from [Hermes Release v0.6.0](https://github.com/facebook/hermes/releases/tag/v0.6.0). After downloading, execute it as follows:

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

Then, if you want to patch it, use your favorite hex editor tool to modify the `index.android.bundle` file directly.

![React Native 3](https://i.imgur.com/7pWByDtl.png)

It can be seen that it is still hard even when the Hermes team provides the `hbctool` for us. The `hbctool` doesn't show all the important information in the assembly. And to patch the application, editing raw bytes with a hex editor is still required.

Therefore, I created a tool named [hbctool](https://github.com/bongtrop/hbctool) to help analyze and patch Hermes bytecode from React Native application. To install it, execute the following command.

```
pip install hbctool
```

Let's do it together. After downloading and installing the lab's APK from [Hermes Reversing Lab](https://lab.suam.wtf/lab/suam-team/hermes-reversing-lab), open it and tap the + button 10 times. The `Increase button has already been broken.` message will be shown and `counter` will not increase anymore.

![React Native 1](https://i.imgur.com/ZDIRFSjl.png)

To get a flag, we have to increase the counter to 1337. Thus, the Hermes bytecode must be patched to bypass the condition checking, or you can also perform reverse engineering on the encrypt function and reconstruct the decrypt function in order to get the flag. In the following steps, I will only show you the Hermes bytecode patching method.

**(1)** Unzip the APK file (you can also use `apktool` in this step).

```
(hack) bongtrop@bongtrop-pc:lab/ $ unzip HermesReversingLab.apk -d HermesReversingLab
Archive:  HermesReversingLab.apk
  inflating: HermesReversingLab/AndroidManifest.xml
  inflating: HermesReversingLab/META-INF/CERT.RSA
  inflating: HermesReversingLab/META-INF/CERT.SF
  inflating: HermesReversingLab/META-INF/MANIFEST.MF
[...]
```

**(2)** Disassemble Hermes bytecode from `index.android.bundle` file by using `hbctool`.

```
(hack) bongtrop@bongtrop-pc:lab/ $ hbctool disasm HermesReversingLab/assets/index.android.bundle HermesReversingLabHASM
[*] Disassemble 'HermesReversingLab/assets/index.android.bundle' to 'HermesReversingLabHASM' path
[*] Hermes Bytecode [ Source Hash: d0310a88a868dfb1ee21d12e9011725b1f716875, HBC Version: 74 ]
[*] Done
```

**(3)** After disassembling HBC (Hermes Bytecode) to HASM (I named it; stands for Hermes Assembly). In the `HermesReversingLabHASM` directory, there are 3 files as follows:

- `metadata.json`: stores the important information of Hermes bytecode file
- `instruction.hasm`: stores the application instructions or logics in HASM format (edit application logics in this file)
- `string.json`: store the application strings or texts (edit strings in this file)

**(4)** Edit the application's instruction in `HermesReversingLabHASM/instruction.hasm`. You can change the goal counter from 1336 to 1, or you can just change the opcode from `JNotGreaterEqual` to another `jmp` opcode. For this example, I would change the goal counter from 1336 to 1 by changing the number 1336 on line 182890 to 1.

From:

```
[...]
    LoadConstInt            Reg8:1, Imm32:1336
    JNotGreaterEqual        Addr8:43, Reg8:2, Reg8:1
[...]
```

To:

```
[...]
    LoadConstInt            Reg8:1, Imm32:1
    JNotGreaterEqual        Addr8:43, Reg8:2, Reg8:1
[...]
```

**(5)** Save the file and assemble HASM to the HBC by using `hbctool`.

```
(hack) bongtrop@bongtrop-pc:lab/ $ hbctool asm HermesReversingLabHASM HermesReversingLab/assets/index.android.bundle 
[*] Assemble 'HermesReversingLabHASM' to 'HermesReversingLab/assets/index.android.bundle' path
[*] Hermes Bytecode [ Source Hash: d0310a88a868dfb1ee21d12e9011725b1f716875, HBC Version: 74 ]
[*] Done
```

**(6)** Remove all signature files from `META-INF` directory and zip them to APK.

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

**(7)** Sign the APK file by using `jarsigner` or `apksigner`.

```
(hack) bongtrop@bongtrop-pc:lab/ $ jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore helloworld.keystore HermesReversingLab.patch.apk helloworld
Enter Passphrase for keystore:                                                
   adding: META-INF/MANIFEST.MF
   adding: META-INF/HELLOWOR.SF
   adding: META-INF/HELLOWOR.RSA
[...]
```

**(8)** Finally, after installing and opening the patched app, tap the + button twice. The flag will be shown. Please note that the example in the following gif file uses the [apkutil](https://github.com/aktsk/apkutil) to bundle and unbundle the APK file.

![hbctool example](https://i.imgur.com/70MBQ2c.gif)

For a higher quality video, see this [MP4](https://github.com/bongtrop/hbctool/raw/main/image/hbctool_example.mp4).

> I created this lab for anyone that wants to try reverse-engineering the Hermes bytecode application. If you get a flag, you can submit it at [lab.suam.wtf](https://lab.suam.wtf/lab/suam-team/hermes-reversing-lab).

## Summary

I wrote this blog because I want more people to be interested in doing an in-depth analysis of the mobile application framework. Researching the internals of any framework takes a huge time. I cannot research all frameworks by myself. Please research them and share your knowledge with the community, to help a poor pentester like me.

However, I hope that my tool can help pentester with bad times on mobile application pentest. The days are getting worse.

From the technical point of view, the Hermes engine is very 
interesting. There is no disadvantage to enabling it (except making it hard to perform reverse engineering), and it makes the application run faster. In the future, I think if the Hermes engine is stable, the React Native applications will be forced to migrate to it. Moreover, in my opinion, AOT for React Native is coming soon.