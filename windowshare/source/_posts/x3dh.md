---
title: X3DH(迪菲赫尔曼三重扩展密钥协商协议)
date: 2019.03.27 22:14:55
categories:
- 翻译
- 端对端协议
tags:
- x3dh
---


迪菲赫尔曼三重扩展密钥协商协议

本文是 [Signal 协议](https://en.wikipedia.org/wiki/Signal_Protocol)中使用的「迪菲赫尔曼三重扩展密钥协议」的中文翻译（部分保持原文不翻译），[英文原文地址](https://www.signal.org/docs/specifications/x3dh/) 

------
[comment:]:[TOC]

<!-- toc -->

# 1. 引言

本文档描述了“X3DH”（或“迪菲赫尔曼三重扩展”）密钥协商协议。 X3DH在基于公钥相互认证的两方之间建立共享密钥。 X3DH提供前向安全性和加密可否性。

X3DH设计用于这样的异步场景，其中一个用户（“Bob”）处于脱机状态但已向服务器发布了一些信息。 另一个用户（“Alice”）想要使用该信息将加密数据发送给Bob，并且还建立用于将来通信的共享密钥。

# 2. 提要

## 2.1. X3DH 参数

使用X3DH的应用程序必须使用以下几个参数：

| 名称    | 定义                                       |
| :------ | :----------------------------------------- |
| *curve* | X25519 or X448                             |
| *hash*  | 256或512位散列函数（例如SHA-256或SHA-512） |
| *info*  | 标识应用程序的ASCII字符串                  |

例如：应用程序可以选择curve=X25519，hash=SHA-512，info="MyProtocol"

应用程序必须另外定义编码函数Encode（PK）以将X25519或X448公钥PK编码为字节序列。 推荐采用一些单字节常量表示curve的类型，和具体的u-coordinate[[1](https://www.signal.org/docs/specifications/x3dh/#ref-rfc7748)]进行little-endian 编码组成的编码方式进行编码。

## 2.2. 加密符号

X3DH 使用以下符号:

- **X** || **Y** 表示字节序列 **X** 和 **Y** 的连接。***(连接具体指什么，没明白？)***

- **DH(PK1, PK2)** 表示以公钥PK1和PK2所表示的密钥对作为输入，通过**椭圆曲线迪菲赫尔曼函数**输出的共享加密字节序列。**椭圆曲线迪菲赫尔曼函数**根据不同的curve参数，采用[[1](https://www.signal.org/docs/specifications/x3dh/#ref-rfc7748)]中表示的X25519或X448函数进行处理。

- **Sig(PK, M)** 表示一个给字节序列M进行XEdDSA签名和用公钥PK验证，并通过使用与PK相符合的私钥对M进行签名来创建的字节序列。 XEdDSA的签名和验证函数具体描述可在[[2](https://www.signal.org/docs/specifications/x3dh/#ref-xeddsa)]中查看。

- *KDF(KM)*

   表示通过HKDF算法得到的32位输出结果 [[3](https://www.signal.org/docs/specifications/x3dh/#ref-rfc5869)]

   输入:

  - *HKDF input key material* = *F* || *KM*, 其中*KM*是包含秘钥元素的输入字节序列，当curve使用X25519时，*F*是包含32位0xFF的字节序列，当curve使用X448时，F是包含57位0xFF的字节序列。*F* 用于与XEdDSA加密域的隔离。
  - *HKDF salt* = 长度等于hash长度且以0补齐不足长度的字节序列
  - *HKDF info* = *info* 参数查看 [第2.1节](https://www.signal.org/docs/specifications/x3dh/#x3dh-parameters).

## 2.3. 角色

X3DH协议包含了3个角色： **Alice（爱丽丝）**, **Bob（鲍勃）**, **server（服务器）**.

- **Alice** 希望发送给Bob一些用于加密的初始数据，同时建立可能用于双向交互的共享秘钥。
- **Bob** 希望允许像Alice这样的人与他建立共享秘钥，并发送加密数据。但是，当Alice尝试这样做的时候，Bob可能处于离线状态。为了实现这一点，Bob与某个server建立了关系。
- **server** 可以存储来自Alice给Bob的消息，让Bob可以延迟获取。server也可以让Bob发布一些数据通过server提供给像Alice这样的角色。具体服务器可信任的量在第4.7节中讨论。

在某些系统中，服务器角色可能在多个实体之间划分，但为简单起见，我们假设一个服务器为Alice和Bob提供上述功能。

## 2.4. 密钥

X3DH 使用以下椭圆曲线公开密钥：

| 名称   | 定义                  |
| :----- | :-------------------- |
| *IKA*  | Alice's identity key  |
| *EKA*  | Alice's ephemeral key |
| *IKB*  | Bob's identity key    |
| *SPKB* | Bob's signed prekey   |
| *OPKB* | Bob's one-time prekey |

所有公钥都有相应的私钥，但为了简化描述，我们将重点关注公钥。

X3DH协议运行中使用的公钥必须全部采用X25519格式，或者必须全部采用X448格式，具体取决于curve参数[[1](https://www.signal.org/docs/specifications/x3dh/#ref-rfc7748)]。

每个角色都有一个长期的身份公开密钥 (*IKA* for Alice, *IKB* for Bob)。

Bob还有一个签名的预密钥SPKB，它将定期更改，以及一组一次性预密钥OPKB，它们分别用于单个X3DH协议运行。 （“Prekeys”之所以如此命名是因为它们本质上是Bob在Alice开始协议运行之前发布到服务器的协议消息）。

在每个协议运行期间，Alice使用公钥EKA生成新的临时密钥对。

在成功执行协议之后，Alice和Bob将共享一个32字节的密钥SK。 该密钥可以在某些post-X3DH安全通信协议中使用，但需遵守[第4节](#4)中的安全考虑。

# 3. X3DH协议

## 3.1. 概述

X3DH有三个阶段：

1. Bob将他的身份密钥和预密钥发布到服务器。
2. Alice从服务器获取“prekey bundle”，并使用它向Bob发送初始消息。
3. Bob接收并处理Alice的初始消息。

以下部分解释了这些阶段。

## 3.2. 发布密钥

Bob向服务器发布一组椭圆曲线公钥，包含：

 -  Bob的 身份密钥*IKB*
 -  Bob的 已签名的预密钥*SPKB*
 -  Bob的 预密钥签名 *Sig(IKB,Encode(SPKB))*
 -  Bob的 一组一次性预密钥（OPKB1，OPKB2，OPKB3，......）

Bob只需要将他的身份密钥上传到服务器一次。 但是，Bob可以在其他时间上传新的一次性预密钥（例如，当服务器通知Bob服务器的一次性预密钥存储变低时）。

Bob还将以某个间隔（例如，每周一次或每月一次）上传新签名的预密钥和预密钥签名。 新签名的预密钥和预密钥签名将替换以前的值。

在上传新签名的预密钥之后，Bob可以将对应于先前签名的预密钥的私钥保持一段时间，用于处理使用该密钥产生的传输延迟的消息。 最终，Bob应删除此私钥保证前向安全（当Bob使用它们接收消息时，将删除一次性预密钥私钥;请参阅[第3.4节](#3.4)）。

## 3.3. 发送初始消息

为了与Bob执行X3DH密钥协议，Alice联系server获取包含以下值得"prekey bundle":

- Bob的 身份密钥 *IKB*
- Bob的 已签名预密钥 *SPKB*
- Bob的 预密钥签名 *Sig(IKB, Encode(SPKB))*
- (可选) Bob的 一次性预密钥 *OPKB*

server应该提供Bob的一次性预密钥中的一个(如果存在)，然后删除它。如果服务器上Bob的所有一次性预密钥都被删除了，预密钥簇(prekey bundle)中将不包含一次性预密钥。

如果验证失败，Alice会验证预密钥签名并中止协议。 Alice然后使用公钥* EKA *生成临时密钥对。

如果预密钥簇中不包含一次性预密钥，进行如下计算：

```
DH1 = DH(IKA, SPKB)
DH2 = DH(EKA, IKB)
DH3 = DH(EKA, SPKB)
SK = KDF(DH1 || DH2 || DH3)
```

如果预密钥簇确实包含一次性预密钥，则修改计算包含额外的*DH*:
```
DH4 = DH(EKA, OPKB)
SK = KDF(DH1 || DH2 || DH3 || DH4)
```

下图显示了密钥之间的* DH *计算。 注意* DH1 *和* DH2 *提供相互认证，而* DH3 *和* DH4 *提供前向保密。

![](https://signal.org/docs/specifications/x3dh/X3DH.png)



​										图1:DH1...DH4

计算*SK*之后，Alice 删除他的临时私钥和*DH*输出

Alice然后计算包含双方身份信息的“关联数据”字节序列* AD *：
```
AD = Encode(IKA) || Encode(IKB)
```

Alice可以选择将附加信息附加到* AD *，例如Alice和Bob的用户名，证书或其他识别信息。

Alice然后给Bob发送一条初始信息，其中包含：

- Alice的身份密钥 *IKA*
- Alice的临时密钥 *EKA*
- Alice使用Bob的哪个预密钥的标志符
- 使用某种AEAD加密方案加密的初始密文 [[4](https://www.signal.org/docs/specifications/x3dh/#ref-aead)] ，使用* AD *作为关联数据并使用加密密钥，该加密密钥是* SK *或由* SK *键入的某些加密PRF的输出。

初始密文通常是某些post-X3DH通信协议中的第一个消息。 换句话说，这个密文通常有两个角色，作为一些post-X3DH协议中的第一个消息，并作为Alice的X3DH初始消息的一部分。

After sending this, Alice may continue using *SK* or keys derived from *SK* within the post-X3DH protocol for communication with Bob, subject to the security considerations in [Section 4](https://www.signal.org/docs/specifications/x3dh/#security-considerations).

在发送之后，Alice可以继续使用* SK *或在post-X3DH协议中从* SK *派生的密钥与Bob进行通信，但需遵守[第4节](#4)中的安全考虑因素。

## 3.4. 接收初始消息

收到Alice的初始消息后，Bob从消息中检索Alice的身份密钥和短暂密钥。 Bob还加载了他的身份私钥，以及与Alice使用的任何已签名的预密钥和一次性预密钥（如果有的话）相对应的私钥。

使用这些密钥，Bob重复上一节中的* DH *和* KDF *计算以导出* SK *，然后删除* DH *值。

Bob然后使用* IKA *和* IKB *构造* AD *字节序列，如上一节所述。 最后，Bob尝试使用* SK *和* AD *解密初始密文。 如果初始密文无法解密，则Bob中止协议并删除* SK *。

如果初始密文解密成功，则Bob完成协议。 Bob删除任何已经使用的一次性预密钥私钥，以保证前向安全性。 然后Bob可以继续使用* SK *或在post-X3DH协议中从* SK *派生的密钥与Alice进行通信，但需遵守[第4节](#4.)中的安全考虑因素

# 4. 安全性考虑

## 4.1. 认证

在X3DH密钥协议之前或之后，双方可以通过一些经过身份验证的通道比较其身份公钥* IKA *和* IKB *。 例如，他们可以手动比较公钥指纹，或者通过扫描QR码。 执行此操作的方法超出了本文档的范围。

如果未执行身份验证，则各方不会收到与其通信对象的加密保证。

## 4.2. 协议重播

如果Alice的初始消息不使用一次性预密钥，则可以将其重播给Bob并且他将接受它。 这可能导致Bob认为Alice已经多次向他发送了相同的消息（或消息集）。

为了缓解这种情况，post-X3DH协议可能希望基于来自Bob的新的随机输入快速协商Alice的新加密密钥。 这是基于Diffie-Hellman的棘轮方案的典型行为[[5](https://www.signal.org/docs/specifications/x3dh/#ref-doubleratchet)]。

Bob可以尝试其他缓解措施，例如维护观察到的消息的黑名单，或者更快地替换旧的签名预密钥。 分析这些缓解措施超出了本文档的范围。

## 4.3. 重播和密钥重用

上一节中讨论的重播的另一个结果是成功重播的初始消息将导致Bob在不同的协议运行中导出相同的* SK *。

因此，任何post-X3DH协议必须在Bob发送加密数据之前随机化加密密钥。 例如，Bob可以使用基于DH的棘轮协议将* SK *与新生成的* DH *输出结合起来以获得随机加密密钥[[5](https://www.signal.org/docs/specifications/x3dh/＃REF-doubleratchet)]。

无法随机化Bob的加密密钥可能会导致灾难性的密钥重用。

## 4.4. 可否性


X3DH不会向Alice或Bob提供他们的通信内容或他们通信事实的可发布的加密证明。

与OTR协议[[6](https://www.signal.org/docs/specifications/x3dh/#ref-otr)]中一样，在某些情况下，已经泄露了Alice或Bob的合法私钥的第三方可以 提供一个似乎在Alice和Bob之间的通信记录，并且只能由一些其他方创建，该另一方也可以访问Alice或Bob的合法私钥（即Alice或Bob本身，或其他已经泄露他们的人。 私钥）。

如果任何一方在协议执行期间与第三方合作，他们将能够向这样的第三方提供他们的通信证明。 对“在线”拒绝的这种限制似乎是异步设置所固有的[[7](https://www.signal.org/docs/specifications/x3dh/#ref-unger)]。

## 4.5. 签名


可能很容易观察到* DH *计算实现了相互认证和前向保密，并省略了预密钥签名。 然而，这将允许“弱前向保密”攻击：恶意服务器可以向Alice提供带有伪造预密钥的预密钥捆绑包，并且稍后损害Bob的* IKB *以计算* SK *。

或者，可能很想用来自身份密钥的签名替换基于DH的相互认证（即* DH1 *和* DH2 *）。 但是，这会降低拒绝性，增加初始消息的大小，并且如果临时或预密钥私钥被泄露，或者签名方案被破坏，则会增加损害。

## 4.6. 密钥妥协

虽然使用临时密钥和预密钥可以对缓解一些影响，但是一方的私钥妥协仍然会对安全性产生灾难性的影响。

某一方的身份私钥妥协就允许他人冒充该方。

根据许多考虑因素，某一方的预密钥私钥的妥协可能会影响较旧或较新的* SK *值的安全性。

对所有可能的妥协方案进行全面分析超出了本文档的范围，但是对一些合理方案的部分分析如下：

- 如果一次性预密钥用于协议运行，那么在蔚来某个时间对Bob的身份密钥和预密钥私钥进行妥协，将不会损害旧的* SK *(假设删除了* OPKB *的私钥)。
- 如果一次性预密钥*没有*用于协议运行，那么来自该协议运行的* IKB *和* SPKB *的私钥的妥协将损害之前计算的* SK *。 频繁更换已签名的预密钥可以减轻这种影响，使用post-X3DH棘轮协议可以用新密钥快速替代* SK *，保证前向安全性 [[5](https://www.signal.org/docs/specifications/x3dh/#ref-doubleratchet)].
- 预密钥私钥的妥协可以实现延伸到未来的攻击，例如被动计算* SK *值，以及模仿被入侵方的任意其他方（“密钥泄露假冒”）。这些攻击是可能的，直到受感染的一方取代他在服务器上的受损预设（在被动攻击的情况下）;或者删除他已妥协的签名预密钥的私钥（在密钥泄露假冒的情况下）。

## 4.7. 服务器可信

恶意服务器可能导致Alice和Bob之间的通信失败（例如，通过拒绝传递消息）。

如果Alice和Bob在[第4.1节](#4.1)中提及的相互验证，则服务器可用的唯一额外攻击是拒绝分发一次性预密钥，使* SK *的前向保密取决于签名的预密钥的生命周期（如上一节所述）。

如果一方恶意消耗另一方的一次性预密钥，也可能发生初始前向保密的减少，因此服务器应该尝试阻止这种情况，例如， 获取prekey bundle的速率限制。

## 4.8. 身份绑定

在[第4.1节](#4.1)中提及的身份验证不一定能防止”身份错误绑定“或”未知密钥共享“攻击。

这导致攻击者（“Charlie”）错误地将Bob的身份密钥指纹呈现给Alice作为他(Charlie's)自己拥有的身份密钥，然后将Alice的初始消息转发给Bob，或者错误地将Bob的联系信息呈现为他自己的。

这样做的结果是Alice认为她正在向Charlie发送初始消息，因为她实际上正在向Bob发送消息。

为了使这更加困难，各方可以将更多识别信息包括在* AD *中，或者将更多识别信息散列到指纹中，例如用户名，电话号码，真实姓名或其他识别信息。 Charlie将被迫对这些额外的价值撒谎，这可能很难。

然而，没有办法可靠地防止Charlie对其他值进行撒谎，并且在协议中包含更多身份信息通常会在隐私，灵活性和用户界面方面带来权衡。 对这些权衡的详细分析超出了本文档的范围。

# 5. 知识产权

特此将此文档置于公共领域。

# 6. 致谢

X3DH协议由Moxie Marlinspike和Trevor Perrin开发。

底层的”三重DH“密钥协议由Caroline Kudla和Kenny Paterson在 [[8](https://www.signal.org/docs/specifications/x3dh/#ref-kudla2005)]中提出, 扩展了之前的”双重DH“（又名”协议4“）来自Simon Blake-Wilson等的密钥协议 [[9](https://www.signal.org/docs/specifications/x3dh/#ref-blakewilson1997)]. 在[[10](https://www.signal.org/docs/specifications/x3dh/#ref-eckpfs)] and [[11](https://www.signal.org/docs/specifications/x3dh/#ref-joint)]作品中讨论了将签名与隐式认证密钥协议使用的问题。

感谢Mike Hamburg关于身份绑定和椭圆曲线公钥的讨论。

感谢Nik Unger和Matthew Green关于拒绝的讨论。

感谢Matthew Green，Tom Ritter，Joseph Bonneau和Benedikt Schmidt的编辑反馈。


# 7. 参考

[1] A. Langley, M. Hamburg, and S. Turner, “Elliptic Curves for Security.” Internet Engineering Task Force; RFC 7748 (Informational); IETF, Jan-2016. <http://www.ietf.org/rfc/rfc7748.txt>

[2] T. Perrin, “The XEdDSA and VXEdDSA Signature Schemes,” 2016. <https://whispersystems.org/docs/specifications/xeddsa/>

[3] H. Krawczyk and P. Eronen, “HMAC-based Extract-and-Expand Key Derivation Function (HKDF).” Internet Engineering Task Force; RFC 5869 (Informational); IETF, May-2010. <http://www.ietf.org/rfc/rfc5869.txt>

[4] P. Rogaway, “Authenticated-encryption with Associated-data,” in Proceedings of the 9th ACM Conference on Computer and Communications Security, 2002. <http://web.cs.ucdavis.edu/~rogaway/papers/ad.pdf>

[5] T. Perrin, “The Double Ratchet Algorithm (work in progress),” 2016.

[6] N. Borisov, I. Goldberg, and E. Brewer, “Off-the-record Communication, or, Why Not to Use PGP,” in Proceedings of the 2004 aCM workshop on privacy in the electronic society, 2004. <http://doi.acm.org/10.1145/1029179.1029200>

[7] N. Unger and I. Goldberg, “Deniable Key Exchanges for Secure Messaging,” in Proceedings of the 22Nd aCM sIGSAC conference on computer and communications security, 2015. <http://doi.acm.org/10.1145/2810103.2813616>

[8] C. Kudla and K. G. Paterson, “Modular Security Proofs for Key Agreement Protocols,” in Advances in Cryptology - ASIACRYPT 2005: 11th International Conference on the Theory and Application of Cryptology and Information Security, 2005. <http://www.isg.rhul.ac.uk/~kp/ModularProofs.pdf>

[9] S. Blake-Wilson, D. Johnson, and A. Menezes, “Key agreement protocols and their security analysis,” in Crytography and Coding: 6th IMA International Conference Cirencester, UK, December 17–19, 1997 Proceedings, 1997. <http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.25.387>

[10] C. Cremers and M. Feltz, “One-round Strongly Secure Key Exchange with Perfect Forward Secrecy and Deniability.” Cryptology ePrint Archive, Report 2011/300, 2011. <http://eprint.iacr.org/2011/300>

[11] J. P. Degabriele, A. Lehmann, K. G. Paterson, N. P. Smart, and M. Strefler, “On the Joint Security of Encryption and Signature in EMV.” Cryptology ePrint Archive, Report 2011/615, 2011. <http://eprint.iacr.org/2011/615>

















