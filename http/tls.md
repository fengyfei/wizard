### SSL/TLS

1. TLS 定义

  **SSL**(Secure Sockets Layer) 安全套接层，是一种安全协议，经历了 SSL 1.0、2.0、3.0 版本后发展成了标准安全协议 - **TLS**(Transport Layer Security) 传输层安全性协议。TLS 有 1.0 (RFC 2246)、1.1(RFC 4346)、1.2( RFC 5246)、1.3(1.3 目前处于草案阶段)版本。

  TLS 在实现上分为**记录层**和**握手层**两层，其中握手层又含四个子协议: 握手协议（handshake protoco 协议（change cipher spec protocol）、应用数据协议（application data protocol）和警报协议（alert protocol）

![image](https://github.com/fengyfei/wizard/raw/master/http/images/tls.png)

2. HTTPS = HTTP over TLS.

  只需配置浏览器和服务器相关设置开启 TLS，即可实现 HTTPS，TLS 高度解耦，可装可卸，与上层高级应用层协议相互协作又相互独立。

![image](https://github.com/fengyfei/wizard/raw/master/http/images/https.png)

3. 加密

TLS/SSL的功能实现主要依赖于三类基本算法：散列函数 Hash、对称加密和非对称加密，其利用非对称加密实现身份认证和密钥协商，对称加密算法采用协商的密钥对数据加密，基于散列函数验证信息的完整性。

![image](https://github.com/fengyfei/wizard/raw/master/http/images/encrypt.png)

TLS的基本工作方式是，客户端使用非对称加密与服务器进行通信，实现身份验证并协商对称加密使用的密钥，然后对称加密算法采用协商密钥对信息以及信息摘要进行加密通信，不同的节点之间采用的对称密钥不同，从而可以保证信息只能通信双方获取。

例如，在 HTTPS 协议中，客户端发出请求，服务端会将公钥发给客户端，客户端验证过后生成一个密钥再用公钥加密后发送给服务端，成功后建立连接。通信过程中客户端将请求数据用得到的公钥加密后发送，服务端用私钥解密；服务端用客户端给的密钥加密响应报文，回复客户端，客户端再用存好的相同的密钥解密。

4. 记录层

记录协议负责在传输连接上交换的所有底层消息，并且可以配置加密。每一条 TLS 记录以一个短标头开始。标头包含记录内容的类型(或子协议)、协议版本和长度。原始消息经过分段(或者合并)、压缩、添加认证码、加密转为 TLS 记录的数据部分。

![image](https://github.com/fengyfei/wizard/raw/master/http/images/message.png)

- 碎片(Fragmentation)

  记录层将信息块分割成携带 2^14 字节 (16KB) 或更小块的数据的 TLSPlaintext 记录。

  记录协议传输由其他协议层提交给它的不透明数据缓冲区。如果缓冲区超过记录的长度限制（2^14），记录协议会将其切分成更小的片段。反过来也是可能的，属于同一个子协议的小缓冲区也可以组合成一个单独的记录。

```
struct { 
  uint8 major, minor; 
} ProtocolVersion;

enum { 
  change_cipher_spec(20),
  alert(21),
  handshake(22), 
  application_data(23), (255) 
} ContentType; 

struct {
  ContentType type; // 用于处理封闭片段的较高级协议
  ProtocolVersion version; // 使用的安全协议版本
  uint16 length; // TLSPlaintext.fragment的长度（以字节为单位），不超过 2^14
  opaque fragment[TLSPlaintext.length]; // 透明的应用数据，被视为独立的块，由类型字段指定的较高级协议处理
} TLSPlaintext;
```

- 记录压缩和解压缩(Record compression and decompression)

  压缩算法将 TLSPlaintext 结构转换为 TLSCompressed 结构。如果定义CompressionMethod 为 null 表示不压缩

```
struct { 
  ContentType type; // same as TLSPlaintext.type
  ProtocolVersion version; // same as TLSPlaintext.version 
  uint16 length; // TLSCompressed.fragment 的长度，不超过 2^14 + 1024
  opaque fragment[TLSCompressed.length]; 
} TLSCompressed;
```

- 空或标准流加密(Null or standard stream cipher)

  流加密（BulkCipherAlgorithm）将 TLSCompressed.fragment 结构转换为流 TLSCiphertext.fragment 结构

```
stream-ciphered struct { 
opaque content[TLSCompressed.length]; 
opaque MAC[CipherSpec.hash_size]; 
} GenericStreamCipher;
```
MAC 产生方法如下：

```
HMAC_hash(MAC_write_secret, seq_num + TLSCompressed.type + 
TLSCompressed.version + TLSCompressed.length + 
TLSCompressed.fragment));
```
seq_num（记录的序列号）、hash（SecurityParameters.mac_algorithm 指定的哈希算法）

> 注意，MAC 是在加密之前计算的。流加密加密整个块，包括 MAC。对于不使用同步向量(例如RC4)的流加密，从一个记录结尾处的流加密状态仅用于后续数据包。如果 CipherSuite 是TLS_NULL_WITH_NULL_NULL，则加密由身份操作(数据未加密，MAC大小为零，暗示不使用MAC)组成。TLSCiphertext.length 是 TLSCompressed.length 加上 CipherSpec.hash_size。

- CBC块加密

  块加密（如RC2或DES），将 TLSCompressed.fragment 结构转换为块 TLSCiphertext.fragment 结构

```
block-ciphered struct { 
  opaque content[TLSCompressed.length]; 
  opaque MAC[CipherSpec.hash_size]; 
  uint8 padding[GenericBlockCipher.padding_length]; 
  uint8 padding_length;
} GenericBlockCipher;
```
padding: 添加的填充将明文长度强制为块密码块长度的整数倍。填充可以是长达 255 字节的任何长度，只要满足 TLSCiphertext.length 是块长度的整数倍。长度大于需要的值可以阻止基于分析交换信息长度的协议攻击。填充数据向量中的每个 uint8 必须填入填充长度值(即 padding_length)。

padding_length: 填充长度应该使得 GenericBlockCipher 结构的总大小是加密块长度的倍数。 合法值范围从零到255（含）。**该长度指定 padding_length 字段本身除外的填充字段的长度。**

加密块的数据长度（TLSCiphertext.length）是 TLSCompressed.length，CipherSpec.hash_size 和 padding_length 的总和加一

> 示例: 如果块长度为 8 字节，压缩内容长度（TLSCompressed.length）为 61 字节，MAC 长度为 20 字节，则填充前的长度为 82 字节（padding_length 占 1 字节）。
因此，为了使总长度为块长度(8 字节)的偶数倍，模 8 的填充长度必须等于6，所以填充长度可以为 6，14，22 等。如果填充长度是需要的最小值，比如6，填充将为 6 字节，每个块都包含值 6。因此，块加密之前的 GenericBlockCipher 的最后 8 个八位字节将为xx 06 06 06 06 06 06 06，其中 xx 是 MAC 的最后一个八位字节。
> ```
> XX  - 06 06 06 06 06 06 - 06
> MAC -     padding[6]    - padding_length
> ```

- 记录有效载荷保护(Record payload protection)

  加密和 MAC 功能将 TLSCompressed 结构转换为 TLSCiphertext。记录的MAC还包括序列号，以便可以检测到丢失，额外或重复的消息。
  
```
struct { 
  ContentType type; // same
  ProtocolVersion version; // same
  uint16 length; // TLSCiphertext.fragment 的长度，不超过 2^14 + 2048
  select (CipherSpec.cipher_type) { 
    case stream: GenericStreamCipher; 
    case block: GenericBlockCipher; 
  } fragment; // TLSCompressed.fragment 的加密形式，带有MAC
} TLSCiphertext;
```

> 注意
这里提到的都是先 MAC 再加密，是基于 RFC 2246 的方案 (TLS 1.0) 写的。但新的方案选择先加密再 MAC，这种替代方案中，首先对明文和填充进行加密，再将结果交给MAC算法。这可以保证主动网络攻击者不能操纵任何加密数据。

- 密钥计算(Key calculation)

  记录协议需要一种算法，从握手协议提供的安全性参数生成密钥、[IV](https://zh.wikipedia.org/wiki/%E5%88%9D%E5%A7%8B%E5%90%91%E9%87%8F) 和 MAC secret.
  主机密信息(Master secret): 在连接中的两个对等体之间共享一个48字节的密钥
  客户随机数(client random): 由客户端提供的32字节值
  服务器随机数(server random): 由服务器提供的32字节值

5. 握手层

- 握手协议的职责是生成通信过程所需的共享密钥和进行身份认证。这部分使用无密码套件，为防止数据被窃听，通过公钥密码或 Diffie-Hellman 密钥交换技术通信。
- 密码规格变更协议，用于密码切换的同步，是在握手协议之后的协议。握手协议过程中使用的协议是“不加密”这一密码套件，握手协议完成后则使用协商好的密码套件。
- 警告协议，当发生错误时使用该协议通知通信对方，如握手过程中发生异常、消息认证码错误、数据无法解压缩等。
- 应用数据协议，通信双方真正进行应用数据传输的协议，传送过程通过 TLS 应用数据协议和 TLS 记录协议来进行传输。

握手是TLS协议中最精密复杂的部分。在这个过程中，通信双方协商连接参数，并且完成身 份验证。根据使用的功能的不同，整个过程通常需要交换 6~10 条消息。根据配置和支持的协议扩展的不同，交换过程可能有许多变种。在使用中经常可以观察到以下三种流程：(1) 完整的握手， 对服务器进行身份验证；(2) 恢复之前的会话采用的简短握手；(3) 对客户端和服务器都进行身份验证的握手。

握手协议消息的标头信息包含消息类型（1字节）和长度（3字节），余下的信息则取决于消息类型：

```
struct {
  HandshakeType msg_type;
  uint24 length;
  HandshakeMessage message;
} Handshake;
```

#### 5.1 完整握手

每一个 TLS 连接都会以握手开始。如果客户端此前并未与服务器建立会话，那么双方会执行一次完整的握手流程来协商 TLS 会话。握手过程中，客户端和服务器将进行以下四个主要步骤:

- 交换各自支持的功能，对需要的连接参数达成一致
- 验证出示的证书，或使用其他方式进行身份验证
- 对将用于保护会话的共享主密钥达成一致
- 验证握手消息并未被第三方团体修改

下面介绍最常见的握手规则，一种在不需要身份验证的客户端与需要身份验证的服务器之间的握手

![image](https://github.com/fengyfei/wizard/raw/master/http/images/full-handshake.png)

5.1.1 **ClientHello**

这条消息将客户端的功能和首选项传送给服务器。

![image](https://github.com/fengyfei/wizard/raw/master/http/images/wireshark-clienthello.png)

- Version: 协议版本（protocol version）指示客户端支持的最佳协议版本
- Random: 一个 32 字节数据，28字节是随机生成的(图中的 Random Bytes)；剩余的4字节包含额外的信息，与客户端时钟有关(图中使用的是 GMT Unix Time)。在握手时，客户端和服务器都会提供随机数，客户端的暂记作 random_C (用于后续的密钥的生成)。这种随机性对每次握手都是独一无二的，在身份验证中起着举足轻重的作用。它可以防止[重放攻击](https://zh.wikipedia.org/wiki/%E9%87%8D%E6%94%BE%E6%94%BB%E5%87%BB)，并确认初始数据交换的完整性。
- Session ID: 在第一次连接时，会话ID（session ID）字段是空的，这表示客户端并不希望恢复某个已存在的会话。典型的会话 ID 包含 32 字节随机生成的数据，一般由服务端生成通过 ServerHello 返回给客户端。
- Cipher Suites: 密码套件（cipher suite）块是由客户端支持的所有密码套件组成的列表，该列表是按优先级顺序排列的
- Compression: 客户端可以提交一个或多个支持压缩的方法。默认的压缩方法是null，代表没有压缩
- Extensions: 扩展（extension）块由任意数量的扩展组成。这些扩展会携带额外数据

5.1.2 **ServerHello**

是将服务器选择的连接参数传送回客户端。

![image](https://github.com/fengyfei/wizard/raw/master/http/images/wireshark-serverhello.png)

这个消息的结构与 ClientHello 类似，只是每个字段只包含一个选项，其中包含服务端的 random_S 参数(用于后续的密钥协商)。服务器无需支持客户端支持的最佳版本。如果服务器不支持与客户端相同的版本，可以提供某个其他版本以期待客户端能够接受

5.1.3 **Certificate**

典型的 Certificate 消息用于携带服务器 X.509 [证书链](https://zh.wikipedia.org/wiki/%E4%BF%A1%E4%BB%BB%E9%8F%88)。服务器必须保证它发送的证书与选择的算法套件一致。比方说，公钥算法与套件中使用的必须匹配。除此以外，一些密钥交换算法依赖嵌入证书的特定数据，而且要求证书必须以客户端支持的算法签名。所有这些都表明服务器需要配置多个证书（每个证书可能会配备不同的证书链）。

![image](https://github.com/fengyfei/wizard/raw/master/http/images/wireshark-certificate.png)

Certificate消息是可选的，因为并非所有套件都使用身份验证，也并非所有身份验证方法都需要证书。更进一步说，虽然消息默认使用X.509证书，但是也可以携带其他形式的标志；一些套件就依赖[PGP密钥](https://zh.wikipedia.org/wiki/PGP)

5.1.4 **ServerKeyExchange**

携带密钥交换的额外数据。消息内容对于不同的协商算法套件都会存在差异。在某些场景中，服务器不需要发送任何内容，
服务器密钥交换消息仅在服务器证书消息（如果发送）不包含足够的数据以允许客户端交换预主secret（premaster secret）时才由服务器发送。
比如基于 DH 的证书，公钥不被证书中包含，需要单独发送

![image](https://github.com/fengyfei/wizard/raw/master/http/images/wireshark-serverhellodone.png)

5.1.5 **ServerHelloDone**

表明服务器已经将所有预计的握手消息发送完毕。在此之后，服务器会等待客户端发送消息。

5.1.6 **verify certificate**

客户端验证证书的合法性，如果验证通过才会进行后续通信，否则根据错误情况不同做出提示和操作，合法性验证包括如下:
- 证书链的可信性 trusted certificate path;
- 证书是否吊销 revocation，有两类方式 - 离线 CRL 与在线 OCSP，不同的客户端行为会不同;
- 有效期 expiry date，证书是否在有效时间范围;
- 域名 domain，核查证书域名是否与当前的访问域名匹配;

5.1.7 **ClientKeyExchange**

![image](https://github.com/fengyfei/wizard/raw/master/http/images/wireshark-clientkeychange.png)

合法性验证通过之后，客户端计算产生随机数字的预主密钥（Pre-master），并用证书公钥加密，发送给服务器并携带客户端为密钥交换提供的所有信息。这个消息受协商的密码套件的影响，内容随着不同的协商密码套件而不同。

此时客户端已经获取全部的计算协商密钥需要的信息：两个明文随机数 random_C 和 random_S 与自己计算产生的 Pre-master，计算得到协商密钥;
```
enc_key=Fuc(random_C, random_S, Pre-Master)
```

5.1.8 **ChangeCipherSpec**

通知服务器后续的通信都采用协商的通信密钥和加密算法进行加密通信

> 注意
ChangeCipherSpec 不属于握手消息，它是另一种协议，只有一条消息，作为它的子协议进行实现。

5.1.9 **Finished (Encrypted Handshake Message)**

Finished消息意味着握手已经完成。消息内容将加密，以便双方可以安全地交换验证整个握手完整性所需的数据。

这个消息包含 verify_data 字段，它的值是握手过程中所有消息的散列值。这些消息在连接两端都按照各自所见的顺序排列，并以协商得到的主密钥计算散列。这个过程是通过一个伪随机函数（pseudorandom function，PRF）来完成的，这个函数可以生成任意数量的伪随机数据。
两端的计算方法一致，但会使用不同的标签（finished_label）：客户端使用 client finished，而服务器则使用 server finished。

```
verify_data = PRF(master_secret, finished_label, Hash(handshake_messages))
```
因为 Finished 消息是加密的，并且它们的完整性由协商 MAC 算法保证，所以主动网络攻击者不能改变握手消息并对 vertify_data 的值造假。在 TLS 1.2 版本中，Finished消息的长度默认是 12 字节（96位），并且允许密码套件使用更长的长度。在此之前的版本，除了 SSL 3 使用 36 字节的定长消息，其他版本都使用 12 字节的定长消息。

5.1.10 **Server**

服务器用私钥解密加密的 Pre-master 数据，基于之前交换的两个明文随机数 random_C 和 random_S，计算得到协商密钥: enc_key=Fuc(random_C, random_S, Pre-Master);
计算之前所有接收信息的 hash 值，然后解密客户端发送的 verify_data_C，验证数据和密钥正确性;

5.1.11 **change_cipher_spec**

![image](https://github.com/fengyfei/wizard/raw/master/http/images/wireshark-serverchangecipher.png)

服务端验证通过之后，服务器同样发送 change_cipher_spec 以告知客户端后续的通信都采用协商的密钥与算法进行加密通信（图中多了一步 New Session Ticket，此为会话票证，会在会话恢复中解释）;

5.1.12 **Finished (Encrypted Handshake Message)**

服务器也结合所有当前的通信参数信息生成一段数据 (verify_data_S) 并采用协商密钥 session secret (enc_key) 与算法加密并发送到客户端;

5.1.13 **握手结束**

客户端计算所有接收信息的 hash 值，并采用协商密钥解密 verify_data_S，验证服务器发送的数据和密钥，验证通过则握手完成;

5.1.14 **加密通信**

开始使用协商密钥与算法进行加密通信。

![image](https://github.com/fengyfei/wizard/raw/master/http/images/wireshark-applicationdata.png)

#### 5.2 客户端身份验证

尽管可以选择对任意一端进行身份验证，但人们几乎都启用了对服务器的身份验证。如果服 务器选择的套件不是匿名的，那么就需要在 Certificate 消息中跟上自己的证书。

![image](https://github.com/fengyfei/wizard/raw/master/http/images/mutualAuthentication.png)

相比之下，服务器通过发送 CertificateRequest 消息请求对客户端进行身份验证。消息中列 出所有可接受的客户端证书。作为响应，客户端发送自己的 Certificate 消息（使用与服务器发 送证书相同的格式），并附上证书。此后，客户端发送 CertificateVerify 消息，证明自己拥有对应的私钥。

只有已经过身份验证的服务器才被允许请求客户端身份验证。基于这个原因，这个选项也被称为相互身份验证（mutual authentication）。

5.2.1 **CertificateRequest**

在 ServerHello 的过程中发出，请求对客户端进行身份验证，并将其接受的证书的公钥 和签名算法传送给客户端。

它也可以选择发送一份自己接受的证书颁发机构列表，这些机构都用其可分辨名称来表示: 
```
struct {
  ClientCertificateType certificate_types;
  SignatureAndHashAlgorithm supported_signature_algorithms; 
  DistinguishedName certificate_authorities;
} CertificateRequest;
```

5.2.2 **CertificateVerify**

在 ClientKeyExchange 的过程中发出，证明自己拥有的私钥与之前发送的客户端证书中的公钥匹配。消息中包含一条到这一步为止的所有握手消息的签名：
```
struct {
  Signature handshake_messages_signature;
} CertificateVerify;
```

#### 5.3 会话恢复

最初的会话恢复机制是，在一次完整协商的连接断开时，客户端和服务器都会将会话的安全参数保存一段时间。希望使用会话恢复的服务器为会话指定唯一的标识，称为会话 ID(Session ID)。服务器在 ServerHello 消息中将会话 ID 发回客户端。

希望恢复早先会话的客户端将适当的 Session ID 放入 ClientHello 消息，然后提交。服务器如果愿意恢复会话，就将相同的 Session ID 放入 ServerHello 消息返回，接着使用之前协商的主密钥生成一套新的密钥，再切换到加密模式，发送 Finished 消息。
客户端收到会话已恢复的消息以后，也进行相同的操作。这样的结果是握手只需要一次网络往返。

Session ID 由服务器端支持，协议中的标准字段，因此基本所有服务器都支持，服务器端保存会话 ID 以及协商的通信信息，占用服务器资源较多。

![image](https://github.com/fengyfei/wizard/raw/master/http/images/simple-handshake.png)

用来替代服务器会话缓存和恢复的方案是使用会话票证（Session ticket）。使用这种方式，除了所有的状态都保存在客户端（与HTTP Cookie的原理类似）之外，其消息流与服务器会话缓存是一样的。
其思想是服务器取出它的所有会话数据（状态）并进行加密(密钥只有服务器知道)，再以票证的方式发回客户端。在接下来的连接中，客户端恢复会话时在 **ClientHello 的扩展字段** session_ticket 中携带加密信息将票证提交回服务器，由服务器检查票证的完整性，解密其内容，再使用其中的信息恢复会话。
这种方法有可能使扩展服务器集群更为简单，因为如果不使用这种方式，就需要在服务集群的各个节点之间同步会话。
Session ticket 需要服务器和客户端都支持，属于一个扩展字段，占用服务器资源很少。

> 警告
> 会话票证破坏了 TLS 安全模型。它使用票证密钥加密的会话状态并将其暴露在线路上。有些实现中的票证密钥可能会比连接使用的密码要弱。如果票证密钥被暴露，就可以解密连接上的全部数据。因此，使用会话票证时，票证密钥需要频繁轮换。

### References

- [RFC 2246 - The TLS Protocol Version 1.0](https://www.ietf.org/rfc/rfc2246.txt)
- [NPN/ALPN - wiki](https://zh.wikipedia.org/wiki/%E5%BA%94%E7%94%A8%E5%B1%82%E5%8D%8F%E8%AE%AE%E5%8D%8F%E5%95%86)
- [MAC (Message authentication code) - 消息认证码](https://zh.wikipedia.org/wiki/%E8%A8%8A%E6%81%AF%E9%91%91%E5%88%A5%E7%A2%BC)
- [PKI (Public Key Infrastructure)- 公开密钥基础设施](https://zh.wikipedia.org/wiki/%E5%85%AC%E9%96%8B%E9%87%91%E9%91%B0%E5%9F%BA%E7%A4%8E%E5%BB%BA%E8%A8%AD)
- [TLS - wiki](https://zh.wikipedia.org/wiki/%E5%82%B3%E8%BC%B8%E5%B1%A4%E5%AE%89%E5%85%A8%E6%80%A7%E5%8D%94%E5%AE%9A)
- [SSL/TLS in detail](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc785811(v=ws.10))
- [HTTPS加密协议详解](https://www.wosign.com/faq/faq2016-0309-02.htm)
- 《HTTPS 权威指南-在服务器和 web 应用上部署 SSL/TLS 和 PKI》

