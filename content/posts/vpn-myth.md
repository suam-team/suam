+++ 
authors = ["bongtrop",]
title = "Virtual Private Network (VPN) Myths"
date = "2021-03-24"
description = "ความเข้าใจแปลก ๆ เกี่ยวกับ Virtual Private Network (VPN)"
images = ["https://i.imgur.com/dbpqE68.png?1",]
+++

![cover](https://i.imgur.com/dbpqE68.png?1)

ช่วงนี้เจอคนเข้าใจเกี่ยวกับ Virtual Private Network (VPN) แบบแปลก ๆ บ่อยมาก (เริ่มรำคาญ) และก็มีคนบางกลุ่มแจก VPN server ให้กับคนอื่นใช้งานฟรี ๆ และคนที่ไปใช้งานส่วนใหญ่ไม่ได้มีความเข้าใจเกี่ยวกับ VPN ที่มากพอ ทำให้ไม่รู้ความเสี่ยงที่ตัวเองจะเจอเมื่อไปใช้งาน VPN server ที่ไม่น่าเชื่อถือ

ก่อนที่จะไปเริ่มวิเคาะห์ความเสี่ยงต่าง ๆ ของ VPN เรามาทำความรู้จักกับ VPN แบบละเอียดกันก่อน เพื่อความเข้าใจ บทความนี้ผมจะมา implement โปรแกรม VPN อย่างง่ายด้วยภาษา Python แบบ step by step เพื่อให้ผู้อ่านสามารถทดลองทำตามและเข้าใจการทำงานของมันได้ง่ายขึ้น

> การจะเข้าใจบทความนี้ ผู้อ่านควรจะมีความรู้พื้นฐานเกี่ยวกับ Operating System และ Network มาก่อน หากใครที่ยังไม่ชำนาญแนะนำให้ข้ามไปอ่านหัวข้อการวิเคาะห์ความเสี่ยงได้เลยครับ

## Virtual Private Network (VPN)

ถ้าจะพูดให้ง่ายที่สุดเลย VPN ก็คือโปรแกรม ๆ หนึ่งที่จะทำการจำลอง network device ขึ้นมา และทำการนำข้อมูล network ต่าง ๆ ที่ถูกส่งเข้าไปใน network device นั้น ส่งต่อไปให้กับเครื่อง computer เครื่องอื่น หรือรับข้อมูล network จากเครื่องอื่นมาส่งออกทาง network device จำลองนั้นเอง เพื่อให้จินตนาการง่าย ๆ ภาพจะเป็นดัง diagram นี้ครับ

![VPN Overview](https://i.imgur.com/24jn7jG.png)

จะเห็นว่าโปรแกรม VPN นั้น implement ง่ายมาก สิ่งที่เราต้องทำมีแค่สามอย่างเท่านั้นคือ

- จำลอง network device ขึ้นมา
- อ่านหรือเขียน ข้อมูลไปที่ network device จำลอง
- ส่งหรือรับ ข้อมูล network ไปให้ computer อีกเครื่องหนึ่ง

## Virtual Network Device

ในการสร้าง network device จำลอง ทุกวันนี้ทำได้ไม่ยากเลย เพราะว่าตัว kernel ต่าง ๆ ส่วนใหญ่จะมีแถมมาให้อยู่แล้ว สำหรับ Linux จะชื่อว่า [TUN/TAP](https://en.wikipedia.org/wiki/TUN/TAP) ในการ load `TUN/TAP` driver แค่รันคำสั่ง `modprobe tun` ก็จะได้แล้ว เพื่อความง่ายผมจะอธิบายการสร้าง virtual network device ของ Linux และใช้งาน `TUN/TAP` kernel module เท่านั้น

โดน virtual network device ที่ถูกสร้างมาจาก `TUN/TAP` driver จะมีอยู่สอง mode คือ `TUN` และ `TAP` โดยมีความแตกต่างดังนี้ครับ

- `TUN`: จะเป็น network device ที่ Layer 3 หรือ Network Layer ตัว device จะมี IP address แต่ไม่มี MAC address และไม่สามารถส่ง packet ใน layer ที่ 2 ได้
- `TAP`: จะเป็น network device ที่ Layer 2 หรือ Data Link Layer ก็คล้าย ๆ กับ `TUN` แต่อยู่ layer 2

มาเริ่มใช้งาน `TUN/TAP` driver กัน หลังจากที่เรา load `TUN/TAP` driver เสร็จแล้วโดยใช้คำสั่ง

```bash
modprobe tun
```

เราจะเจอไฟล์ `/dev/net/tun` ถูกสร้างขึ้นมา จากนั้นเราจะสั่งงาน `TUN/TAP` driver ผ่านทางการอ่านและเขียนไฟล์นี้ครับ ใครที่อยากรู้วิธีการส่งงานต่าง ๆ ผ่านการอ่านและเขียนไฟล์ตรง ๆ สามารถอ่านได้จากไฟล์นี้ครับ [/Documentation/networking/tuntap.rst](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/networking/tuntap.rst?id=HEAD)

ไฟล์ที่อยู่ใน `/dev` เกือบทุกไฟล์ จะเป็นวิธีหนึ่งที่ใช้ติดต่อกับ kernel คนที่คุ้นเคยกับ Linux น่าจะรู้อยู่แล้ว จริง ๆ ก็ไม่ได้ใจร้ายขนาดที่ต้องไป interact กับ dev ไฟล์ ตรง ๆ ครับ ใน OS ส่วนใหญ่ก็จะแถมโปรแกรม `ip` มาให้ด้วยเราก็สามารถสั่งงาน `TUN/TAP` driver ด้วยวิธีนี้ได้เหมือนกัน เช่นสร้าง virtual network interface ชื่อ `tun0` เป็น mode `TUN` ก็จะใช้คำสั่งด้านล่างได้เลยครับ

```bash
ip tuntap add dev tun0 mode tun
```

เมื่อลอง `ip a` ก็จะเจอว่ามี virtual network device `tun0` ถูกสร้างขึ้นมาแล้ว

```
$ ip a                                
[...]
27: tun0: <POINTOPOINT,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 500
    link/none
```

คำสั่ง `ip` เราสามารถทำได้หลายอย่างมาก ๆ เลย เช่น กำหนดค่า IP ให้กับ device หรือกำหนด network route ต่าง ๆ สำหรับใครที่สนใจสามารถอ่านได้ [ip(8) — Linux manual page](https://man7.org/linux/man-pages/man8/ip.8.html)

หลังจากที่เราได้ virtual network device เราก็จะสามารถอ่านข้อมูล network ที่วิ่งผ่าน network device ที่เราสร้างขึ้นมาได้โดยการอ่านไฟล์ `/dev/net/tun` ได้เลยครับ แต่ก่อนจะอ่านได้อาจจะต้องมีการบอกกับ driver ก่อนว่าอยากจะดึงหรือเขียนข้อมูลอะไร ผมจะไม่ลงรายละเอียดในบทความนี้ สำหรับใครที่สนใจสามารถอ่านได้จาก source code ตัวอย่างนี้ได้เลยครับ [Reading/writing Linux's TUN/TAP device using Python](https://gist.github.com/glacjay/585369)

สำหรับในบทความนี้ เพื่อความง่ายเราจะใช้ Python library ชื่อว่า [python-pytun](https://github.com/montag451/pytun) ซึ่งจะจัดการเรื่องต่าง ๆ ที่เคยอธิบายในหัวข้อนี้ทั้งหมดให้ ติดตั้งโดยรันคำสั่งต่อไปนี้เลยครับ

```bash
pip install python-pytun
```

จากนั้นสร้างไฟล์ `tun_test.py` โดยที่จะเป็นการสร้าง tun device และทำการอ่านข้อมูล network ที่วิ่งเข้ามาที่ tun device

```python
from pytun import TunTapDevice

# Create TunTabDevice object
tun = TunTapDevice()

# Define tun device IP address, destination IP address (Peer to Peer connection), and netmask
tun.addr = '1.3.3.7'
tun.dstaddr = '1.3.3.8'
tun.netmask = '255.255.255.255'
tun.mtu = 1500

# Start tun device
tun.up()

print(f"[+] Enable {tun.name} device")

# Loop to read the network packet from tun device
while True:
    print(tun.read(tun.mtu))
```

จากนั้นทำการรันไฟล์ `tun_test.py` และทดลอง `ping` เข้าไปที่ device ที่ถูกสร้างขึ้นมาผลที่ได้คือ เราจะเห็นข้อมูล network ที่วิ่งผ่าน tun device แล้วดังรูปต่อไปนี้

![](https://i.imgur.com/1kdBe7T.png)

จากตัวอย่างจะเป็นการอ่านโดยใช้ `tun.read()` ถ้าเราจะเขียนข้อมูล network ลงไปใน device ก็แค่เปลี่ยนเป็น `tun.write()` สำหรับรายละเอียดเพิ่มเติมสามารถอ่านได้ใน Github repository ของ [python-pytun](https://github.com/montag451/pytun)  ครับ

จะเห็นว่าตอนนี้เราสามารถสร้าง virtual network device ได้แล้ว และสามารถเขียนโปรแกรมเพื่ออ่านและเขียนข้อมูล network กับ device ได้แล้วที่เหลือก็จะเป็นการส่งข้อมูลไปให้กับ computer เครื่องอื่น

## Socket Programming

ผมเลือกใช้ TCP socket เพื่อส่งข้อมูลระหว่างเครื่องครับ เพราะว่ามันเขียนง่าย, อธิบายง่าย, และคนส่วนใหญ่จะรู้จักอยู่แล้ว โดยผมจะแบ่งการทำงานออกเป็น 2 thread มีหน้าที่ดังนี้

- **Thread 1**: loop อ่านข้อมูลจาก tun device และทำการส่งออกทาง TCP socket ไปให้กับอีกเครื่องหนึ่ง
- **Thread 2**: loop รับข้อมูลจาก TCP socket จากนั้นนำมาเขียนเข้าไปใน tun device

ผมว่าใครอ่านมาถึงตรงนี้น่าจะนำไป implement VPN กันเองได้แล้ว ง่ายใช่ไหมครับ ไปดู code กันเลย

code ฝั่ง server (b7z_vpn_server.py) กำหนดให้ตัวเองมี IP address `1.3.3.7` และ peer ตรงข้ามเป็น `1.3.3.8` บางคนอาจจะไม่ชินกับการสร้าง network device เป็น peer to peer สรุปง่าย ๆ เลยคือมันจะเป็น device ที่เอาไว้คุยกับเครื่อง ๆ เดียวเท่านั้น ในตัวอย่างนี้คือจะคุยกับ `1.3.3.8` เท่านั้น

```python
from pytun import TunTapDevice
import threading
import socket

# Create TunTabDevice object
tun = TunTapDevice()

# Define tun device IP address, destination IP address (Peer to Peer connection), and netmask
tun.addr = '1.3.3.7'
tun.dstaddr = '1.3.3.8'
tun.netmask = '255.255.255.255'
tun.mtu = 1500

# Start tun device
tun.up()

print(f"[+] Enable {tun.name} device")

# Thread 1
def tun_task(conn):
    global tun
    global IS_CONNECT
    while IS_CONNECT:
        # Read network data from tun device
        data = tun.read(tun.mtu)
        try:
            # Send data with TCP Socket
            conn.send(data)
            print(f"[+] Send data {len(data)} bytes")
        except OSError as e:
            IS_CONNECT = False
            break

IS_CONNECT = False

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(('0.0.0.0', 31337))
    s.listen()
    conn, addr = s.accept()
    with conn:
        print('Connected by', addr)
        IS_CONNECT = True

        # Create and start Thread 1
        task = threading.Thread(target=tun_task, args=(conn,))
        task.start()

        # Thread 2
        while IS_CONNECT:
            # Receive network data from TCP Socket
            data = conn.recv(tun.mtu)
            if data:
                # Write network data to tun device
                tun.write(data)
                print(f"[+] Recv data {len(data)} bytes")
            else:
                IS_CONNECT = False


        task.join()
        conn.close()
```

code ฝั่ง client (b7z_vpn_client.py) ก็จะคล้าย ๆ กับ server เลย เปลี่ยนแค่ IP address ของ tun device สลับกับ server คือ ตัวเอง IP address (1.3.3.8) และ peer ตรงข้าม IP address (1.3.3.7) และเปลี่ยนจากรอรับ connection เป็น connect ไปหา server แทน

```python
from pytun import TunTapDevice
import threading
import socket

# Create TunTabDevice object
tun = TunTapDevice()

# Define tun device IP address, destination IP address (Peer to Peer connection), and netmask
tun.addr = '1.3.3.8'
tun.dstaddr = '1.3.3.7'
tun.netmask = '255.255.255.255'
tun.mtu = 1500

# Start tun device
tun.up()

print(f"[+] Enable {tun.name} device")

# Thread 1
def tun_task(conn):
    global tun
    global IS_CONNECT
    while IS_CONNECT:
        # Read network data from tun device
        data = tun.read(tun.mtu)
        try:
            # Send data with TCP Socket
            conn.send(data)
            print(f"[+] Send data {len(data)} bytes")
        except OSError as e:
            IS_CONNECT = False
            break

IS_CONNECT = False

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    # My testing server (replace this with your server)
    s.connect(('ggez.cc', 31337))
    IS_CONNECT = True

    # Create and start Thread 1
    task = threading.Thread(target=tun_task, args=(s,))
    task.start()

    # Thread 2
    while IS_CONNECT:
        # Receive network data from TCP Socket
        data = s.recv(tun.mtu)
        if data:
            # Write network data to tun device
            tun.write(data)
            print(f"[+] Recv data {len(data)} bytes")
        else:
            IS_CONNECT = False
    
    task.join()
    s.close()
```

จะเห็นว่า code ทั้งฝั่ง server และ client แทบไม่ต่างกันเลย และก็เขียนได้ง่ายมาก เมื่อรัน code VPN ของเราทั้งสองฝั่ง เราก็จะสามารถเชื่อต่อไปหา ฝั่งตรงข้ามได้แล้ว จากรูปต่อไปนี้จะเห็นว่าสามารถ connect ไปหา server ผ่าน IP address 1.3.3.7

![](https://i.imgur.com/J9cSOpU.png)

ตัวอย่างจากรูปด้านบนยังอาจจะไม่เห็นภาพชัดเจน มาลองดูตัวอย่างที่น่าจะมีคนชอบทำมากที่สุดดีกว่าคือ ให้เครื่อง client ส่ง network traffic มาออก internet ที่เครื่อง server

จากโปรแกรม VPN ของเราได้ทำการส่งข้อมูล network ไปมาได้อยู่แล้ว ดังนั่นสิ่งที่เราต้องทำก็จะเหลือไม่เยอะแล้วคือ

- ให้เครื่อง server forward network traffic จาก tun device ไปออก internet ที่ eth device (device ที่ใช้ออก internet เครื่อง server ผมคือ eth)
- จากนั้นทำ NAT เปลี่ยน source IP address ให้เป็นเครื่อง server ของทุก packet

วิธีการก็ทำตามคำสั่งต่อไปนี้ที่เครื่อง server เลยครับ

```bash
# เปิดใช้งาน function IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# อนุญาติ traffic ทั้งหมด จาก tun0 device
iptables -I INPUT 1 -i tun0 -j ACCEPT

# ยอมให้ forward traffic ทั้งหมด จาก tun0 ไปที่ eth0
iptables -I FORWARD 1 -i tun0 -o eth0 -j ACCEPT

# ยอมให้ forward traffic ทั้งหมด จาก eth0 ไปที่ tun0 
iptables -I FORWARD 1 -i eth0 -o tun0 -j ACCEPT

# ทำ NAT เมื่อมี output ไปที่ eth0
iptables -t nat -I POSTROUTING 1 -o eth0 -j MASQUERADE
```

เมื่อรันคำสั่งต่อไปนี้ครบแล้ว เครื่อง server จะสามารถทำการ forward ข้อมูลจาก tun ไปที่ eth ได้แล้ว จากนั้นเราต้องทำการกำหนด route table ในเครื่อง client ให้เมื่อมีการออก internet ให้ไปใช้ tun device แทน รันคำสั่งต่อไปนี้ (ผม route เฉพาะ 216.239.0.0/16 เนื่องจากจะ route เฉพาะ website `ifconfig.me`)

```bash
# IP address ที่อยู่ใน range 216.239.0.0/16 จะถูกส่งไปที่ tun0
ip route add 216.239.0.0/16 dev tun0
```

จากนั้นเมื่อ `curl ifconfig.me` ที่ฝั่ง client ก็จะได้ IP address ของ server แล้วดังรูปต่อไปนี้

![](https://i.imgur.com/S5VfdYj.png?1)

ถ้าอ่านถึงตรงนี้ แล้วผมว่าทุกคนน่าจะเข้าใจการทำงานของ VPN โดยละเอียดแล้วว่ามันทำงานยังไง flow ของข้อมูลเป็นยังไง ก็มาถึงช่วงสำคัญที่อยากจะพูดแล้ว

## ความเสี่ยงต่าง ๆ เมื่อใช้งาน VPN

> ขอบอกก่อนว่าโปรแกรม VPN ตัวอย่างจากหัวข้อที่แล้ว __อย่านำไปใช้งานกันจริงนะครับ__ มันไม่มีความปลอดภัยเลย อยากจะให้เป็นตัวอย่างเพื่อให้เห็นภาพแค่นั้นเอง

มาดูกันทีละส่วนกันครับ ส่วนแรกจะเป็นส่วนที่คน focus กันมากคือส่วนที่โปรแกรม VPN ส่งข้อมูลหากันครับ ดังรูปมีวงสีแดงไว้

![](https://i.imgur.com/YKecEjN.png)

ในส่วนนี้จะเป็นส่วนข้อมูลที่จะผ่าน router บ้านเรา, internet provider, หรือ gateway ต่าง ๆ แล้วเข้าไปหา VPN server ครับ ในส่วนนี้โปรแกรม VPN ที่ปลอดภัยจะต้องมีความสามารถแก้ปัญหาต่อไปนี้ได้ครับ

- **Confidentiality**: ข้อมูลที่วิ่งกันผ่าน internet เนี่ยจะต้องอ่านไม่ออก ถ้ามีคนมา sniff network ก็จะต้องอ่านไม่ออกเช่นกันครับ
- **Authentication**: จะต้องยืนยันตัวตนคนส่งข้อมูล network ได้ว่าเป็นคนที่มีสิทธิ์ในการส่งไหม
- **Integrity**: ข้อมูล network ที่ส่งไปมาห้ามถูกปลอมแปลงได้

เรามาลองวิเคราะห์ความปลอดภัยจาก 3 factor นี้ใน VPN ที่เราสร้างมาดูครับ คำตอบคือ

- **Confidentiality**: ไม่มีแน่นอนครับ ไม่ได้ทำการเข้ารหัส รับอะไรมาจาก tun device ก็ส่งอย่างนั้นเลยครับ
- **Authentication**: ก็ไม่มีอีกเช่นเคยครับ ใคร connect เข้ามาถูก port ตัว server ก็รับหมด ใจถึงพึ่งได้
- **Integrity**: Confidentiality ไม่มีก็อย่าหวังกับ Integrity ครับ

ใครที่จะใช้โปรแกรม VPN จากหัวข้อด้านบน ก็ไปเพิ่มความปลอดภัยก่อนใช้ด้วยนะครับ (เข้ารหัส, ทำ authen, และเพิ่ม integrity check) แต่ผมก็ไม่แนะนำให้ใช้หรือเขียนเองอยู่ดีครับ ในโลกตอนนี้มีหลาย ๆ protocol ให้เลือกใช้งานครับ

- [Internet Protocol Security (IPsec)](https://en.wikipedia.org/wiki/Internet_Protocol_Security)
- [Transport Layer Security (SSL/TLS)](https://en.wikipedia.org/wiki/Transport_Layer_Security)
- [Datagram Transport Layer Security (DTLS)](https://en.wikipedia.org/wiki/Datagram_Transport_Layer_Security)
- [WireGuard](https://en.wikipedia.org/wiki/WireGuard)

ส่วนใครขี้เกียจก็ใช้ VPN สำเร็จรูปที่มีคนสร้างมาอยู่แล้วได้เลยครับ เช่น

- [OpenVPN](https://en.wikipedia.org/wiki/OpenVPN) จะใช้ protocol Transport Layer Security (SSL/TLS)
- [Cisco AnyConnect VPN](https://en.wikipedia.org/wiki/AnyConnect) จะใช้ Datagram Transport Layer Security (DTLS)
- [WireGuard VPN](https://www.wireguard.com/) อันนี้ก็เป็น protocol ของตัวเอง **ผมชอบตัวนี้สุดแล้ว ใช้ง่ายจัดเลย แถมไวด้วย**

ถึงแม้ว่าจะใช้ product ที่มี 3 factor ครบ และเป็น product ที่ได้รับการยอมรับอยู่แล้ว ผู้ใช้ตัวโปรแกรมก็จำเป็นจะต้องมี security awareness ด้วย หลาย ๆ ครั้งจะมีเคสที่ถูกแฮกเนื่องจากใช้งานตัว VPN อย่างไม่เหมาะสม

ผมจะยกตัวอย่างง่าย ๆ เลย ลองดูภาพต่อไปนี้ครับ

![image from https://www.cisco.com/c/en/us/support/docs/security/anyconnect-secure-mobility-client/200533-AnyConnect-Configure-Basic-SSLVPN-for-I.html](https://i.imgur.com/HJLxnZT.jpg)

เท่าที่เจอมาผู้ใช้งานจะกดปุ่ม `Connect Anyway` โดยไม่คิดเลยครับ บางที IT Support ก็จะแนะนำผู้ใช้ให้กดปุ่มนี้เช่นกัน การกดปุ่มนี้โดยไม่คิดไม่ตรวจสอบ Server Certificate ให้ดีก่อน จะทำให้แฮกเกอร์สามารถทำ Man-in-the-Middle โปรแกรม VPN ได้เลย เราก็จะเสีย factor ความปลอดภัย หลาย ๆ ตัวไปเช่น Confidentiality และ Integrity

ต่อมาจะเป็นอีกจุดหนึ่งที่คนจะไม่ค่อยจะสนใจกัน แต่เป็นอีกจุดหนึ่งที่ควรจะใส่ใจเป็นพิเศษด้วยซ้ำ จากภาพที่วงแดงไว้เลยครับ

![](https://i.imgur.com/huLHQHe.png)

ความน่าเชื่อถือของ VPN server ก็เป็นสิ่งสำคัญ เนื่องจากเราส่งข้อมูล network ไปให้กับ VPN server ถึงแม้ว่าเราจะทำการ encrypt ข้อมูลแน่นหนายังไงคนที่จะแกะได้ก็จะเป็น VPN server ครับ ทำให้ VPN server มีสิทธิที่จะอ่านหรือแก้ไข ข้อมูล network ทั้งหมดที่เราส่งไปให้ สำหรับ VPN ภายในองค์กรที่ endpoint จะเป็นเครื่อง server ภายในองค์กรเอง และจะมีแค่ข้อมูลที่วิ่งเข้า network ภายในองค์กรเท่านั้น จะไม่ค่อยมีเรื่องให้ต้องห่วง แต่สำหรับ VPN ฟรีที่ใช้สำหรับ ออก internet ทั้งหลาย ใครใช้ก็คิดไว้เลยนะครับ ว่ายอมให้เขา Man-in-the-Middle แบบเต็มใจสุด ๆ ไปเลย เอาข้อมูลไปประเคนให้เขาถึงที่ 555555+

## ช่วงถามตอบ

**Q:** VPN เป็นเน็ตฟรี ? \
**A:** ถ้าอ่านมาถึงตรงนี้ก็น่าจะตอบได้ทุกคนว่า ไม่ใช่ แถมเสีย overhead ให้กับการทำ encryption อีก

**Q:** การใช้ VPN จะทำให้ internet เร็วขึ้น ? \
**A:** ถ้าอ่านมาถึงตรงนี้หลาย ๆ คนอาจจะตีความว่ามันจะเร็วขึ้นได้ยังไง มันไปอ้อม VPN server มาทีหนึ่ง ก็ต้องช้ากว่าอยู่แล้ว ผมก็เคยคิดแบบนั้นมาก่อนครับ แต่จริง ๆ แล้วมันเป็นไปได้อยู่ครับ ถ้าเจ้าของ VPN server มี private route ที่เร็วกว่า public route ปกติ แต่ก็นั่นแหละครับ การจะมี private route ที่ใหญ่และดี เป็นเรื่องที่ยาก อาจจะเป็นไปได้กับผู้ให้บริการเจ้าใหญ่ ๆ ครับ และก็อย่างของ Cloudflare จะเร็วขึ้นเพราะเขาเก็บ cache ของ website ส่วนใหญ่ไว้ให้แล้ว

**Q:** จะทำให้ใช้งาน internet ปลอดภัยขึ้นไหม ? \
**A:** ก็ต้องดูว่าปลอดภัยจากใคร VPN ก็เป็นแค่การย้ายจุดออก internet ไปที่อื่น แถมยังมีความเสี่ยงเรื่องโดน hack จากเจ้าของ VPN server เอง แต่ถ้าเลือก VPN server ดีๆ มีความน่าเชื่อถือ ก็จะปลอดภัยขึ้นแน่นอน

**Q:** การใช้ internet ในองค์กร ด้วย VPN จะปลอดภัยกว่า ? \
**A:** ก็แค่องค์กรจะดักจับข้อมูลเราไม่ได้ ไม่น่าจะปลอดภัยขึ้น

## สรุป

คือไปเจอหลาย ๆ คน ให้ความรู้เกี่ยวกับ VPN ได้วิบัติมาก และมีหลาย ๆ คนใช้ประโยชน์จากความไม่รู้ของคนอื่น ๆ หาประโยชน์ โดยการแจก VPN ฟรีไปทั่วเลย บางคนก็หลอกว่าเป็นเน็ตฟรี บางคนก็หลอกว่าใช้แล้วปลอดภัยขึ้น **ผมก็บอกได้แค่ว่าอย่าไปเชื่อคนพวกนั้นเลยครับ** เราไม่มีทางรู้ได้เลยว่าเขาจะเอา network traffic เราไปทำอะไรบ้าง ก็ถ้าจำเป็นต้องใช้งานจริง ๆ ก็เลือกเจ้าใหญ่ ๆ ที่มีความน่าเชื่อถือหน่อยละกันนะครับ