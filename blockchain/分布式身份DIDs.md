[Understanding Decentralized IDs (DIDs)](https://medium.com/@adam_14796/understanding-decentralized-ids-dids-839798b91809)

在 [Internet Identity Workshop (IIW)](https://www.internetidentityworkshop.com/) 上，分布式身份开始引起了我的注意，在这儿大约有30%的与会人员是关于DIDs的。我感觉像是这个会议的后来者，但是在Wiki上找不到关于DIDs或任何关于它的历史或他们是什么时候开始的，或许我并不是迟到者。

DIDs 提议使用区块链来让注册身份，该身份可以让用户用来证明他们自己。这篇文章用来记录我在过去几个月我对DIDs了解了什么 - 既可以帮助其他对DIDs感兴趣的团体，也可以让指出我的失误，进而学习到更多。需要澄清的是，我既不支持也不反对DIDs；我也不支持或反对区块链和/或政府。我现在不得不说，因为所有这些事情似乎都是非常热门的话题，非常两极分化，很多人都有非常强烈的观点。我希望下面的内容是客观和中立的。

本文从DIDs、DID文档、可验证声明和DID认证的概述开始——基本上列出了该技术是如何工作的。然后，本课程将探讨DIDs的经济，试图理解他们提出要解决什么问题、为谁解决问题以及他们如何着手解决这些问题。

# DID技术概览

我不会深入探究DIDs的架构——部分原因是我不是这方面的专家，但部分原因是我认为我可以在高层次上解释它，而不会用所有的比特和字节来迷惑人们。W3C的[Decentralized Identifiers (DIDs)](https://w3c-ccg.github.io/did-spec)目前的版本是0.10—所以我确信它是非常灵活的，并且随时会发生变化。

首先，用户可以在任何时间以任何理由创建一个新的DID。一个DID包含两件事：一个唯一标识符和一个相关联的DID文档。惟一标识符类似于“`did:example:123456789abcdefghi`”，可以安全地将其看作查找did文档的惟一ID。[DID Document](https://w3c-ccg.github.io/did-spec/#x4-did-documents)是一个[JSON-LD](https://json-ld.org/)对象，它存储在某个中心位置，以便容易查找。do应该是“[持久的和不可修改的](https://w3c-ccg.github.io/did-spec/#x3-6-did-persistence)”，这样它就不会受到所有者以外的任何人的影响。

DID文档必须包含一个DID。它要可能包含：

* 何时被创建的时间戳
* 该DID文档是有效的加密证明
* 加密公钥列表
* DID可用于身份验证的方法列表
* DID可以被使用的服务列表
* 任意数量的外部定义的扩展

> 示例2： 最小的自理DID文档
>
> ```json
> {
>     "@context": "https://www.w3.org/ns/did/v1",
>   "id": "did:example:123456789abcdefghi",
>   "authentication": [{
>     
>     "id": "did:example:123456789abcdefghi#keys-1",
>     "type": "Ed25519VerificationKey2018",
>     "controller": "did:example:123456789abcdefghi",
>     "publicKeyBase58": "H3C2AVvLMv6gmMNam3uVAjZpfkcJCwDwnZn6z3wXmqPV"
>   }],
>   "service": [{
>     
>     "id":"did:example:123456789abcdefghi#vcs",
>     "type": "VerifiableCredentialService",
>     "serviceEndpoint": "https://example.com/vc/"
>   }]
> }
> ```

注意，在DID文档中没有 个人身份信息：没有名字，没有地址，没有手机号码。这是通过可验证的声明（ Verifiable Claims）实现的，我将在下一节中解释。但在此之前，我还要提两件事。
首先，似乎大多数人都对DID文档中的公钥感到兴奋——我没有听到太多关于如何使用其他字段或为什么人们对它们感到兴奋的讨论。

其次，我在DID的格式上撒了谎。它实际上是" `did`: " + <method> + ": " <method-specific-identifier>。在上面，我的方法是`example`，方法-specific-identifier是“`123456789abcdefghi`”。实际上，该方法定义了如何/在何处查找DID。目前有[9 注册的方法(methods)](https://w3c-ccg.github.io/did-method-registry/)，包括比特币、以太坊、Sovrin、IPFS和Veres One——都是区块链(或者更准确地说是“分布式账本技术”)。DID中的方法将定义您使用的是哪一个，并且您需要知道使用DID来获得DID文档的协议。如果你听说过“解析器”，这只是一个用于查找DID的协议的通用术语，“通用解析器(universal resolver)”是可以通过任何方法查找DID的东西。

# 可验证的声明（ Verifiable Claims）

围绕DIDs的大多数话题和他们如何使用就是可验证声明。一般的，可验证声明有三个部分：

* **主题(Subject)**：可能是人，也可以是公司，宠物或任何可以被描述的东西
* **发行者(Issuer)**：可能是某种组织，比如DMV，一个大学或我银行等
* **声明(Claim)**：可以是任何语句，一般的例子是人或包括“是否大于21见证”，“在这个地方生活..." 或”叫这个名字"。可以是任何描述性语句。

所以可验证声明是当一些发行者对主题做出的声明。“可验证”部分是它是值得依赖的和不可篡改的，因为它是被发行者的密钥签名的。

> 示例3：一个简单的可验证声明
>
> ```
> {
>   // 设置 context, 将建立我们使用的特殊的术语，例如 '发行者' 和 'alumniOf'.
>   "@context": [
>     "https://www.w3.org/2018/credentials/v1",
>     "https://www.w3.org/2018/credentials/examples/v1"
>   ],
>   // 指定凭证的身份 
>   "id": "http://example.edu/credentials/1872",
>   // 凭证的类型，它声明了在凭证中将有什么数据
>   "type": ["VerifiableCredential", "AlumniCredential"],
>   // 发行凭证的实体
>   "issuer": "https://example.edu/issuers/565049",
>   // 凭证的签发日期
>   "issuanceDate": "2010-01-01T19:73:24Z",
>   // 凭证主题的声明
>   "credentialSubject": {
>     // 凭证的唯一主题的标识
>     "id": "did:example:ebfeb1f712ebc6f1c276e12ec21",
>     // 断言凭证的唯一主题
>     "alumniOf": {
>       "id": "did:example:c276e12ec21ebfeb1f712ebc6f1",
>       "name": [{
>         "value": "Example University",
>         "lang": "en"
>       }, {
>         "value": "Exemple d'Université",
>         "lang": "fr"
>       }]
>     }
>   },
>   // 凭证不可篡改的数字证明，更多细节可以看该部分结尾的 注意
>   "proof": {
>     // 用来生成签名的密钥签名套件
>     "type": "RsaSignature2018",
>     // 签名创建的日期
>     "created": "2017-06-18T21:19:10Z",
>     // 证明的目的
>     "proofPurpose": "assertionMethod",
>     // 可以验证签名的公钥标识
>     "verificationMethod": "https://example.edu/issuers/keys/1",
>     // 数字签名的值
>     "jws": "eyJhbGciOiJSUzI1NiIsImI2NCI6ZmFsc2UsImNyaXQiOlsiYjY0Il19..TCYt5X
>       sITJX1CxPCT8yAV-TVkIEq_PbChOMqsLfRoPsnsgw5WEuts01mq-pQy7UJiN5mgRxD-WUc
>       X16dUEMGlv50aqzpqh4Qktb3rk-BuQy72IFLOqV0G_zS245-kronKb78cPN25DGlcTwLtj
>       PAYuNzVBAh4vGHSrQyHUdBBPM"
>   }
> }
> ```

> **注意**
>
> 对上机使用的 `proof` 感兴趣的实现者可以在 [§ 4.7 Proofs (Signatures)](https://w3c.github.io/vc-data-model/#proofs-signatures) 了解更多，阅读下面的规范：数据证明链接[[LD-PROOFS](https://w3c.github.io/vc-data-model/#bib-ld-proofs)]，数据签名的链接 [[LD-SIGNATURES](https://w3c.github.io/vc-data-model/#bib-ld-signatures)]。2018 RSA 签名套件 [[LDS-RSA2018](https://w3c.github.io/vc-data-model/#bib-lds-rsa2018)]，JSON 网页签名（JWS)编码载荷选项[[RFC7797](https://w3c.github.io/vc-data-model/#bib-rfc7797)]。证明机制列表可以访问可验证凭证扩展注册[[VC-EXTENSION-REGISTRY](https://w3c.github.io/vc-data-model/#bib-vc-extension-registry)]

**文档太老，与W3C规范不一致了**