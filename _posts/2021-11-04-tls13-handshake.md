---
layout: post
title: "tls1.3 handshake"
description: ""
category: 
tags: []
---
## Overview

这篇blog是关于tls1.3协议handshake部分的介绍和分析，主要内容来自于[rfc8446](https://datatracker.ietf.org/doc/html/rfc8446)，这里主要侧重tls1.3区别于tls1.2的部分，并尝试搞清楚tls1.3这些变更的设计动机。


tls1.3将握手概括为三个阶段
- key exchange阶段：双方通过ClientHello和ServerHello进行所有必要的协议加密材料交换和密码学参数协商，此阶段后的所有消息都是经过加密的，这也是tls1.3和tls1.2握手过程一个很显著且重要的区别。
- Server Parameters阶段： 该阶段Server通过EncryptedExtensions消息将server端一些非密码学参数扩展告诉client。此外还包括一个可选的CertificateRequest消息。和tls1.2一样，如果server端选择认证client，就需要此时发送这个消息给client。
- Authentication阶段：该阶段包括client和sever双方相互认证的所有消息，首先server端会发送Certificate消息将自己的证书传给client，并随即发送一个CertificateVerify消息来证明自己持有该证书，然后server完成握手发送一个Finished消息。client端这边如果被要求提供证书来认证身份的话首先也需要通过Certificate、CertificateVerify来发送证书、证明持有证书，最后client发送Finished消息完成握手。


<!--more-->

![tls1.3 handshake](/public/fig/tls1_3_handshake.png)

可以看出tls1.3在第一个rtt就开始进行密钥交换以及server端的身份认证了，一个rtt后就可以开始application data的传输了，之所以能做到这点，是因为tls1.3只保留(EC)DHE方式的密钥交换，其不依赖于server端证书。

此外tls1.3还允许server端在第一个rtt完成同时就先向client端传输加密的application data，对于应用层是http协议而言，这个特性可能用处不大。

### resumption and pre-shared key handshake

tls1.3废弃了1.2中的session resumption机制，采用更为的通用的PSK机制，PSK在原理上很类似于tls1.2的session ticket。一个带有PSK扩展的握手过程如下：
![tls1.3 PSK handshake](/public/fig/tls1_3_psk_handshake.png)

可以看出PSK的handshake过程也是1-rtt，但是相比于full handshake省去了身份认证的过程。另外PSK可以和(EC)DHE密钥交换结合，做到前向安全。

有关PSK机制的更多内容后面再介绍。

### 0-rtt

tls1.3另外一个重要的升级是支持0-rtt模式，所谓0-rtt就是client在第一个rtt基于PSK握手的同时带上加密的application data，server也是在第一个rtt完成的同时响应application data，协议称之为early data。0-rtt模式下加密application data的密钥仅能由PSK密钥派生，也没有server端握手信息参与，因此本身不能防止被重放。一个0-rtt的握手和数据交换过程如下：
![tls1.3 0-rtt handshake](/public/fig/tls1_3_0_rtt_handshake.png)


## Client Hello
```
   uint16 ProtocolVersion;
   opaque Random[32];

   uint8 CipherSuite[2];    /* Cryptographic suite selector */

   struct {
       ProtocolVersion legacy_version = 0x0303;    /* TLS v1.2 */
       Random random;
       opaque legacy_session_id<0..32>;
       CipherSuite cipher_suites<2..2^16-2>;
       opaque legacy_compression_methods<1..2^8-1>;
       Extension extensions<8..2^16-1>;
   } ClientHello;
```
1.3的ClientHello在设计上吸取了当年部署tls1.2时候教训，不采用此前tls基于ProtocolVersion的版本协商机制。此外由于需要兼容各种中间件，在协议结构上保持和tls1.2完全一致。体现在legacy_version设置为1.2的值，保留legacy_session_id字段，为了防止僵化，要求实现设置一个随机值。保留legacy_compression_methods字段。random和cipher_suites字段依旧生效，其他协议版本、加密材料、密钥参数、握手相关协商等都放到了扩展中。

### Extensions: 
- `supported_versions` 是强制存在的扩展, 为了避免tls1.2版本协商机制带来的问题，tls1.3采用的是在client hello的supported_versions扩展里带上自己所支持的versions，在server hello的supported_versions选择一个version。
    
- 由于去掉了rsa-based key exchange，只保留DHE-based key exchange，使得tls1.3在client hello里就开始进行密钥交换，通过`key_share`扩展将自己的DHE公钥发给server端。
    
- tls1.3中废弃了tls1.2的session resumption和session ticket机制，采用更为通过的pre_shared_key机制（可以视作session ticket的升级版）。具体来说client在hello消息里可以通过`pre_shard_key`扩展带上自己支持的key标识列表（可能是之前的tls session里生成的，也可能是其他外部方式预先配置的），同时通过`psk_key_exchange_modes`扩展指定支持的psk的使用方式（目前有两种方式：PSK-only和PSK-with-DHE）。server端选择也通过`pre_shard_key`扩展选择一个PSK，通过`psk_key_exchange_modes`选择一种mode，双端达成对PSK的一致。
    
- tls1.3还有个重要的改动是摒弃了之前异常复杂的各种算法排列组合形成的cipher-suite-list，转而采用各种算法各自正交协商，其中cipher_suites字段只是用来协商对称加密算法，并且只保留了AEAD系列算法。密钥交换算法只支持DHE-based算法，通过`supported_groups`扩展来协商用什么椭圆曲线， 通过`signature_algorithms`来协商签名算法。
    
#### 0-rtt

tls1.3支持0-rtt，即紧随clientHello后就开始发送Application data，0-rtt要求client在ClientHello里至少需要带上`early_data`扩展以及`pre_shared_key`扩展，`early_data`用于告知server接下来会有early application data，这些data使用的是`pre_shared_key`扩展中第一个PSK对应的密钥加密的（client_early_traffic_secret）。如果server端接受early data，那么也需要在消息里带上`early_data`扩展，告诉client端自己接受并处理early data，tls1.3规定server端的`early_data`在Server的EncryptedExtensions消息里带上。注意early data使用的密钥和后面正常的application data使用的密钥是不一样的，所以client在收到server的Finished消息后要切换密钥，并且发送一个EndOfEarlyData消息告知对段后续数据用正常密钥加密。

#### PSK机制
`pre-shared-key`扩展数据字段定义如下：
```
struct {        
    opaque identity<1..2^16-1>;        
    uint32 obfuscated_ticket_age;    
} PskIdentity;    

opaque PskBinderEntry<32..255>;    

struct {        
    PskIdentity identities<7..2^16-1>;        
    PskBinderEntry binders<33..2^16-1>;    
} OfferedPsks;    

struct {        
    select (Handshake.msg_type) {           
        case client_hello: OfferedPsks;            
        case server_hello: uint16 selected_identity;
    };    
} PreSharedKeyExtension;
```

- 首先对于client来说，在ClientHello的`pre-shared-key`扩展里提供了一个PSK的列表供server选择，对于server来说选择一个PSK在ServerHello扩展里返回给client
- PSK的identity相对于tls1.2 里的ticket更为通用，可以是server段在NewSessionTicket消息里生成告诉client的，也可以是双方在tls之外预先配置的。
- 每个PSK都对应一个binder，用于将PSK和当前handshake绑定，其实就是确认双方都只能PSK对应的密钥。binder由client生成，由server段校验。其计算和Finished消息类似，都是对handshake消息进行transcript hash，然后再计算hash的HMAC，区别在于使用的key不一样，binder使用的key是通过对应PSK关联的密钥派生得到的binder_key。binder的作用是client向server证明自己知道PSK identity对应的密钥是什么，是否和server端记录的一致（如果不一致，server端将验证binder失败）。如果不一样，说明此前session很可能出现了MITM攻击导致client记录了一个和server端不一致的密钥。
- 关于PSK还有个重要的概念就是Ticket age，是client视角下这个PSK ticket的age，即从收到这个ticket对应的NewSessionTicket消息到当前时间的duration，单位是ms。Server在通过NewSessionTicket消息生成ticket时候除了ticket本身外，还附带这个ticket的生存时间(ticket_lifetime)，client端不能使用超出生存时间的ticket。Client在pre-shared-key扩展里需要带上一个混淆版本的ticket age，就是那个obfuscated_ticket_age字段。首先为什么client需要带上这个ticket age给server？毕竟只要ticket里面编码了createtime，那么server就应该能计算出age（后面我们会知道这个ticket age可以用来在一定程度上防止0-rtt的重放攻击）。其次为什么需要混淆这个age？协议里给的解释是为了避免偷听者将resumption connection和之前的connection通过时间关联对应起来。混淆的方法是server在NewSessionTicket里面还会给一个随机生成的ticket_age_add字段，client通过给原始的age加上这个ticket_age_add作为obfuscated_ticket_age。关于这个Ticket age, Stackexchange的crypto区有类似的[问题](https://crypto.stackexchange.com/questions/60925/whats-the-ticket-age-add-and-obfuscated-ticket-age-used-for-in-tls-1-3)。TSL1.3的作者的Martin Thomson给出了比较详细的解释。
- 协议依然没有规定实现怎么去构造ticket，ticket在实现中，可能作为一个database的key，value里存储了对应的PSK状态信息（对应tls1.2的session cache）。也可能在ticket里直接编码了PSK状态信息（对应tls1.2里的session ticket）。无论怎样通过ticket，client和server都能恢复出PSK状态（至少包括resumption_secret），从而双方都能派生出PSK key。

## Server hello
```
 struct {
       ProtocolVersion legacy_version = 0x0303;    /* TLS v1.2 */
       Random random;
       opaque legacy_session_id_echo<0..32>;
       CipherSuite cipher_suite;
       uint8 legacy_compression_method = 0;
       Extension extensions<6..2^16-1>;
   } ServerHello;
```

- extensions: ServerHello MUST only include extensions which are required to establish the cryptographic context and negotiate the protocol version，意味着： All TLS 1.3 ServerHello messages MUST contain the "supported_versions" extension. Current ServerHello messages additionally contain either the "pre_shared_key" extension or the "key_share" extension, or both。 其他的扩展需求在单独的EncryptedExtensions中加密传给客户端
- 为了兼容中间件，HelloRetryRequest也会按照ServerHello形式发送，tls1.3通过将random设置为`HelloRetryRequest`的SHA-256值来区别是ServerHello还是HelloRetryRequest。


### downgrade protection mechanism
所谓的downgrade protection在tls1.3中有个正式的定义
```
The cryptographic parameters should be the same on both sides and should be the same as if the peers had been communicating in the absence of an attack

```

在tls1.2以及之前版本的tls协议存在多种downgrade攻击，比如FREAK、logjam等，攻击者在中间通过篡改handshake消息让双方协商出一个弱密钥，然后break掉master secret，从而能改篡改为了防止downgrade的Finished消息，以使得双方继续以被破解的密钥进行通信。另一方面，防止downgrade的另一个有效机制——签名算法在tls1.2以及之前并没有得到合理利用，整个握手过程的消息并没有签名保护（tls1.2只在ServerKeyExchange里用签名保护了DH公钥和random）。


因此tls1.3在downgrade这块一个关键的改动是签名覆盖整个握手消息（server 在CertificateVerify里面将整个handshake消息进行签名发给client端，如果有篡改，client终止连接）。这点阻隔了大部分的downgrade attacks，但是仅仅是这一点还不够，因为协议需要后向兼容的原因downgrade attack依旧存在。攻击者可以篡改握手让都支持tls1.3的client和server双方版本降级到tls1.2或之前上，然后再利用低版本的协议进行downgrade攻击。

因此tls1.3为了防止这个版本降级，在server random里嵌入了一个降级保护机制，具体来说，如果支持tls1.3的server被协商以低版本tls通信，会在random的最后8个字节里注入一个标识（如果协商的是tls1.2，注入`44 4F 57 4E 47 52 44 01`; 如果协商的是tls1.1，注入`44 4F 57 4E 47 52 44 00`），tls1.3的client会去检测random里有没有这个标识，如果有的话，表示存在版本降级攻击，中止连接。另外协议还建议最高支持tls1.2的client也去检测random，这样tls1.2的client也能够防止被降级到tls1.1或之前。至于为什么在random里面去注入这个降级标识呢，因为tls1.2和之前在ServerKeyExchange消息中有对random的签名保护，所以不用担心MITM偷偷修去掉random里的降级标识。


至此，tls1.3 key exchange阶段结束。

## Encrypted Extensions
Encrypted Extensions紧随Server Hello之后，这是tls1.3中首个加密的消息，加密采用的key基于server_handshake_traffic_secret派生得到。专门用于Server发送一些加密扩展的消息

## Certificate Request
和tls1.2中的同名消息类似，作用也一样。区别是
- 在tls1.3中这个消息还可以用于post-handshake authentication阶段，后面再说。
- tls1.2中这个消息的内容里包括一个server端支持的验证证书的签名算法列表和CA列表，在tls1.3中这两个分别在该消息的扩展中分别用`signature_algorithms`、`signature_algorithms_cert`和`"certificate_authorities`扩展给出。

至此，tls1.3 Server Parameters阶段结束。

## Certificate
和tls1.2的同名消息区别不大。tls1.3的Certificate消息还支持扩展，比如在扩展里将各证书的OCSP状态返回。

验证证书等内容不在tls协议标准范围内

## Certificate Verify
这个消息是tls1.3和之前一个重要升级的地方。在tls1.2中只有client端在提供了client 证书后才需要发送，server则不需要依赖这个消息来证明自己确实拥有Certificate消息中的证书（因为ServerKeyExchange消息server需要证书的私钥去签名DH公钥匙+random）。

在tls1.3中，Server端将也需要发送这个消息，一方面是为了证明自己拥有证书，另一方面通过对整个handshake消息计算transcript-hash，然后进行签名，保护握手免受FREAK、logjam这类MITM攻击。


## Finished
Finished消息是Authentication阶段的最后一个消息。作用和之前一样，区别不大。


## Post-Handshake Messages
TLS支持在handshake完成后，继续发送一些类型属于Handshake的消息，主要有：

### New Session Ticket Message
NewSessionTicket作用和tls1.2 session ticket对应的基本一致，都是用于server生成一个ticket给client作为PSK，以恢复后续握手。

### Post-Handshake Authentication
为什么会有这个机制？

```
- servers that have the ability to serve requests from multiple domains over the same connection but do not have a certificate that is simultaneously authoritative for all of them 
- servers that have resources that require client authentication to access and need to request client authentication after the connection has started
- clients that want to assert their identity to a server after a connection has been established
- clients that want a server to re-prove ownership of their private key during a connection
- clients that wish to ask a server to authenticate for a new domain not covered by the certificate included in the initial handshake

quote from http://www.watersprings.org/pub/id/draft-sullivan-tls-post-handshake-auth-00.html

```  

这里几个场景我认为很典型，首先对于server，可能有一些资源访问是需要认证client的，但是tls连接建立的一开始不需要认证client，这时候post-handshake认证就有用了。要支持Post-Handshake Authentication, Client首先要在ClientHello里带上`post_handshake_auth`表示自己支持Post-Handshake Authentication, 然后server就可以在handshake建立后的任意时候通过发送CertificateRequest消息来要求client验证身份。Client则需要回复依次回复Certificate, CertificateVerify,  Finished消息。

### Key and IV Update
KeyUpdate消息作用是让两端同步更新key/IV, 由一端发起，另一端ack


## Record protocol
对于没有进行加密保护前的Record层的结构和tls1.2一致，如下TLSPlaintext的定义所示：
```
enum {
    invalid(0),        
    change_cipher_spec(20),       
    alert(21),        
    handshake(22),        
    application_data(23),        
    (255)    
} ContentType;   

struct {        
    ContentType type;        
    ProtocolVersion legacy_record_version;        
    uint16 length;        
    opaque fragment[TLSPlaintext.length];    
} TLSPlaintext;
```


对于需要加密保护的record，相对于tls1.2，tls1.3对record层密码学保护做的最重要的改动是只保留AEAD加密。此外tls1.3选择将除了ClientHello、ServerHello外的握手消息都采用加密传输，为了前向兼容，这些加密的握手消息在record层的type都是application_data。因此对于需要加密保护的record来说，结构如下：
```

struct {        
    opaque content[TLSPlaintext.length];       
    ContentType type;        
    uint8 zeros[length_of_padding];    
} TLSInnerPlaintext;

struct {
    ContentType opaque_type = application_data; /* 23 */
    ProtocolVersion legacy_record_version = 0x0303; /* TLS v1.2 */
    uint16 length;
    opaque encrypted_record[TLSCiphertext.length];
} TLSCiphertext;
```

- 其中，encrypted_record是TLSInnnerPlaintext经过AEAD加密得到的结果。注意TLSInnerPlaintext中除了包含TLSPlaintext的fragment作为content之外，还包含实际的type，以及一个可选的padding，padding的目的在于混淆实际length。
- TLSCiphertext.length的计算看起来是aead encryt前就可以计算出来，就是TLSInnerPlaintext的总长度 + aead算法加密后的额外长度


### AEAD
aead 的additional_data构造、加密、解密过程如下：
```
additional_data = TLSCiphertext.opaque_type ||
                  TLSCiphertext.legacy_record_version ||    
                  TLSCiphertext.length

AEADEncrypted = AEAD-Encrypt(write_key, nonce, additional_data, plaintext)

plaintext of encrypted_record =        
                AEAD-Decrypt(peer_write_key, nonce, additional_data, AEADEncrypted)
```

可以看出，和tls1.2不同的是additional_data没有seq_num的参与，tls1.3将seq_num编码进pre_record的nonce中。nonce通过seq_num XOR (client_write_iv / server_write_iv) 得到。在tls1.2中nonce则有显式部分和隐式部分拼接构成。



## Key Schedule

相比于tls1.2自制的PRF，tls1.3采用的HKDF标准，HKDF标准中派生密钥分为两个部分：
- HKDF-Extract：用输入的原始密钥和salt生成一个伪随机密钥
- HKDF-Expand：将HKDF-Extract输出的伪随机密钥扩展成任意长度，
定义分别如下：

```
HKDF_Extract(salt, IKM) -> PRK
//salt： 可选的“盐”，如果不提供，则默认为0串
//IKM： 初始密钥材料
//PRK: 定长的伪随机密钥

HKDF_Expand(PRK, info, L) -> OKM
//PRK：HKDF_Extract的输出
//info：可选的上下文信息，默认是空字符串“”，当IKM被用于多种业务时，就可以用info来保证导出不一样的OKM
//L：指定输出的OKM的字节长度，不能超过255*HashLen
//OKM: 输出密钥材料

```


tls1.3在HKDF_Expand基础之上定义了Derive_Secret函数:

```
       HKDF-Expand-Label(Secret, Label, Context, Length) =
            HKDF-Expand(Secret, HkdfLabel, Length)

       Where HkdfLabel is specified as:

       struct {
           uint16 length = Length;
           opaque label<7..255> = "tls13 " + Label;
           opaque context<0..255> = Context;
       } HkdfLabel;

       Derive-Secret(Secret, Label, Messages) =
            HKDF-Expand-Label(Secret, Label,
                              Transcript-Hash(Messages), Hash.length)
```
Derive_Secret将当前handshake消息的Transcript-Hash作为HKDF_Expand的上下文。

整个key派生过程就是对两个原始密钥(PSK密钥和ECDHE协商出的密钥)使用HKDF-Extract、Derive-Secret两个进行密钥生成和派生过程。

```
             0
             |
             v
   PSK ->  HKDF-Extract = Early Secret
             |
             +-----> Derive-Secret(., "ext binder" | "res binder", "")
             |                     = binder_key
             |
             +-----> Derive-Secret(., "c e traffic", ClientHello)
             |                     = client_early_traffic_secret
             |
             +-----> Derive-Secret(., "e exp master", ClientHello)
             |                     = early_exporter_master_secret
             v
       Derive-Secret(., "derived", "")
             |
             v
   (EC)DHE -> HKDF-Extract = Handshake Secret
             |
             +-----> Derive-Secret(., "c hs traffic",
             |                     ClientHello...ServerHello)
             |                     = client_handshake_traffic_secret
             |
             +-----> Derive-Secret(., "s hs traffic",
             |                     ClientHello...ServerHello)
             |                     = server_handshake_traffic_secret
             v
       Derive-Secret(., "derived", "")
             |
             v
   0 -> HKDF-Extract = Main Secret
             |
             +-----> Derive-Secret(., "c ap traffic",
             |                     ClientHello...server Finished)
             |                     = client_application_traffic_secret_0
             |
             +-----> Derive-Secret(., "s ap traffic",
             |                     ClientHello...server Finished)
             |                     = server_application_traffic_secret_0
             |
             +-----> Derive-Secret(., "exp master",
             |                     ClientHello...server Finished)
             |                     = exporter_secret
             |
             +-----> Derive-Secret(., "res master",
                                   ClientHello...client Finished)
                                   = resumption_secret

```


- 首先是仅由PSK密钥派生出的Early Secret。通过Early Secret扩展出：binder_key（前面说了为了生成PSK的binder，确认双方都有PSK密钥）、client_early_traffic_secret（为了加解密0-rtt中的early data）、early_exporter_secret（？）
- 然后是涉及ECDHE密钥和上一轮Early Secret派生出的Handshake secret。通过Handshake secret扩展出client和server端分别对应的client_handshake_traffic_secret和server_handshake_traffic_secret
- 最后再将上一轮的Handshake secret派生出Main Secret。通过Main Secret扩展出client_application_traffic_secret_0、server_application_traffic_secret_0、exporter_secret、resumption_secret


有了各阶段的secret后，再派生出对应的对称密钥和iv
```
   [sender]_write_key = HKDF-Expand-Label(Secret, "key", "", key_length)
   [sender]_write_iv  = HKDF-Expand-Label(Secret, "iv", "", iv_length)

   [sender] denotes the sending side.  The value of Secret for each
   record type is shown in the table below.

       +-------------------+---------------------------------------+
       | Record Type       | Secret                                |
       +-------------------+---------------------------------------+
       | 0-RTT Application | client_early_traffic_secret           |
       |                   |                                       |
       | Handshake         | [sender]_handshake_traffic_secret     |
       |                   |                                       |
       | Application Data  | [sender]_application_traffic_secret_N |
       +-------------------+---------------------------------------+

```



## 0-rtt 和防重放手段
TLS在0-rtt模式下协议本身不能防重放，需要应用层措施来防止重放，这里列举几个措施

### 一次性的PSK

顾名思义PSK只能使用一次，这需要server端存储所有颁发的并且还在生效中的PSK，一旦某个PSK使用了，从存储里删除这个PSK，0-rtt建连时候检查有没有对应的PSK，有的话接受连接和数据，没有则中断或者fallback到完全握手

### 记录Client hello

server端记录此前一个时间窗口内所有带PSK的Client Hello的唯一标识（可能是random字段也可以是PSK binder字段），如果后面带有同样标识的Client hello过来，直接拒绝掉或者fallback到完全握手，Client Hello标识没有出现过才接受连接和数据。时间窗口大小设置和PSK的有效期保持一致就行。

### 检查PSK新鲜度(Freshness)

提法有些奇怪，协议原文叫做Freshness Checks。说的是server通过高效检查ClientHello里PSK是否足够新鲜来决定是否接受PSK。之前在介绍pre_shared_key扩展时留了个疑问，为啥扩展里需要带上client视角的ticket_age，这时候就用上了。如果攻击者复制了某个0-rtt的ClientHello过一段时间后重放这个ClientHello，server可以很简单地通过校验这个client视角下的ticket_age是否和server视角的ticket age匹配，如果差距很大的话（比如server视角下ticket age要比client视角的ticket_age大很多，表示不新鲜了）就可以拒绝接受这个PSK。server可以通过将ticket的createtime编码在ticket做到无状态计算这个server视角的ticket_age。



## Some References

https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/

https://www.ietf.org/id/draft-ietf-tls-rfc8446bis-02.html

https://datatracker.ietf.org/doc/html/rfc8446

https://blog.cloudflare.com/why-tls-1-3-isnt-in-browsers-yet/

https://tls13.ulfheim.net/