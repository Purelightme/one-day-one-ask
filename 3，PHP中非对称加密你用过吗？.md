Rsa是目前用的最多的非对称加密算法。

### 原理

* [RSA算法原理（1）](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)
* [RSAS算法原理（2](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)

#### 限制

RSA算法本身要求加密内容也就是明文长度m必须0<m<密钥长度n。如果小于这个长度就需要进行padding，因为如果没有padding，就无法确定解密后内容的真实长度，字符串之类的内容问题还不大，以0作为结束符，但对二进制数据就很难，因为不确定后面的0是内容还是内容结束符。而只要用到padding，那么就要占用实际的明文长度，于是实际明文长度需要减去padding字节长度。我们一般使用的padding标准有NoPPadding、OAEPPadding、PKCS1Padding等，其中PKCS#1建议的padding就占用了11个字节。

这样，对于1024长度的密钥。128字节（1024bits）-减去11字节正好是117字节，但对于RSA加密来讲，padding也是参与加密的，所以，依然按照1024bits去理解，实际的明文只有117字节了。

建议解决办法是：先将明文用对称加密算法加密（DES,AES……），再用RSA加密；解密时先RSA解密，再对称解密。

### PHP中使用

#### 生成密钥

```openssl genrsa -out 2048_rsa_private_key.pem 2048 ```

```openssl rsa -in 2048_rsa_private_key.pem -pubout -out 2048_rsa_public_key.pem ```

java 开发使用的 PKCS8 格式转换命令：

```openssl pkcs8 -topk8 -inform PEM -in 2048_rsa_private_key.pem -outform PEM -nocrypt -out 2048_rsa_private_key_pkcs8.pem```

#### 对称加密

```
openssl_encrypt()
openssl_decrypt() 
```

### 对称加密中的Padding

