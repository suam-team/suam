+++ 
authors = ["bongtrop",]
title = "Crypto: เรื่อง Nonce ๆ ของ Stream Cipher Part 1"
date = "2020-12-03"
description = "ความเสี่ยงและวิธีการแฮก Stream Cipher ในชิวิตประจำวัน Part 1"
+++

ช่วงนี้ Trend การใช้งาน Stream Cipher กำลังมาแรงมาก เนื่องจากความเร็วในการเข้ารหัส และความง่ายของตัว Algorithm อย่าง TLS 1.3 ก็หันมาใช้งาน Stream Cipher เป็นหลักแล้ว เช่น AES-GCM และ ChaCha20 

และการใช้งาน Stream Cipher มันจะมีข้อหนึ่งที่ต้องระวังมาก ๆ ๆ ๆ ๆ คือการใช้คู่ Nonce และ Key ซ้ำ ในการเข้ารหัสมากกว่า 1 รอบ (โครตบาป) จะเห็นได้จากช่องโหว่ ดัง ๆ มากมายที่ Impact เกิดจากการใช้งาน Nonce ซ้ำ เช่น 
KRACK Attacks บน WPA2 หรือ Forbidden Attack บน TLS

## Stream Cipher คือ ?

ก่อนที่จะมาดูการโจมตี เรามาทำความรู้จักกับ Stream Cipher กันก่อน ตัว Stream Cipher จะเป็นการเข้ารหัสแบบ Symmetric Key Cipher แบบหนึ่ง คือ การเข้ารหัส และถอดรหัส จะใช้ Key เดียวกัน

