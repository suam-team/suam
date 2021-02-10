+++ 
authors = ["bongtrop",]
title = "Crypto: เรื่อง Nonce ๆ ของ Stream Cipher Part 2"
date = "2021-02-10"
description = "ความเสี่ยงและวิธีการแฮก Stream Cipher ในชิวิตประจำวัน Part 2"
+++

![banner](https://i.imgur.com/caEa57b.png)

สำหรับ Part อื่น ๆ
- [Crypto: เรื่อง Nonce ๆ ของ Stream Cipher Part 1](/posts/nonce-reuse-attack-on-stream-cipher/)
- Crypto: เรื่อง Nonce ๆ ของ Stream Cipher Part 2 (Part ปัจจุบัน)
- Crypto: เรื่อง Nonce ๆ ของ Stream Cipher Part 3 (Coming soon)

จาก Part ที่แล้วได้ฝาก Lab ไว้ให้ลองทำ **[GCM Nonce Reuse Lab](https://lab.suam.wtf/lab/suam-team/gcm-nonce-reuse-lab)** ถ้าใครลองอ่านหรือศึกษามาแล้ว จะเจอว่า GCM Mode ทุกอย่างจะเหมือนกับ CTR Mode เลย ต่างกันแค่เพิ่ม Authentication Tag เข้ามา ดังนั้นการ Solve ข้อนี้จะทำเหมือนกับ Lab **[CTR Static Nonce Lab](https://lab.suam.wtf/lab/suam-team/ctr-static-nonce-lab)** เลยครับ ขอไม่เฉลยละกันเพราะมันจะเหมือนกับหัวข้อ **Known-plaintext Attack on Stream Cipher** ในบทความ Part ที่ 1

ตอนแรกว่าจะไป xSalsa20 และ Poly1305 MAC ของ `libsodium` เลย แต่มาแวะเรื่องนี้ก่อนเพราะว่าตอนทำ Pentest เจอบ่อยพอสมควรที่ลูกค้าจะเข้าใจว่าแค่ใช้ AES-GCM แล้วก็คือปลอดภัยแล้ว **ขอย้ำตรงนี้เลยครับว่า วิธีการใช้งาน Algorithm ต่าง ๆ ก็สำคัญมาก** ควรจะใช้ให้ถูกต้องตาม Best Practice และควรจะมีคนที่มีความรู้มาช่วย Audit เป็นประจำ

## GCM Mode of Operation คือ

GCM ย่อมาจาก [Galois/Counter Mode](https://en.wikipedia.org/wiki/Galois/Counter_Mode) สรุปง่าย ๆ เลยนะครับ GCM จะเป็นการนำการเข้ารหัสด้วย CTR มารวมกับ GMAC หรือ Galois Message Authentication Code โดย CTR จะทำหน้าที่เข้ารหัสข้อมูล ส่วน GMAC จะทำหน้าที่ช่วยตรวจสอบ Integrity ของข้อมูล

โดยทั่วไปแล้ว ในการเข้ารหัสข้อมูลจะมีการทำ Authentication หลาย ๆ แบบ เช่น Encrypt-then-MAC (EtM), Encrypt-and-MAC (E&M), หรือ MAC-then-Encrypt (MtE) ใครสนใจอ่านได้ที่ [Authenticated Encryption](https://en.wikipedia.org/wiki/Authenticated_encryption) แต่ใน GCM จะเป็นการ Built-in ตัว Authentication เข้ามาในตัวมันเลยทำให้หลังจากเราเข้ารหัสแล้ว เราจะได้ผลลัพธ์ออกมาสองตัวคือ Ciphertext และ Authentication Tag เวลาต้องการจะถอดรหัสก็จะต้องใช้ Ciphertext และ Authentication Tag ที่ถูกต้อง เพื่อให้การถอดรหัสสำเร็จ

### การทำงานของ GCM

![GCM Diagram](https://i.imgur.com/Xko0R2G.png)

ภาพ Diagram ด้านบนจะแสดงภาพรวมการทำงานของ GCM Mode นะครับ โดยส่วนประสอบต่าง ๆ จะมีคำอธิบายดังนี้

| Symbol      | Description |
| ----------- | ----------- |
| $a \mathbin\Vert b$ | เป็นการนำ String b ไปต่อกับ String a |
| $P_i$ | Plaintext ใน Block ที่ i |
| $C_i$ | Ciphertext ใน Block ที่ i |
| $A$ | เป็นข้อมูลเพิ่มเติมเพื่อเอาไว้ทำ Authentication |
| $Nonce$ | ก็ Nonce นั่นแหละ ใน GCM จะมีขนาด 12 bytes หรือ 96 bits |
| $cnt$ | ตัวเลข Counter เหมือนกับ CTR มีขนาด 4 byttes |
| $J_i$ | เป็น Counter Block ใน Block ที่ $i$ จะเป็นการต่อกันของ $Nonce$ และ $cnt$ มีขนาด 128 bits หรือ 16 bytes โดย $cnt$ จะมีค่าคือ $i+1 \mod 2^{32}$ ตัวอย่างเช่น $J_0 = IV \mathbin\Vert 0^{31} \mathbin\Vert 1$ |
| $Enc_k(X)$ | จะเป็นการ Encrypt $X$ ด้วย Key $k$ โดยใช้ Block Cipher Encryption เช่น AES |
| $H$ | คือ $Enc_k(0^{128})$ |
| $EJ$ | คือ $Enc_k(J_0)$ |
| $Gmul_H(X)$ | จะเป็นการคูณกัน ระหว่าง Galois Field $GF(2^{128})$ สองตัว คือ H กับ X |
| $T$ | ค่า Authentication Tag |
| $len(X)$ | ความยาว bit ของ String X มีขนาด 64 bits |
| $L$ | คือ $len(A) \mathbin\Vert len(C)$ |

ถึงตรงนี้อาจจะ งง อยู่มาลองดูแบบละเอียดทีละส่วนกันครับ ผมจะอธิบายไปทีละส่วนคือ ส่วนของ Encryption และส่วนของ Authenticaion

### ส่วนแรก Encryption

![GCM in Encryption Part](https://i.imgur.com/1sK9Gk7.png)

ใครยังจำ CTR Mode จาก Part ที่แล้วได้บ้างครับ ใครยังไม่ได้อ่าน [ไปอ่านก่อน](/posts/nonce-reuse-attack-on-stream-cipher/) จะเห็นว่ามันคืออันเดียวกันเลยครับ เอา $Nonce$ กับ $cnt$ มาสร้างเป็น Key Stream และไป $\oplus$ (xor) กับ Plaintext ก็อย่างที่บอกไปตอนเกริ่นนำว่า **Known-plaintext Attack** จะใช้กับ $Nonce$ ซ้ำได้เหมือนกับ CTR เลย

จะมีข้อแตกต่างอยู่ไม่กี่อย่าง คือ $cnt$ ตัวแรก หรือ $J_0$ จะถูกนำไปใช้กับส่วน Authentication แล้ว ทำให้ $cnt$ เพื่อ Encryption จะเริ่มที่ $J_1$ แทน และ GCM ได้กำหนดตายตัวมาเลยว่า $Nonce$ และ $cnt$ จะมีขนาดเท่าไหร คือ $Nonce$ 12 bytes และ $cnt$ 4 bytes

ในส่วนของ Encryption ก็จะไม่มีอะไรมากผมจะข้ามไปเพราะได้เคยอธิบายไปหมดแล้วใน Part 1

### ส่วนที่สอง Authentication

GCM จะใช้ GMAC มาทำ Authentication และ GMAC จะเป็นการคำนวณบน Galois Field ดังนั้นก่อนที่จะทำความรู้จักกับ GMAC มาทำความรู้จักกับ Galois Field กันก่อน 

> ขอบอกไว้ก่อนว่าผมก็ไม่ค่อยจะแน่นเรื่องนี้ (ได้เรียนมาน้อยมากจากมหาลัยและโรงเรียน) ดังนั้นที่จะอธิบายต่อไปนี้เป็นความเข้าใจส่วนตัวอาจจะมีเข้าใจผิดบ้างนะครับ

#### Galois Field

Galois Field หรืออีกชื่อคือ Finite Field จะเป็น Set ของตัวเลขที่สามารถนำมา บวก, ลบ, คูณ, และหาร กับเพื่อนใน Set ได้ โดยผลลัพธ์จะต้องเป็นอีกตัวใน Set นั้น

งง แน่ ๆ ครับ งั้นกลับไปเรื่อง Abstract Algebra นิดหนึ่ง ใน Abstract Algebra จะมี Structure ยอดฮิตอยู่ 3 ตัว ที่ทุกคนน่าจะรู้จักกันคือ Group, Ring และ Field โดยจะมีลักษณะดังนี้

ต่อไปนี้จะเป็นการพูดให้เข้าใจง่าย ๆ คือจะใช้ บวก ลบ คูณ และ หาร ที่ทุกคนชินกันอยู่แล้วมายกตัวอย่างนะครับ จริง ๆ แล้วจะเป็น Operation อะไรก็ได้ที่มีคุณสมบัติครบ ถ้าใครอยากอ่าน Definition จริง ๆ อ่านได้ที่ [Link](http://www-users.math.umn.edu/~brubaker/docs/152/152groups.pdf) นี้

**(1) Group** จะเป็น Set ที่มี Operation ที่มีคุณสมบัติ Identity, Inverse, Associativity คุณสมบัติมันจะเหมือนกันการ $+$ ที่เรารู้จักกัน และสมาชิกใน Set 2 ตัวมาบวกกันและจะต้องได้ผลลัพธ์เป็นสมาชิกอีกตัว

ผมจะยกตัวอย่าง Group เป็น Integers หรือจำนวนเต็ม ($\mathbb{Z}$) นะครับ โดย Group $\mathbb{Z}$ จะมี Operation เป็น $+$ โดยจะมีคุณสมบัติดังนี้

- **Identity** คือค่าที่สมาชิกใน Group $+$ แล้ว จะได้ตัวเดิม สำหรับ $\mathbb{Z}$ คือ $0$
$$x+0=x$$
- **Inverse** คือเมื่อนำสมาชิกตัวใดตัวหนึ่งมา $+$ กับ Inverse ของมันจะได้ค่าเป็น Identity โดยที่ Inverse ของ + จะใช้เป็นเครื่องหมาย $-$
$$x+(-x)=0$$
- **Associativity** พูดง่าย ๆ คือการจัดหมู่หรือย้ายวงเล็บ
$$x+(y+z)=(x+y)+z$$

**(2) Ring** จะเป็น Set ที่มี Operation 2 ตัว คือตัวแรกจะมีคุณสมบัติเหมือนกับ Group Operation คือมี Identity, Inverse, Associativity ส่วนอีกตัวจะมีคุณสมบัติ Associativity และ Distributive มันจะเหมือนกับการคุณที่เรารู้จักกัน

สำหรับ Operation ตัวแรก $+$ พูดไปแล้วใน Group ส่วน Operation ตัวที่สองคือ $\times$ จะมีคุณสมบัติดังนี้

- **Associativity** พูดง่าย ๆ คือการจัดหมู่หรือย้ายวงเล็บ เหมือนกับ $+$ เลยครับ
$$x \times (y \times z)=(x \times y) \times z$$
- **Distributive** พูดง่าย ๆ คือการกระจายเข้าไปในวงเล็บคือ
$$x \times (y + z) = (x \times y) + (x \times z)$$

**(3) Field** เหมือน Ring เลยแต่เพิ่มเติมคือ Operation ที่สองจะมี Identity และ Inverse ทำให้เราสามารถหารได้ โดย $\mathbb{Z}$ จะมาใช้ยกตัวอย่างไม่ได้แล้วเพราะว่า $\mathbb{Z}$ ไม่สามารถหา Inverse ของ $\times$ ที่อยู่ใน Set ได้ เช่น

$$2^{-1}=0.5$$

จากที่ $0.5$ ไม่ได้อยู่ใน Set ของ $\mathbb{Z}$ ดังนั้น $\mathbb{Z}$ จึงเป็น Field ไม่ได้ ดังนั้นผมจะยกตัวอย่างเป็น Rational Number หรือจำนวนตรรกยะ ($\mathbb{Q}$) สำหรับ $\mathbb{Q}$ สมาชิกใน Set จะสามารถเขียนในรูปแบบของ $\mathbb{Z}$ สองตัวเป็นเศษส่วนกันได้ เช่น $\dfrac{1}{2}$

- **Indentity** ของ $\times$ ก็จะเป็น $1$
$$x \times 1=x$$
- **Inverse** ของ $\times$ ก็จะใช้เป็นยกกำลัง -1 เช่น $x^{-1}$ เมื่อเอามา $\times$ กันก็จะได้ Indentity กลับมา
$$x \times x^{-1} = x \times \dfrac{1}{x} = 1$$

ต่อไปเรามารู้จักกับตัวหลักของเราคือ Galois Field ($GF$) จะเป็นกลุ่มของตัวเลขจำนวนเต็มที่มีขนาดจำกัดเช่น $GF(5)$ ก็จะมีสมาชิกแค่ 5 ตัวคือ {0, 1, 2, 3, 4} ครับ หลายคนคงจะสงสัยว่าแล้วมันมีคุณสมบัติครบได้ยังไง เช่น 

$$3+4=7$$

$7$ ก็ไม่อยู่ใน Set มันจะเป็น Group ได้ยังไง คำตอบคือจะเป็นการนำ Modulus เข้ามาใช้ครับ เช่น 

$$3+4 \equiv 2 \pmod{5}$$

ทำให้เราสามารถ $+$ ได้แล้ว ส่วน $\times$ ก็แทบจะไม่ต่างกับการ $+$ เท่าไหร่นัก

ส่วน Inverse และ Identity ของ $\times$ การที่เราจะ Inverse ทุกตัวใน Set ได้ ตัวที่เป็น ตัว Modulus จะต้องเป็น Prime เท่านั้น ทำให้ $GF$ จะต้องมีขนาดเป็น Prime เช่น $GF(2)$ หรือ $GF(13)$ เป็นต้น ถึงจะมีสามารถหา Inverse ได้ โดย $GF$ ที่มีขนาดเป็น Prime จะเรียกว่า Prime Field ครับ

ใน $GF$ ก็จะมีอีกแบบหนี่ง และเป็นอันที่ใช้ทำ GMAC ของ GCM ชื่อว่า Extended Field โดย Field นี้ขนาดของ $GF$ จะอยู่ในรูปของ $p^m$ หรือ Prime ยกกำลังดัวจำนวนนับ เช่นใน GCM จะใช้ $GF(2^{128})$ โดย Extended Field จะมองตัวเลขใน Set เป็น Polynomial ครับ เช่น ใน $GF(3^2)$ ก็จะเป็นดังตารางนี้

| Number      | Polynomial |
| ----------- | ----------- |
| 0 | $1$ |
| 1 | $2$ |
| 2 | $x$ |
| 3 | $x + 1$ |
| 4 | $x + 2$ |
| 5 | $2x$ |
| 6 | $2x + 1$ |
| 7 | $2x + 2$ |

จะเห็นว่า Coefficient ของ Polynomial จะอยู่ใน $GF(3)$ และ Degree ของ Polynomial จะมีค่า 2 คือ $m$ จาก $GF(3^2)$ ในการ + ก็สามารถทำได้แล้ว แค่เอา Coef ใน Degree เดียวกันมา $+$ กันใน $GF(3)$ เหมือนกับ Polynomial ธรรมดาเลย เช่น

$$(x + 1) + (x + 2) = (1+1)x + (1+2)$$
$$(x + 1) + (x + 2) = 2x + 0$$
$$(x + 1) + (x + 2) = 2x $$

แต่จะมีปัญหาที่การ $\times$ เพราะว่าในการ $\times$ กันของ Polynomial จะทำให้ค่า Degree ของ Polynomial เพิ่มขึ้น และทำให้ผลลัพธ์ไม่ได้อยู่ใน Set เช่น

$$(x + 1) * (x + 2) = x^2 + x + 2x + 2$$
$$(x + 1) * (x + 2) = x^2 + 0x + 2$$
$$(x + 1) * (x + 2) = x^2 + 2$$

$x^2 + 2$ ไม่ได้อยู่ใน $GF(3^2)$ ดังนั้นในการจะทำ Extended Field จะมีสิ่งที่ต้องการเพิ่มมาคือ Irreducible Polynomial ($P$) หรือ Polynomial ที่ไม่สามารถลดรูปได้แล้วเพื่อจะได้นำผลลัพธ์จากการ $\times$ มา mod กับ $P$ และจะได้ค่าที่อยู่ใน Set กลับมา สำหรับการ mod ของ Polynomial ใครตั้งหารยาว Polynomial เป็นก็น่าจะ mod เป็นแล้ว ก็แค่หารแล้วเอาเศษเหลือมาเป็นผลลัพธ์ ดังนั้นผมจะไม่ขอพูดถึงนะครับ เช่น ผมกำหนดให้ $P$ ของ $GF(3^2)$ ของผมคือ $2x+1$ ถ้านำผลลัพธ์จากด้านบนมา mod ก็จะได้

$$x^2 + 2 \equiv y \pmod{2x+1}$$


$y$ มีค่าเท่าไหร่ไม่บอกหรอกไปทำเป็นการบ้านนะครับตั้งหารยาวโลด สำหรับ $P$ ของ GHASH ใน GCM คือ

$$P = 1 + a + a^2 + a^7 + a^{128}$$

คำถามต่อมาทำไมถึงต้องใช้ Irreducible Polynomial มา mod ไม่ mod ด้วยอะไรก็ได้หละ คำตอบคือเราต้องการหารด้วยครับ การ Inverse จำเป็นจะต้องใช้ Irreducible Polynomial ครับ สำหรับการ Inverse จะไม่พูดถึงในบทความนี้เพราะว่าไม่ค่อยจำเป็นมาก

ถ้าอ่านมาถึงตรงนี้จะเข้าใจแล้วว่า $GF$ ทำงานยังไงและเราสามารถนำ $GF$ ไปใช้งานได้ยังไง GCM ก็จะใช้ $GF(2^{128})$ ที่มี Irreducible Polynomial เป็น $1 + a + a^2 + a^7 + a^{128}$ เนี่ยแหละมา $+$ กับ $\times$ กันจนได้ค่า Authentication Tag

โดย $GF$ ก็จะมีคน Implement เป็นภาษาต่าง ๆ ไว้อยู่แล้ว เราสามารถไปเลือกใช้ได้เลยครับยกตัวอย่างเช่น ของ Python (https://github.com/popcornell/pyGF2) ไปลองใช้กันดูนะครับ

#### Galois Message Authentication Code (GMAC)

![GCM in Authentication Part](https://i.imgur.com/kbrdkpI.png)

> ผมจะขอตัด $A$ หรือข้อมูล Authentication เสริมออกไปก่อนนะครับเพื่อความง่ายในการอธิบาย พูดง่าย ๆ คือไม่ใส่ $A$

ถ้าลองไล่ตามรูปดูจะได้เป็นสมการดังต่อไปนี้เลยครับ คือเอา $GF(2^{128})$ มา $+$ กับ $\times$ กันไปเรื่อย ๆ จนได้ $T$ ครับ โดยที่ $H$ คือ $Enc_k(0^{128})$ และ $EJ$ คือ $Enc_k(J_0)$

$$T = (((((C_1*H)+C_2)*H)+L)*H)+EJ$$

สำหรับ Code GMAC ดูจาก [pycryptodome](https://github.com/Legrandin/pycryptodome/blob/master/lib/Crypto/Cipher/_mode_gcm.py) ได้เลยครับเขียนไว้อ่านง่ายมาก และมี Comment แทบทุกบรรทัด แนะนำให้ดูคู่ไปกับ Diagram

## The Forbidden Attack

จากสมาการด้านบน ถ้าเราทำการคูณ H เข้าไปในวงเล็บ ก็จะได้สมการดังนี้ครับ

$$T = C_1H^3 + C_2H^2 + L*H + EJ$$

จากนั้นนำ $T$ มา + ทั้ง 2 ฝั่งของสมการจะได้

$$0 = C_1H^3 + C_2H^2 + L*H + EJ + T$$

เกือบลืมบอกไปว่าใน $GF(2)$ การบวกและลบ จะเหมือนกันครับเพราะว่า

$$a+b \equiv a-b \pmod{2}$$

จะเห็นว่าเป็นสมการ Polynomial โดยที่เราไม่รู้ค่าอยู่ 2 ตัวครับคือ $H$ และ $EJ$  การจะแก้สมการ 2 ตัวแปรได้จะต้องใช้ 2 สมการใช้ไหมครับ ถ้ามีการใช้งาน Nonce และ Key ซ้ำกันเมื่อไหร่ เราจะได้สองสมการที่มี $H$ กับ $EJ$ ซ้ำกันได้ดังนี้

สมการที่ 1:
$$0 = C_{11}H^3 + C_{12}H^2 + L_{1}*H + EJ + T_{1}$$

สมการที่ 2:
$$0 = C_{21}H^3 + C_{22}H^2 + L_{2}*H + EJ + T_{2}$$

จากนั้นถ้าเรานำสมการที่ 1 และ 2 มา + กันจะได้

$$0 = (C_{11}+C_{21})H^3 + (C_{12}+C_{22})H^2 + (L_{1}+L_{2})H + (T_{1}+T_{2})$$

$EJ$ จะถูก + กันแล้วหายไป ทำให้ตอนนี้ เราจะมีตัวแปรที่ไม่รู้แค่ตัวเดียวคือ $H$ ดังนั้น ถ้าเรา Solve Polynomial สามารถหา roots ของ Polynomial นี้เราก็จะได้ H แล้ว มีหลาย Algorithm มากที่สามารถหาคำตอบของ Polynomial ได้ครับ ถ้าสนใจดูได้ที่ (https://en.wikipedia.org/wiki/Root-finding_algorithms)

จากนั้นเมื่อเรานำ $H$ ที่ได้มามาแทนค่าเข้าไปในสมการ 1 หรือ 2 เราก็จะได้ $EJ$ กลับมาครับ เช่น

$$EJ = C_{11}H^3 + C_{12}H^2 + L_{1}*H + T_{1}$$

จากนั้นเมื่อเราได้ $EJ$ กับ $H$ ครบแล้วต่อไปนี้เราจะสามารถหา Authentication Tag ของ Ciphertext อะไรก็ได้แล้วครับทำให้เราสามารถทำ Bit-Flipping Attack จาก Part 1 ได้เลยครับ คำนวณจากสมการต่อไปนี้

$$T = C_1H^3 + C_2H^2 + L*H + EJ$$

ถ้าอยากได้ตัวอย่างที่จับต้องได้ มีน้องได้เขียน Writeup โจทย์ CTF ที่เป็น Forbidden Attack ไว้ สามารถไปอ่านได้ที่ https://www.notion.so/sshv3-Writeups-Fincybersec2020-e69f68c26ef74997b1e79f2b32db13fd

สำหรับใครที่ งง ลองไปอ่าน [Part 1](/posts/nonce-reuse-attack-on-stream-cipher/) ก่อนนะครับ

## ทดลองแฮกกัน

ผมได้ทำ Lab ไว้ให้ ไปลองแฮกดูได้ครับ จะเป็นช่องโหว่ Forbidden Attack ให้ลองโจมตีครับ อยู่ที่ [Forbidden Attack Lab](https://lab.suam.wtf/lab/suam-team/forbidden-attack-lab) ลองทำกันได้ครับ

## การบ้านก่อนไป Part 3

ใน Part หน้าจะเกี่ยวกับ xSalsa20 และ Poly1305 MAC ที่อยากพูดแล้ว และก็น่าจะเป็น Part สุดท้ายแล้ว ใน Part นี้ก็จะมี Math เยอะหน่อย Part หน้าก็น่าจะไม่ต่างกันครับ

> ปล. หวังว่าจะยังไม่ทิ้งกันนะ รออ่าน Part ต่อไปด้วย

ใน Part ถัดไปจะพูดเกี่ยวกับ xSalsa20 และ Poly1305 MAC ของ `libsodium` จะเป็น Ciphersuite ยอดฮิตในปัจจุบัน ก่อนจะได้เขียน Part 3 ไปลองทำการบ้าน Lab นี้กันก่อนได้ครับ **[NACL Nonce Reuse Lab](https://lab.suam.wtf/lab/suam-team/nacl-nonce-reuse-lab)**