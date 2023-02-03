+++ 
authors = ["bongtrop",]
title = "TDEA (3DES) ปลอดภัยกว่า AES-128 จริงไหม"
date = "2023-02-03"
description = "มาทำความรู้จักกับการโจมตี Meet-in-the-Middle (MitM) เพื่ออธิบายความเสี่ยงของการเข้ารหัสแบบ Triple Data Encryption Algorithm (TDEA) และเลิกทำการเปรียบเทียบกับ AES-128 ซะที"
images = ["https://i.imgur.com/mQZLZOf.png",]
+++

![Cover](https://i.imgur.com/mQZLZOf.png)

สืบเนื่องมาจาก `Table 1` ในเอกสาร [NIST Special Publication 800-131A Revision 2](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-131Ar2.pdf) ตัว Three-key TDEA Encryption จะ deprecated และไม่อนุญาตให้ใช้งานแล้วหลังจาก 2023 (คือปีที่เขียนบทความนี้) เผื่อใครยังไม่รู้ TDEA ที่นิยมเลยคือ 3DES 

![NIST.SP.800-131Ar2 Table 1](https://i.imgur.com/qDQAWjC.png)

ดังนั้นหลาย ๆ standard หรือหลาย ๆ องค์กรได้เริ่มปรับตัวกันไปแล้วคือห้ามใช้ 3DES เพื่อเข้ารหัสใน application ใหม่ ๆ และตัว algorithm เองก็ archived ไปแล้วตั้งแต่ 2005 ทำให้เมื่อผม consult หรือ review security จะเจอเหตุผลต่าง ๆ นา ๆ มาโต้แย้ง โดยไม่ได้มีความเข้าใจในความเสี่ยงจริง ๆ ของตัว algorithm ผมเลยเขียนบทความนี้มาเป็นภาษาไทย เพื่อให้หลาย ๆ คนสามารถเข้าใจความเสี่ยงของการใช้งาน TDEA แบบมีหลักการและเหตุผลมากขึ้น

## TDEA Risks

ในบาง application บางทีจะติดเรื่องของ limitation ที่ยังจะจำเป็นที่จะต้องใช้ TDEA หรือ 3DES อยู่ เราจึงจำเป็นจะต้องมา analyse ถึงความเสี่ยง โดยความเสี่ยงใหญ่ ๆ ของตัว 3DES จะมีดังนี้

### Sweet32

จะเป็น Birthday Attack บนการเข้ารหัสแบบ 64-bit block ก็คือ DES หรือ 3DES นั้นเอง การโจมตีคือเมื่อมี collision ในบาง block เราจะสามารถได้ information ของ plaintext ออกมาได้ ซึ้ง การเข้ารหัสที่มีขนาด block เล็ก เช่น 64-bit จะทำให้มี sample space ของ plaintext และ ciphertext จึงมีโอกาศที่จะซ้ำกันสูง

แต่ใน case ของ Sweet32 เราสามารถที่จะ compansate ความเสี่ยงโดยการ rotate key ที่ใช้สำหรับ encrypt บ่อยมากขึ้น และใช้ 3DES ในการ encrypt data ที่ไม่ได้ใหญ่มากได้ ไม่เกิน $2^{n/2}$ blocks การซ้ำกันของ ciphertext จึงเกิดขึ้นได้ยากอยู่ใน level ที่รับได้

อย่างไรก็ตาม Sweet32 จะไม่ใช้หัวข้อหลักสำหรับบทความนี้ ดังนั้นจะขอข้ามไปก่อน สำหรับใครสนใจเป็นพิเศษสามารถอ่านเพิ่มเติมได้ที่ https://sweet32.info/

### Meet-in-the-Middle Attack (MitM)

ในปี 1977 สองตัวตึงแห่งวงการ Diffie และ Hellman เจอวิธีการที่จะสามารถ break double encryption schema ตัวอย่างง่าย ๆ คือการ encrypt ซ้อนกัน 2 ครั้ง คนละ key ได้โดยใช้เวลาแค่ 2 เท่าจากการ encrypt ครั้งเดียว โดยใช้ memory หรือ storage เข้ามาช่วย จากปกติจะต้องเพิ่มเป็น exponential เช่นการ DES ครั้งเดียวจะได้ security strength ที่ 56 bits คือจำเป็นจะต้อง brute force มากสุด $2^{56}$ ครั้งถึงจะได้ key กลับมา ถ้าเรา DES ซ้อนกัน 2 ครั้ง ($Enc_{k1}(Enc_{k2}(data))$) คนละ key เราก็ควรจะได้ security streangth ที่ 112 bits ใช้ไหมครับ เพราะ sample space ของ key คือ $2^{112}$ แต่โดยการใช้ MitM เราสามารถลดเวลาในการ brute force ลงเป็น $2^{57}$ สองครั้ง คือ $2^{56} + 2^{56} = 2^{57}$ เท่านั้น

ความคาดหวังของ security strength ของ algorithm double encryption schema คือการที่เราจะ break DES 2 ชั้น เราจำเป็นจะต้องเดา key ที่มีความยาว 112 bits (key 1 + key 2) ดังรูปต่อไปนี้

![double encryption schema](https://i.imgur.com/DLKYvsH.png)

แต่การโจมตี MitM สามารถ break ความคาดหวังนั้นโดยการที่เราแบ่งการ brute force ออกเป็น 2 ส่วน คือ

1. $Intermediate = Enc_{k1}(Plaintext)$
2. $Intermediate = Dec_{k2}(Ciphertext)$

การโจมตีจะเริ่มจาก 1 หรือ 2 ก่อนก็ได้ แต่เพื่อให้อธิบายง่ายผมจะเล่า โดยเริ่มจากทำส่วนที่ 2 ก่อน การโจมตีจะมีขั้นตอนดังนี้

1. ทำการ decrypt ciphertext โดยใช้ key ทุกค่าที่เป็นไปได้ 
2. เก็บ Intermediate ที่คู่กับ key ทุก ๆ ความเป็นไปได้ไว้ใน memory หรือ storage
3. เริ่มทำการ encrypt plaintext โดยใช้ key ที่ทำการสุ่มหรือเลือกมา
4. จับ Intermediate ที่ได้จากขั้นตอนที่ 3 ไป match กับที่เก็บไว้ในขั้นตอนที่ 2
   1. ถ้า match กันเราจะสามารถ break หา key ของการ encryption สำเร็จแล้ว
   2. ถ้าไม่ match กลับไปทำ 3 ใหม่จนกว่าจะเจอ key ที่ทำให้ match

\* ขั้นตอนที่ 1 และ 2 เป็นส่วนที่ 2 และ ขั้นตอนที่ 3 และ 4 เป็นส่วนที่ 1

จะเห็นว่า complexity ของขั้นตอนที่ 2 คือ $2^{56}$ และของขั้นตอนที่ 3 และ 4 ก็ยังคงเป็น $2^{56}$ เหมือนเดิม ทำให้การ break encryption double encryption schema มี complexity แค่เพียง $2^{56} + 2^{56} = 2^{57}$ เท่านั้นแต่เราคาดหวังกับ ตัว algorithm double encryption schema ตั้ง $2^{112}$ หายไปเกือบครึ่ง แลกกับการที่เราต้องมีพึ้นที่ในการ cache ผลลับของขั้นตอนที่ 2 เท่ากับจำนวน key (8 bytes) และ intermediate (8 bytes) เป็นจำนวน $2^{56}$ ชุด ซึ่งก็เป็นจำนวนไม่น้อยเช่นกัน

สำหรับ TDEA เราคาดหวังว่า security strength จะเป็น 168 bits เนื่องจากใช้ DES 3 รอบคือ $56 * 3 = 168$ เราก็สามารถนำการโจมตีแบบนี้มาประยุคใช้กับการ TDEA เพื่อลด security strength ลงเหลือ 112 bits ได้ดังนี้

![tdea](https://i.imgur.com/caEl8XX.png)

เหมือนเดิมเราก็แบบการโจมตีออกเป็น 2 ส่วนเช่นเดิมคือ

1. $Intermediate = Enc_{k1}(Dec_{k2}(Plaintext))$
2. $Intermediate = Dec_{k3}(Ciphertext)$

จากนั้นโจมตีโดยใช้ขั้นตอนเดิมเลยแค่เปลี่ยนการเตรียมการ logic นิดหน่อยในขั้นตอนทึ่ 3 เราก็จะสามารถ break TDEA โดยใช้ complexity ส่วนที่ 1 เป็น $2^{112}$ และส่วนที่ 2 เป็น $2^{56}$ รวมกันก็จะประมาณ $2^{112} + 2^{56} \approx 2^{112}$ เท่านั้น

ถ้าจะให้สรุปง่าย ๆ คือการโจมตีแบบ MitM เป็นการลด complexity ของการ brute force โดยใช้ memory หรือ storage มาจำ intermediate state ไว้ เพื่อให้สามารถทำแค่ครึ่งเดียว และจากนั้นนำมา search ของส่วนที่เหลือใน memory หรือ storage ได้ จุดสำคัญคือ technique นี้จะ work กับ algorithm ที่ความซับซ้อนเพิ่มขึ้นเป็น non-linear ใน layer ที่เพิ่มขึ้น อีกทั้งมีการนำ technique นี้ไปใช้ในการโจมตีประเภทอื่น ๆ ทางด้าน cyber security เช่น ไปใช้หา hash collision เป็นต้น

### สรุป

ดังนั้นถ้าพูดถึงความปลอดภัยของ TDEA เช่น 3DES เราจะไม่ได้ให้ security strength เป็น 168 bits เราจะมองมันมี security strength แค่ 112 ซึ้งน้อยกว่า AES-128 ที่มี security strength 128 bits ซะอีก แถมมีเรื่อง Sweet32 รวมไปถึงใช้ power ในการ computation ที่มากกว่าด้วย

ดังนั้นเหตุผลที่มีคนจำนวนไม่น้อยมาใช้อ้างเพื่อยังใช้งาน 3DES หรือ TDEA ได้ต่อไปคือ AES-128 ยังสามารถใช้ได้ดังนั้น 3DES ที่มี key ยาวตั้ง 168 bits ก็จะต้องใช้ได้ จึงไม่เป็นจริง เพราะ

1. AES มี blocksize 128 bits ส่วน 3DES มี blocksize แค่ 64 bits 3DES จึงเกิด birthday paradox ได้ง่ายกว่า (Sweet32)
2. เมื่อเจอการโจมตี MitM DES ไม่ได้มี security strength ที่ 168 bits มีแค่ 112 bits แม้ key จะยาวถึง 168 bits ก็ตาม

หวังว่าทุกคนจะเห็นภาพของความเสี่ยงของ 3DES มากขึ้นนะครับ แค่ไม่กี่คนก็ยังดี ขนาด NIST ยังไม่ effective ยังต้องอธิบายซ้ำ ๆ ขนาดนี้ ถ้า effective แล้วจะขนาดไหน ขอบคุณที่ฟังผมบ่นมาถึงตรงนี้นะครับ บทความนี้เกิดมาจากความเหนื่อยจากการต้องอธิบายซ้ำ ๆ ในเรื่องเดิม ๆ

จริง ๆ แล้ว MitM มีการโจมตีแบบหลาย ๆ dimension ที่มีความเจ๋งกว่านี้อีกมาก ถ้าใครสนใจสามารถเข้าไปอ่านได้ที่ https://en.wikipedia.org/wiki/Meet-in-the-middle_attack และกลับมาเขียน blog แชร์ความรู้สู้ community กันได้นะครับ