[对称加密算法的PKCS5和PKCS7填充]([https://zhiwei.li/text/2009/05/17/%E5%AF%B9%E7%A7%B0%E5%8A%A0%E5%AF%86%E7%AE%97%E6%B3%95%E7%9A%84pkcs5%E5%92%8Cpkcs7%E5%A1%AB%E5%85%85/](https://zhiwei.li/text/2009/05/17/对称加密算法的pkcs5和pkcs7填充/))

code：

```php
//AES加密
function encrypt_aes($encrypt,$key){

    $size = mcrypt_get_block_size(MCRYPT_RIJNDAEL_128,MCRYPT_MODE_CBC);

    $encrypt = pkcs5Pad($encrypt,$size);

    $iv = "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0";

    $passcrypt = mcrypt_encrypt(MCRYPT_RIJNDAEL_128, $key, $encrypt, MCRYPT_MODE_CBC,$iv);

    $encode = base64_encode($passcrypt);

    return $encode;

}

//pkcs5加密
function pkcs5Pad($text,$blocksize){
  
    $pad = $blocksize-(strlen($text)%$blocksize);

    return $text.str_repeat(chr($pad),$pad);
}

//AES解密
function decrypt_aes($str,$key) {
  
    $str =  base64_decode($str);

    $iv = "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0";

    $or_data = mcrypt_decrypt(MCRYPT_RIJNDAEL_128, $key, $str, MCRYPT_MODE_CBC,$iv);

    $str = pkcs5Unpad($or_data);

    return $str;
}

//pcks5解密
function pkcs5Unpad($text) {

    $pad = ord($text{strlen($text)-1});

    if ($pad>strlen($text))

        return false;

    if (strspn($text,chr($pad),strlen($text)-$pad)!=$pad)

        return false;

    return substr($text,0,-1*$pad);

}
```

#### 非对称加密

```php
openssl_pkey_get_private(file_get_contents($path)); 
openssl_pkey_get_public(file_get_contents($path)); 
openssl_private_encrypt() - Encrypts data with private key
openssl_private_decrypt() - Decrypts data with private key
openssl_public_encrypt() - Encrypts data with public key
openssl_public_decrypt() - Decrypts data with public key
```

#### 签名与验证

```php
openssl_sign()
openssl_verify()
//支付场景用得比较多
```

### 密钥库文件格式【Keystore】

```
格式     扩展名    描述及特点
JKS     .jks .ks  【Java Keystore】 SUN提供密钥库的Java实现版本        密钥库和私钥用不同的密码进行保护
JCEKS   .jce      【JCE Keystore】 SUN JCE提供密钥库的JCE实现版本      相对于JKS安全级别更高，保护Keystore私钥时采用TripleDES
PKCS12  .p12 .pfx 【PKCS #12】 个人信息交换语法标准                    1、包含私钥、公钥及其证书 2、密钥库和私钥用相同密码进行保护
BKS     .bks      【Bouncycastle Keystore】 密钥库的BC实现版本         基于JCE实现
UBER    .ubr      【Bouncycastle UBER Keystore】 密钥库的BC更安全实现版本
```

### 证书文件格式【Certificate】

```
格式     扩展名           描述及特点
DER     .cer .crt .rsa   【ASN .1 DER】用于存放证书           不含私钥、二进制
PKCS7   .p7b .p7r        【PKCS #7】加密信息语法标准           p7b以树状展示证书链，不含私钥；p7r为CA对证书请求签名的回复，只能用于导入。
CMS     .p7c .p7m .p7s   【Cryptographic Message Syntax】    p7c只保存证书，p7m：signature with enveloped data，p7s：时间戳签名文件
PEM     .pem             【Printable Encoded Message】       PEM是【Privacy-Enhanced Mail】广泛运用于密钥管理，一般基于base 64编码。
PKCS10  .p10 .csr        【PKCS #10】公钥加密标准【Certificate Signing Request】 证书签名请求文件，ASCII文件，CA签名后以p7r文件回复。
SPC     .pvk .spc        【Software Publishing Certificate】 微软公司特有的双证书文件格式，经常用于代码签名，其中pvk用于保存私钥，spc用于保存公钥。
```

### 非对称加密代码实例

```php
<?php

$public_key = '-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCOxol7sQui5Wkprc1+ducNql61
Vma7cQynd+xKmxjodY94GCcQC/OroxoPTXmGI8y0ziGfcpvTVYqAp/ek+kxiKkLN
IXipYT/0UNjqQtJ+9LyHfBU/COnfe73on8OkWJUwUgao8Sx6NCuXSpi/KRmjk8bp
9WrBUCtcgH9VcQeC2QIDAQAB
-----END PUBLIC KEY-----';

$private_key = '-----BEGIN RSA PRIVATE KEY-----
MIICXAIBAAKBgQCOxol7sQui5Wkprc1+ducNql61Vma7cQynd+xKmxjodY94GCcQ
C/OroxoPTXmGI8y0ziGfcpvTVYqAp/ek+kxiKkLNIXipYT/0UNjqQtJ+9LyHfBU/
COnfe73on8OkWJUwUgao8Sx6NCuXSpi/KRmjk8bp9WrBUCtcgH9VcQeC2QIDAQAB
An9bG/mIgM9ggQTgk/ExNt2vN9p93XZrbIoyVA7pu4PjvDii3sbI08BlUoEI5cS+
GqXkAOwuAlH5ilbvNylFTKLlCwEH26aesBURSb7JmtWemiyG6dABJbzCIE4Xwu2b
+GWQ3ohYHNB0+cdbyz9JodTbwEOwb1SMaevUlLJgdWmBAkEApMHBmBmboDnOebwn
uttb7y4v55DQ0J5X3t2E+uSSoAj9v2531lXTOkWRJfyyfVAWe/euNNAyd3u5QLJb
R+Oy0QJBAN3YZlttsMRXBVlUM2FhoUTYydaoCCOdl39JwEf0bVbPmHEaxxnguq4G
jQJ3mC+Er8uLKP3UtSRO886S64daAYkCQQCSz0xQ2lDAn4ILG8xTRvBO2ts4/uPz
YYVvQ/khD9hP3nMtx6PlS6ji/eZu8ROjcl/2qyeCTBsMOSVELyoDjzRhAkEAgw4g
CcsXLiYqZsczQ0gluUJImqLRjBjBMtUi3l8raKli6Q5kqIj2P3BnRRnZsdi08Y3Y
PXu3NyfdKB/rPB6T4QJBAIJ+vrw7zT4TzqqKMuVPC8k1uY+d1WX14NWeosbB6Bj4
5I8SspnHyUD/F/SI60MXfkkmOcAGHM1v0uWeeH4Glmk=
-----END RSA PRIVATE KEY-----';


$pi_key =  openssl_pkey_get_private($private_key);//这个函数可用来判断私钥是否是可用的，可用返回资源id Resource id
$pu_key = openssl_pkey_get_public($public_key);//这个函数可用来判断公钥是否是可用的

$data = str_repeat("1",117);//原始数据
$encrypted = "";
$decrypted = "";


$res = openssl_private_encrypt($data,$encrypted,$pi_key);//私钥加密
var_dump($res);
$encrypted = base64_encode($encrypted);//加密后的内容通常含有特殊字符，需要编码转换下，在网络间通过url传输时要注意base64编码是否是url安全的
echo $encrypted."\n";
openssl_public_decrypt(base64_decode($encrypted),$decrypted,$pu_key);
echo $decrypted;
```

### 快钱中使用的函数库(只能用于php7.1及以下版本，因为mcrpt扩展在7.2+已移除)

```php
<?php
/**
 * Created by PhpStorm.
 * User: purelightme
 * Date: 2019/2/19
 * Time: 09:24
 */

//私钥加密RSAwithSHA1
function crypto_seal_private($original_data){
    $pfx_path =  config('kuaiqian.'.config('kuaiqian.env').'.pfx_path');//商户PFX证书地址
    $key_password = config('kuaiqian.'.config('kuaiqian.env').'.key_password');//证书密码
    $pfx_file='file://'.$pfx_path;
    $pfx=file_get_contents($pfx_file);
    $certs=array();
    openssl_pkcs12_read($pfx,$certs,$key_password);
    $privkey=$certs['pkey'];
    openssl_sign($original_data,$signature,$privkey,OPENSSL_ALGO_SHA1);
    return base64_encode($signature);
}

//公钥加密OPENSSL_PKCS1_PADDING
function crypto_seal_pubilc($original_data){
    $pubkey_path = config('kuaiqian.'.config('kuaiqian.env').'.public_key_path');//快钱公钥地址
    $pubkey_file=$pubkey_path;
    $pubkey=file_get_contents("file://".$pubkey_file);
    openssl_public_encrypt($original_data,$signature,$pubkey,OPENSSL_PKCS1_PADDING);
    return base64_encode($signature);
}

//私钥解密OPENSSL_PKCS1_PADDING
function crypto_unseal_private($digitalEnvelope){
    $pfx_path =  config('kuaiqian.'.config('kuaiqian.env').'.pfx_path');//商户PFX证书地址
    $key_password = config('kuaiqian.'.config('kuaiqian.env').'.key_password');//证书密码
    $pfx_file='file://'.$pfx_path;
    $pfx=file_get_contents($pfx_file);
    $certs=array();
    openssl_pkcs12_read($pfx,$certs,$key_password);
    $privkey=$certs['pkey'];
    $digitalEnvelope = base64_decode($digitalEnvelope);
    $res = openssl_private_decrypt($digitalEnvelope,$receiveKey,$privkey,OPENSSL_PKCS1_PADDING);
//    dd($res);
    return $receiveKey;
}

//AES加密
function encrypt_aes($encrypt,$key){
//    dd($key);
    $size = mcrypt_get_block_size(MCRYPT_RIJNDAEL_128,MCRYPT_MODE_CBC);
    $encrypt = pkcs5Pad($encrypt,$size);
    $iv = "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0";
    $passcrypt = mcrypt_encrypt(MCRYPT_RIJNDAEL_128, $key, $encrypt, MCRYPT_MODE_CBC,$iv);
    $encode = base64_encode($passcrypt);
    return $encode;
}



//pkcs5加密
function pkcs5Pad($text,$blocksize){
    $pad = $blocksize-(strlen($text)%$blocksize);
    return $text.str_repeat(chr($pad),$pad);
}


//AES解密
function decrypt_aes($str,$key) {
    $str =  base64_decode($str);
    $iv = "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0";
    $or_data = mcrypt_decrypt(MCRYPT_RIJNDAEL_128, $key, $str, MCRYPT_MODE_CBC,$iv);
    $str = pkcs5Unpad($or_data);
    return $str;
}


//pcks5解密
function pkcs5Unpad($text) {
    $pad = ord($text{strlen($text)-1});
    if ($pad>strlen($text))
        return false;
    if (strspn($text,chr($pad),strlen($text)-$pad)!=$pad)
        return false;
    return substr($text,0,-1*$pad);
}

//公钥验签
function crypto_unseal_pubilc($signedData,$receiveData){
    $pubkey_path = config('kuaiqian.'.config('kuaiqian.env').'.public_key_path');//快钱公钥地址
    $MAC = base64_decode($signedData);
    $fp = fopen($pubkey_path, "r");
    $cert = fread($fp, 8192);
    fclose($fp);
    $pubkeyid = openssl_get_publickey($cert);
    $ok = openssl_verify($receiveData, $MAC, $pubkeyid);
    return $ok;
}
```

### 最后

慕课网php加密算法课程：[慕课网php加密解密](https://www.imooc.com/learn/1133)

```2019-07-10```