![Symmetric Key Cipher](https://ggez.cc/img/symmetric_key_cipher.png)

สำหรับทำงานแบบง่าย ๆ เลยคือ Stream Cipher จะเป็นการใช้ประโยชน์จาก Function Random อะไรก็ได้ มาสร้างข้อมูลมั่ว ๆ ยาว ๆ ออกมา โดยหลังจากนี้ Function Random ผมจะเรียกว่า [PRNG](https://en.wikipedia.org/wiki/Pseudorandom_number_generator) และข้อมูล มั่ว ๆ ที่ได้ออกมาจาก PRNG จะเรียกว่า Key Stream

```
Seed = Do_Something(SecretKey, Nonce)
KeyStream = PRNG(Seed)

# KeyStream จะเป็น Byte มั้ว ๆ ต่อกันไปเรื่อย ๆ
```

หลังจากได้ Key Stream ที่เป็นค่า มั่ว ๆ มาแล้ว ก็จะเอามันมา XOR กับ Plaintext ก็จะเป็นอันเสร็จแล้ว

```
Ciphertext = KeyStream ^ Plaintext
```

ในการถอดรหัส ถ้าคนถอดรหัสมี SecretKey และ Nonce ค่าเดียวกับตอนเข้ารหัส Seed ที่ได้ก็จะมีค่าเท่ากันดังนั้น Function PRNG ก็จะให้ Key Stream เป็นค่าเดียวกันกับตอนเข้ารหัส เมื่อเอามันมา XOR กับ Ciphertext ก็จะได้ Plaintext กลับมา

```
Plaintext = KeyStream ^ Ciphertext
```

ลองดูรูปด้านล่างเผื่อจะเข้าใจมากขึ้น

![Stream Cipher](https://ggez.cc/img/stream_cipher.png)

ตัวอย่างยอดฮิตในอดีตก็จะเป็น [RC4](https://en.wikipedia.org/wiki/RC4) ที่แตกพ่ายไปแล้ว และก็จะเป็น Stream Cipher ที่เอา Block Cipher มาสร้าง Key Stream เช่น AES-CTR หรือ [AES-GCM](https://en.wikipedia.org/wiki/Galois/Counter_Mode) (บางคนจะมองเป็น Block Cipher แต่ผมมองเป็น Stream Cipher) และยอดฮิตล่าสุดก็จะเป็น ChaCha20 หรือ [Salsa20](https://en.wikipedia.org/wiki/Salsa20)

## Known-plaintext Attack on Stream Cipher

ก่อนที่จะอ่านต่อไป ผมอยากให้ทุกคนลองคิดวิธีโจมตี ถ้าเกิดว่าเรารู้ ทั้ง Plaintext และ Ciphertext เราจะสามารถตบ Stream Cipher ยังไงดี ให้เวลา 3.14159265359 วินาที

> ปล. ถ้าใครรู้แล้วก็ข้ามบทนี้ไปได้เลยครับ และไปลองทำ Lab [CTR Static Nonce Lab](https://lab.suam.wtf/lab/suam-team/ctr-static-nonce-lab) ดู

มาเริ่มกันเลย ทวนกันก่อน วิธีการเข้ารหัสของ Stream Cipher จะเป็นการเอา Key Stream ที่สร้างมาจาก PRNG มา XOR กับ Plaintext และได้ Ciphertext ใช่ไหมครับ แล้วถ้าเราเอา Plaintext มา XOR กับ Ciphertext หละจะได้อะไร คำตอบคือเราก็จะได้ Key Stream กลับมานั้นเอง

```
KeyStream = Ciphertext ^ Plaintext
```

คำถามต่อมาคือ แล้วเราจะเอา Key Stream ไปทำอะไรได้หละ ยังจำหัวข้อของ Blog นี้ได้อยู่ไหมครับ คือการใช้งาน Nonce ซ้ำในการเข้ารหัสหลาย ๆ ครั้ง ลองจินตนาการตามผมดูครับ ถ้าเราใช้ Nonce ซ้ำกันเมื่อไหร่ Key Stream ที่ได้มาจะซ้ำกันใช่ไหมครับ แล้วรวมกับท่า Known-plaintext Attack ที่สามารถ leak Key Stream ได้เราก็จะสามารถ ถอดรหัสได้โดยที่ไม่จำเป็นต้องรู้ Secret Key เลย ลองคิดตาม Scenario ต่อไปนี้ครับ

1. เรารู้ Plaintext และ Ciphertext ของ set A
```
PlaintextA = "I love you"
CiphertextA = "\x86\x39\xfa\xcc\x80\x6d\xe9\x3e\x96\x72"
```

2. จากนั้นมีการใช้ Key และ Nonce ซ้ำเข้ารหัส Plaintext B แต่เรารู้แค่ Ciphertext B
```
CiphertextB = "\x86\x39\xfe\xc2\x82\x6d\xe9\x3e\x96\x72"
```

3. เราสามารถ Leak Key Stream ได้ดังนี้
```
KeyStream = CiphertextA ^ PlaintextA = "\xcf\x19\x96\xa3\xf6\x08\xc9\x47\xf9\x07"
```

4. สุดท้ายเราก็จะสามารถ ถอดรหัส Ciphertext B ได้โดยที่ไม่จำเป็นต้องรู้ Secret Key
```
PlaintextB = KeyStream ^ CiphertextB = "I hate you"
```

จะเห็นว่าถ้าเราใช้ Nonce ซ้ำเมื่อไหร่ ปัญหาจะเกิดง่ายมาก

## Choosen-ciphertext Attack on Stream Cipher

Stream Cipher ยังมีอีกเรื่องที่น่ากลัวครับ มันอ่อนแอกับการโดนเปลี่ยน Ciphertext มาก หลาย ๆ คนจะเรียกมันว่าการทำ Bit-flipping Attack 

วิธีการโจมตีจะคล้าย ๆ กับหัวข้อที่แล้ว แต่ครั้งนี้จะเป็นการที่เราเปลี่ยน Ciphertext เพื่อให้ Plaintext ที่ได้หลังจากถอดรหัสแล้วเปลี่ยนเป็นค่าที่เราต้องการแทนครับ 

จากหัวข้อที่แล้วเราสามารถ leak Key Stream และนำ Key Stream ดังกล่าวไปถอดรหัสข้อความอื่น ๆ ได้ แต่ครั้งนี้เราจะนำ Key Stream มาสร้างเป็น Ciphertext ใหม่ขึ้นมาแทน ลองคิดตาม Scenario นี้นะครับ

1. นางสาว Alice จะส่งข้อความให้ นาย Bob ว่า "I love you"
```
KeyStream = PRNG(Key, Nonce)
Ciphertext = KeyStream ^ "I love you"
```

2. แต่บังเอิญว่าถูกนางสาว Eve ทำการดักข้อมูลได้ และรู้ว่านางสาว Alice จะส่งข้อความอะไร นางสาว Eve เลยทำการโจมตีด้วย Bit-flipping Attack แก้ไขข้อความเป็น "I hate you"
```
KeyStream = Ciphertext ^ "I love you"
Ciphertext = KeyStream ^ "I hate you"
```

3. เมื่อ นาย Bob ได้รับ Ciphertext และทำการถอดรหัสก็พบว่าเป็นข้อความ "I hate you" และบอกเลิก นางสาว Alice
```
KeyStream = PRNG(Key, Nonce)
Plaintext = KeyStream ^ Ciphertext = "I Hate you"
```

4. จบปิ้ง

จะเห็นว่าการโจมตีพวกนี้จะวน ๆ กับการที่เราสามารถ leak Key Stream ได้แล้วเราทำอะไรได้ต่อครับ

## การเข้ารหัสแบบ AES-CTR

ตอนแรกว่าจะเริ่มที่ RC4 เลย แต่พอคิดไปคิดมาแล้ว มันเป็น การเข้ารหัสที่ Broken และไม่มีใครใช้งานแล้วในปัจจุบัน ผมจะข้าม ๆ ไปนะครับ

โดยการเข้ารหัส แบบ AES-CTR จะเป็นการนำ AES ที่เป็น Block Cipher มาใช้สร้าง Key Stream แทนที่จะใช้ PRNG ครับ การทำงานแบบง่าย ๆ คือ มันจะใช้ AES มาทำการเข้ารหัส Nonce รวมกับ Counter โดย Counter จะเป็นเลขเรียงกันที่เพิ่มขึ้นเรื่อย ๆ เราก็จะได้ Key Stream มา จากนั้นก็เหมือนกับ Stream Cipher อื่น ๆ แล้ว

![AES-CTR](https://ggez.cc/img/aes_ctr.png)

## ทดลองแฮกกันเลย

หลังจากลองจินตนา การการตบ Stream Cipher กันไปแล้ว มาลองตบจาก Lab จริงกันครับโดย Lab นี้จะต้องใช้การโจมตีทั้งสองแบบเลยครับ ฃองไปทำกันดู **[CTR Static Nonce Lab](https://lab.suam.wtf/lab/suam-team/ctr-static-nonce-lab)**

## ช่องโหว่ของแถมของ CTR Mode

ในการสร้าง Key Stream ของ CTR จะเป็นการที่บวกเลขขึ้นไปเรื่อย ๆ ใช่ไหมครับ ลองคิดดูเล่น ๆ นะครับ ถ้ามีวันหนึ่ง Counter มันถูกเพิ่มขึ้นไปจนเกินกว่า ที่จะเข้ารหัสได้ด้วย AES จะเป็นยังไง ก็จะเป็นอีก Lab หนึ่งที่ทำไว้ ให้ลองตบเล่น ๆ ครับ **[CTR Broken Couter Lab](https://lab.suam.wtf/lab/suam-team/ctr-broken-couter-lab)**

ขอใบ้ด้วยรูป เดียวจะไม่มีคนทำ ลองหาวิธีดูครับ

![Broken Counter](https://ggez.cc/img/broken_counter.png)

## การบ้านก่อนจะไป Part 2

ตอนแรกผมคุยกับเพื่อน P (@pe3zx) ว่าจะทำ Research เกี่ยวกับ ChaCha20 และ Poly1305 แต่ว่า หลังจาก Research เสร็จ แล้วจะเอามาเขียนกลัวจะไม่มีใครอ่าน เลยทำเป็น Part ปูความรู้พื้นฐานไปก่อน และทำ Lab ต่าง ๆ เผื่อใครไม่เข้าใจจะได้ไปลองเล่นเองได้

> ปล. หวังว่าจะอ่านรู้เรื่องกันนะครับ เขิน

ใน Part ถัดไปจะพูดเกี่ยวกับ AES-GCM ที่จะเป็น AES-CTR ที่มีการเพิ่มตัว Built-in Authentication Mechanism เข้ามา แต่วิธีแฮกจะคล้าย ๆ กันมาก ลองไปอ่านกันดูก่อนนะครับ และก็ลองเล่น Lab นี้ดูครับ **[GCM Nonce Reuse Lab](https://lab.suam.wtf/lab/suam-team/gcm-nonce-reuse-lab)**
