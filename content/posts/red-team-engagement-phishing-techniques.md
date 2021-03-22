+++ 
authors = ["P",]
title = "Red Team Engagement: Phishing Techniques"
date = "2020-07-06"
description = "สรุปเทคนิค Phishing สำหรับการทำ Red Team Engagement"
images = ["https://i.imgur.com/5FCon4F.png",]
+++

![cover](https://i.imgur.com/5FCon4F.png)

โพสต์นี้เป็นอีกโพสต์หนึ่งซึ่งผมจะใช้มันเป็นโน้ตสั้น ๆ สำหรับทดสิ่งที่ได้รู้มาเผื่อซักวันนึงเราจะได้ใช้มันและก็หวังว่าคนอื่นจะได้รับประโยชน์เช่นกัน สำหรับเนื้อหาที่เราจะเลือก*ทด*กันในวันนี้มาจาก SANS@MIC Webcast ในหัวข้อ **Catch and release: phishing techniques for the good guys** โดย Jan Kopřiva ครับ

> ดาวโหลดสไลด์ประกอบการบรรยายหรือดู Webcast เต็มย้อนหลังได้ที่ [SANS Webcast](https://www.sans.org/webcasts/sansatmic-catch-release-phishing-techniques-good-guys-115430)

- คีย์เวิร์ดสำคัญในการทำ content ของ phishing operation ให้มีโอกาสสำเร็จสูงคือ**การเล่นกับคน**ผ่านปัจจัยซึ่งทำให้การวิเคราะห์และตัดสินใจด้วยเหตุและผลถูกบั่นทอนลง ปัจจัยเหล่านั้นมีได้หลากหลาย แต่ในการบรรยายนี้จะเน้นไปที่ปัจจัยอยู่ 3 ลักษณะคือ **Known**, **Trustoworthy** และ **Urgent** ทั้ง 3 ปัจจัยนี้หากใช้อย่างถูกต้องก็จะสามารถช่วยเพิ่มอัตราความสำเร็จได้เป็นอย่างดี
- ขั้นตอนการ Recon ยังคงเป็นขั้นตอนที่สำคัญเพื่อให้รู้ Defensive mechanism ที่เป้าหมายมี อาทิ ผู้ให้บริการอีเมลของเป้าหมาย*น่าจะ*มีความสามารถในการตรวจจับและป้องกันอีเมลปลอมได้ดีแค่ไหน หรือในฝั่งของเป้าหมายเองมีการตั้งค่า SPF/DKIM/DMARC ไว้อย่างไรเพื่อป้องการโจมตีในลักษณะนี้
- เทคโนโลยีอย่าง SPF/DKIM/DMARC เป็นเทคโนโลยีซึ่งบางครั้งการเอามาใช้ให้ถูกต้องก็เป็นเรื่องที่ยาก ดังนั้นการนำผลลัพธ์จากการ Recon มาดูอาจช่วยให้เราพบอะไรที่น่าสนใจ และนำไปสู่การ bypass การป้องกันได้ ตัวอย่าง [#56742 SPF whitelist of mandrill leads to email forgery](https://hackerone.com/reports/56742)
- เทคนิคการเล่นกับชื่อผู้ส่ง
  - ปลอมทั้งหมดผ่าน SMTP server ของเราเองแต่อาจเจอปัญหาในการตรวจสอบย้อนกลับ หรือปัญหาเกี่ยวกับ reputation ของ SMTP server
  - ใช้ทริคปลอมอีเมลแอดเดรส เช่น ระบุ Sender address ให้มี `First Last <fake@emai-address.com>` ไปด้วย
- เทคนิคการเล่นกับลิงค์ทั้งลิงค์ของ SMTP server หรือในเนื้อหาของอีเมล
  - ใช้เทคนิค Character substituion โดยอาจลองหาผ่าน [elceef/dnstwist](https://github.com/elceef/dnstwist)
  - ใช้เทคนิค IDN homograph โดยใช้ [Homoglyph Attack Generator](https://www.irongeek.com/homoglyph-attack-generator.php) ช่วยสร้าง
  - ใช้เทคนิค multi-level domain name เช่น [ตัวอย่างในเคสของ DNC hack ตอนปี 2016](https://www.vice.com/en_us/article/mg7xjb/how-hackers-broke-into-john-podesta-and-colin-powells-gmail-accounts)
  - ใช้เทคนิค Open redirect ให้ลิงค์ที่ไปยังเว็บไซต์ปลอมผ่านเว็บไซต์ที่น่าเชื่อถือก่อน
  - ใช้การใส่ attribute `title` ในแท็ก `<a>` กับเป้าหมายที่ใช้เว็บอีเมลเป็นไคล์เอนต์หลัก เช่น `<a href="https://fake.com>" title="https://godd.com">link</a>`
- เทคนิคการเล่นกับเนื้อหาของอีเมล
  - หลีกเลี่ยงการใช้คีย์เวิร์ดยอดนิยมที่อาจถูกระบบตรวจจับแบบ keyword matching ตรวจเจอ เช่น Microsoft, Account, Urgent, Invoice
  - ใช้เทคนิค [Z-WASP](https://www.avanan.com/blog/zwasp-microsoft-office-365-phishing-vulnerability) ในการช่วยแตกคำเพื่อป้องกันการตรวจจับ
  - ใช้เทคนิค [Ropemaker](https://blog.knowbe4.com/the-ropemaker-email-exploit-can-change-an-already-delivered-email) ซึ่งมีการนำโค้ด CSS มาใช้ในการเปลี่ยนหน้าตาเนื้อหาเมื่ออีเมลถูกเปิดอ่าน
  - ใช้เทคนิค Zerofont หรือการแทรกตัวอักษรขนาดเล็กระหว่างคำที่มักจะถูกตรวจจับโดยระบบ

อัปเดตเทคนิคเพิ่มเติมจากประสบการณ์การทำ Phishing simulation

- พิจารณาการใช้บริการอย่าง Mailchip หรือ Sendgrid ในการช่วยส่งอีเมลเนื่องจาก reputation ที่ดีกว่า
- ตั้งค่า `Return-Path` ให้ตรงกับอีเมลเป้าหมายเพื่อกำหนดปลายทางหากเกิดกรณี email bouncing
- พิจารณาการตั้งค่าหรือเปลี่ยน A record ใน DNS หรือตั้ง TTL ให้ต่ำหลังจากมีการส่งอีเมลออกไปแล้ว เนื่องบริการอีเมลบางบริการจะมีการตรวจสอบอีเมลหลังจากที่มีการส่งออกไปและจะพบปัญหาในการตรวจสอบ ซึ่งอาจช่วยให้สามารถข้ามผ่านกระบวนการตรวจสอบได้
- หากเป็นไปได้ควรหลีกเลี่ยงลักษณะของเนื้อหาที่สามารถถูกตรวจสอบได้ด้วยวิธีการ pattern matching เช่น HTML หรือ text ทั่วไป และควรระมัดระวังเป็นอย่างยิ่งหากมีการใช้ payload ที่เป็นนิยม เช่น ไฟล์เอกสารแนบมาโครสคริปต์
- หากต้องมีการใส่ไฟล์แนบ วิธีการปลอดภัยที่สุดคือการอัปโหลดไฟล์แนบดังกล่าวไปยังบริการฝากไฟล์ที่มีความน่าเชื่อถือสูง หรือในระบบที่เราสามารถควบคุมได้ เพื่อลดความเสี่ยงในการที่จะถูกดึงไฟล์แนบออกไปตรวจสอบ
- ทำการปรับเปลี่ยนคำใน payload เพื่อป้องกันการทำ pattern matching ตัวอย่างเช่น หาก payload มีการใช้คำว่า `powershell.exe` เราอาจสามารถใช้สคริปต์อย่าง [Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation) มาช่วยในการปกปิด ให้ได้ผลลัพธ์ที่ไม่สามารถตรวจจับได้ทันที เช่น `"po" & "w" & "er" & "s" & "he" & "l" & "l" & ".e" & "x" & "e" & " "`
- ต้องตรวจสอบการเข้าถึงหน้าเว็บไซต์ปลอมเสมอ และใช้วิธีการ เช่น `<meta>` ในการเปลี่ยนเส้นทางความพยายามเข้าถึงที่ไม่ถูกต้องออก

แหล่งข้อมูลเพิ่มเติม:

- [Red Team Techniques: Gaining access on an external engagement through spear-phishing](https://blog.sublimesecurity.com/red-team-techniques-gaining-access-on-an-external-engagement-through-spear-phishing/)
- [MetaMorph HTML Obfuscation Phishing Attack](https://www.avanan.com/blog/metamorph-html-obfuscation-phishing-attack)