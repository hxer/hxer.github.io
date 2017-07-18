---
title: 基于CBC模式模式的密文攻击
date: 2017-03-09 21:12:13
categories:
    - 密码学
tags:
    - CBC
    - 比特翻转
    - Padding Oracle
---

关于CBC模式下的加密和解密，这里不过多阐述，可以参见 {% post_link 分组密码工作模式 分组密码工作模式 %}。

对于CBC解密可以用以下式子表示：

* Plaintext-1 = Decrypt(Ciphertext-1) XOR IV  只用于第一个组块
* Plaintext-N = Decrypt(Ciphertext-N) XOR Ciphertext-N-1   用于第二及剩下的组块

<!-- more -->

## 比特翻转攻击

从CBC解密表达式可以知道，得到明文的前一步操作是异或操作，改变密文中的`一位`只会导致其对应的明文块完全改变，下一个明文块中的`对应位`发生改变，而不会影响到其它明文块的内容。
因此，修改IV可以操纵解密出的第一组明文，修改某一个密文分组可以操纵后一个解密出的明文分组，也就是说CBC模式存在两个攻击点：

* IV向量，影响第一个明文分组
* 第N个密文分组, 影响第N+1个明文分组

因此对IV和密文分组进行比特翻转攻击，可以伪造某个明文分组对应的比特位。

这里给出如下Demo程序，输入参数`a`为AES CBC模式加密后的密文(16进制形式)，程序输出解密后的部分明文信息(id, name, email)，要求伪造密文，使得明文`id`对应的值为`0`, 程序如下：

```php
<?php
// a=89b52bac0331cb0b393c1ac828b4ee0f07861f030a8a3dc4b6e786f473b52182000a0d4ce2145994573a92d257a514d1
$cipherText = $_GET['a'];       
$key = hex2bin('66616974683434343407070707070707');     // 16 bytes
$iv = hex2bin('f4ebb2df9c29efd7625561a15096cd24');      // 16 bytes
$td = mcrypt_module_open(MCRYPT_RIJNDAEL_128, '', MCRYPT_MODE_CBC, '');
if (mcrypt_generic_init($td, $key, $iv) != -1){
    $p_t = mdecrypt_generic($td, hex2bin($cipherText));
    mcrypt_generic_deinit($td);
    mcrypt_module_close($td);
    $p_t = trimEnd($p_t);
    $tmp = explode(':', $p_t);
    if ($tmp[2] == '0'){
        print @'id:'.@$tmp[2].'<br/>';
        echo 'you are right';
    } else{
        echo @'id:'.@$tmp[2].'<br/>';
        echo @'name:'.@$tmp[0].'<br/>';
        echo @'email:'.@$tmp[1].'<br/>';
    }
}

function pad2Length($text, $padlen){
    $len = strlen($text)%$padlen;
    $res = $text;
    $span = $padlen-$len;
    for($i=0; $i<$span; $i++){
        $res .= chr($span);
    }
    return $res;
}

function trimEnd($text){
    $len = strlen($text);
    $c = $text[$len-1];
    if(ord($c) <$len){
        for($i=$len-ord($c); $i<$len; $i++){
            if($text[$i] != $c){
                return $text;
            }
        }
        return substr($text, 0, $len-ord($c));
    }
    return $text;
}
```

{% asset_img CBC_Dec.png %}

已知密文长度为48 bytes, 若要改变明文id的值，假设id在第N块明文中第x个字符，表示为`N[x]`(x从1开始)，这里仅简单假设id为单字符，则改变密文`N-1[x]`或IV[x]， 即可伪造我们需要的id值。显然，我们需要一种方案快速定位到相应的密文位置或IV位置。

已知XOR异或运算: X xor Y = Z, X、Y、Z三个数任意两个运算都可得到第三个。

若不考虑IV, 则明文id肯定可以表示为 `id=A XOR B`，A是我们可控用来伪造id的密文值，那么改变密文A为:`A1 = id`, 则得到的明文id为`A1 XOR B = A XOR B XOR B = A`, 即判断明文id是否修改为`A`，就能知道是否找到相应的密文位置`N-1[x]`。假设我们需要伪造的id为`fake_id`, `fake_A= A XOR id XOR fake_id`, 得到的明文id就是`fake_A XOR B = A XOR id XOR fake_id XOR B = fake_id`, 伪造id成功。 程序实现如下：

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import re
import requests

url = 'http://test.loc/cbc.php'

clipher_hex = ('89b52bac0331cb0b393c1ac828b4ee0f07861f030a8a3dc4'
               'b6e786f473b52182000a0d4ce2145994573a92d257a514d1')

plain_id = '1'
clipher_indexs = []
for i in range(len(clipher_hex)/2):
    clipher_text = list(clipher_hex.decode('hex'))
    c = clipher_text[i]
    clipher_text[i] = plain_id
    new_clipher_hex = ''.join(clipher_text).encode('hex')
    resp = requests.get(url, params={'a': new_clipher_hex})
    new_plain_id = resp.content[3]
    if new_plain_id == c and resp.content[4] == '<':
        # 有可能出现 c == palin_id 的情况，所以这里可能有多组结果
        print i, c.encode('hex'), c
        clipher_indexs.append(i)

fake_id = '0'
for i in clipher_indexs:
    clipher_text = list(clipher_hex.decode('hex'))
    c = clipher_text[i]
    fake_c = chr(ord(c) ^ ord(plain_id) ^ ord(fake_id))
    clipher_text[i] = fake_c
    new_clipher_hex = ''.join(clipher_text).encode('hex')
    resp = requests.get(url, params={'a': new_clipher_hex})
    if resp.content[3] == fake_id and resp.content[4] == '<':
        print "make a={}, get id='0'".format(''.join(clipher_text).encode('hex'))
        break
```

得到结果:

```
make a=89b52bac0331cb0b393c1ac828b4ee0e07861f030a8a3dc4b6e786f473b52182000a0d4ce2145994573a92d257a514d1, get id='0'
```

### 应用条件

能获取明文信息，密文或IV可控
