# 零知识证明学习资源汇总

零知识证明技术是现代密码学三大基础之一，由 S.Goldwasser、S.Micali 及 C.Rackoff 在 20 世纪 80 年代初提出。早期的零知识证明由于其效率和可用性等限制，未得到很好的利用，仅停留在理论层面。直到近年来，零知识证明的理论研究才开始不断突破，同时区块链也为零知识证明创造了大展拳脚的机会，因而走进大众视野。

零知识证明这项“黑科技”随着它的热度逐渐增加，相关的学习资源也慢慢丰富起来了。但是由于，一方面零知识证明背后的原理颇为复杂，且内容繁多；另一方面，针对零知识证明的学习资源质量参差不齐，尚未形成系统。因此对绝大多数读者来说，学习零知识证明的难度依然很大。

本文收集了关于零知识证明的一些学习资料（包括科普文章，论文，开源仓库及相关学习网站等），并对这些资源进行了整理分析，希望能对大家有所帮助。

*由于整理时间有限和笔者自身知识的局限性，文章存在不足之处，欢迎纠正、补充和探讨。*


## 1. 故事中的零知识证明

初次接触零知识证明的小伙伴一定会问，究竟什么是零知识证明呢？它到底在做什么？

推荐几篇适合小白的文章：

* **「推荐文章一」[一个数独引发的惨案：零知识证明（Zero-Knowledge Proof）](https://medium.com/qed-it/the-incredible-machine-4d1270d7363a)**

   推荐值：❤️❤️❤️❤️❤️

   难度值：⭐️

   这篇文章的作者是著名的 Ghost 和 Spectre 这两个协议的创始团队的领队 Aviv Zohar。文章非常接地气且通俗易懂，通过三个好朋友一起玩数独游戏的故事介绍了什么是零知识证明。

   原文链接：https://medium.com/qed-it/the-incredible-machine-4d1270d7363a

   中文翻译：https://zhuanlan.zhihu.com/p/34072069

   另外这篇文章中引用了两篇介绍零知识证明的论文，也值得看一看。

* **「推荐文章二」[How to explain zero-knowledge protocols to your children](http://pages.cs.wisc.edu/~mkowalcz/628.pdf)**

   推荐值：❤️❤️❤️

   难度值：⭐️

   这篇来自上个世纪的文章，正如它的标题一样，作者以给孩子讲故事的口吻，讲了一个阿里巴巴与四十大盗的故事，这个故事后来也成为了介绍零知识证明的经典故事。以故事的形式讲述零知识证明使得这篇文章理解起来也很简单。

   原文链接：http://pages.cs.wisc.edu/~mkowalcz/628.pdf

   中文翻译：https://blog.dreamerryao.wiki/archives/%E8%AF%91howtoexplainzero-knowledgeprotocolstoyourchildren

* **「推荐文章三」[Cryptographic and Physical Zero-Knowledge Proof Systems for Solutions of Sudoku Puzzles](http://www.wisdom.weizmann.ac.il/~naor/PAPERS/sudoku.pdf)**

   推荐值：❤️❤️❤️

   难度值：⭐️⭐️⭐️

   如何在不泄漏任何信息的前提下向别人证明你有一个数独问题的答案呢？同样这个问题也是介绍零知识证明的经典案例。论文中提出了使用一个零知识证明协议解决这个问题的方案，这篇论文相比较于前两篇文章，理论性更强一些，篇幅更长，协议的介绍更为详细，但总体来说还算比较好理解。

   原文链接：http://www.wisdom.weizmann.ac.il/~naor/PAPERS/sudoku.pdf

* **「推荐文章四」[Zero knowledge proofs: a tale of two friends](https://medium.com/hackernoon/zero-knowledge-proofs-a-tale-of-two-friends-d7a0ffac3185)**

  推荐值：❤️❤️

  难度值：⭐️⭐️

  与前面几篇文章类似，这篇文章也是通过讲故事的形式来向读者介绍零知识证明的。文中 Prover 要向 Verifier 证明其知道魔法的解法。这篇文章篇幅较短，内容理解起来难度较小。

  原文链接：https://medium.com/hackernoon/zero-knowledge-proofs-a-tale-of-two-friends-d7a0ffac3185

* **「推荐文章五」[Explain Like I’m 5: Zero Knowledge Proof (Halloween Edition)](https://medium.com/hackernoon/eli5-zero-knowledge-proof-78a276db9eff)**

  推荐值：❤️❤️

  难度值：⭐️⭐️

  这同样是一篇讲故事的文章，哈哈~

  这篇文章讲述了一个糖果和百万富翁的故事（Candy bars and millionaires），文章同样篇幅较短，内容理解起来难度较小。
  
  原文链接：https://medium.com/hackernoon/eli5-zero-knowledge-proof-78a276db9eff

兴许是因为如何解释零知识证明的问题并不简单，所以绝大部分入门级的科普文章都是从讲故事开始的。

## 2. 深入理解零知识证明



零知识证明技术涉及的知识点繁多，性质也各不相同。了解了什么是零知识证明以后，就需要对零知识证明更深刻的理解，推荐以下几篇零知识证明系列科普文。

* **「推荐文章六」零知识证明: 抛砖引玉**

  推荐值：❤️❤️❤️❤️

  难度值：⭐️⭐️⭐️

  作者是 Zerocash 协议的创建者之一，密码学大神 Matthew Green[1]。这两篇文章几乎涵盖了学习零知识证明原理所有的基本概念，文章思路很清晰。

  * [零知识证明: 抛砖引玉](https://blog.cryptographyengineering.com/2014/11/27/zero-knowledge-proofs-illustrated-primer/)

    第一篇文章主要从零知识证明的起源开始讲起，然后同样借助了地图三染色和 “时光机”来对零知识证明进行介绍。

    原文链接：https://blog.cryptographyengineering.com/2014/11/27/zero-knowledge-proofs-illustrated-primer/

    中文翻译版本：https://ethfans.org/posts/zero-knowledge-proofs-illustrated-primer

  * [零知识证明：抛砖引玉，Part-2](https://blog.cryptographyengineering.com/2017/01/21/zero-knowledge-proofs-an-illustrated-primer-part-2/)

    这篇文章在第一篇的基础上，进一步对零知识证明的三个性质：可靠性，完整性和零知识，展开介绍。另外还结合 Schnorr 协议介绍了交互式和非交互式的概念。
    
    原文链接：https://blog.cryptographyengineering.com/2017/01/21/zero-knowledge-proofs-an-illustrated-primer-part-2/
    
    中文翻译版本：https://ethfans.org/posts/zero-knowledge-proofs-an-illustrated-primer-part-2

* **「推荐文章七」安比实验室零知识证明介绍系列文章**

  推荐值：❤️❤️❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这个系列的作者是安比实验室创始人郭宇，文章与以往的零知识证明科普文章的不同之处就是它没有单独去讲解零知识的基本原理。而且结合更多的概念和原理，更透彻得将零知识证明技术涉及得诸多原理逐一进行讲解，文章专业性较强，还包含了作者大量的思考，但理解起来也较为直观易懂，非常适合想要深入理解零知识证明的小伙伴。

  另外这个系列的文章还在持续更新中。

  * [探索零知识证明系列一：初识「零知识」与「证明」](https://sec-bit.github.io/blog/2019/07/31/zero-knowledge-and-proof/)

    作为系列的第一篇，这篇文章首先介绍了「证明」的发展历程和「零知识」的作用，并举了一个地图三染色的例子，然后又对「信息」、「知识」和可满足电路的概念展开了介绍。

    原文链接：https://sec-bit.github.io/blog/2019/07/31/zero-knowledge-and-proof/

  * [探索零知识证明系列二：从「模拟」理解零知识证明：平行宇宙与时光倒流](https://sec-bit.github.io/blog/2019/08/06/understand-zero-knowledge-proof-by-simulator/)

    这篇文章介绍了零知识证明中的一个非常重要的概念——模拟（Simulator），「模拟」可以说是安全协议中核心的核心。文章中借助 “平行世界” 的假设去理解零知识读起来也非常有意思。

    原文链接：https://sec-bit.github.io/blog/2019/08/06/understand-zero-knowledge-proof-by-simulator/

  * [探索零知识证明系列三：读心术：从零知识证明中提取「知识」](https://sec-bit.github.io/blog/2019/08/28/extractor-and-proof-of-knowledge/)

    零知识证明有三个重要的性质：可靠性，完整性和零知识。这篇文章探讨了可靠性。文章解释了如何借助「抽取器」和时间倒流的超能力把 Alice 的「知识」完整地「抽取」出来，并可给出了一个与之相关攻击实例 —— ECDSA 签名攻击。

    原文链接：https://sec-bit.github.io/blog/2019/08/28/extractor-and-proof-of-knowledge/

  * [探索零知识证明系列四：亚瑟王的「随机」挑战：从交互到非交互式零知识证明](https://sec-bit.github.io/blog/2019/11/01/from-interactive-zkp-to-non-interactive-zkp/)

    这篇文章主要在讲零知识证明的信任根基——随机挑战。文章对零知识证明协议在两种不同的形式（交互式和非交互式）下随机挑战的方式进行了介绍。另外文章还对交互和非交互形式展开了介绍。

    原文链接：https://sec-bit.github.io/blog/2019/11/01/from-interactive-zkp-to-non-interactive-zkp/

* **「推荐文章八」[零知识证明：一个略微严肃的科普](https://zhuanlan.zhihu.com/p/29491567)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️⭐️

  邓老师这篇“略微严肃”的科普，主要涉及两部分：1. 交互式证明的巨大威力；2. 零知识证明的定义和那些广泛流传的错误的例子

  原文链接：https://zhuanlan.zhihu.com/p/29491567

* **「推荐文章九」[Zero-Knowledge Proofs: A Layman’s Introduction](https://blog.aventus.io/zero-knowledge-proofs-a-laymans-introduction-7020b93beeda)**

  推荐值：❤️❤️

  难度值：⭐️⭐️

  这篇文章首先介绍了零知识证明协议中的三个参与者（Creator，Prover，Verifier）以及 Proofs 和 Verification的概念，并对 zkSNARK （一类零知识证明协议）和椭圆曲线的相关资料进行了介绍。

  原文链接：https://blog.aventus.io/zero-knowledge-proofs-a-laymans-introduction-7020b93beeda

* **「推荐文章十」[白话零知识证明（一）](https://zhuanlan.zhihu.com/p/33189921)**

  推荐值：❤️❤️

  难度值：⭐️⭐️

  这篇来自秘猿科技的文章通过阿里巴巴的故事引出了零知识证明的一些概念，并对其进行了介绍。

  原文链接：https://zhuanlan.zhihu.com/p/33189921

零知识证明涉及很多很有意思的思想和原理，都很值得探讨。在此不得不感叹于数学与密码学的精妙之处，也不得不钦佩密码学家们的厉害。

## 3. 零知识证明的发展

零知识证明的研究今年来一直有新的进展，密码学家们提出了各种不同的协议，推荐两篇文章介绍零知识证明研究的发展过程。

* **「推荐文章十一」[区块链学习笔记 (1)：零知识证明的江湖](https://zhuanlan.zhihu.com/p/31651393)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️

  这篇文章讲了自 1895 年提出以来，零知识证明理论研究的发展过程，以及 zk-SNARKs 与零知识证明技术结合起来的发展过程。推荐给想了解零知识理论研究的发展过程的小伙伴。

  原文链接：https://zhuanlan.zhihu.com/p/31651393

* **「推荐文章十二」[Efficient Cryptographic Arguments and Proofs – Or How I Became a Fractional Monetary Unit](https://www.benthamsgaze.org/2019/05/22/efficient-cryptographic-arguments-and-proofs-or-how-i-became-a-fractional-monetary-unit/)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️

  这篇文章来自UCL信息安全研究人员的博客 [Bentham’s Gaze](https://www.benthamsgaze.org/about/)[2],文章介绍了自零知识证明提出以来，这群研究人员在理论研究上的研究历程及成果，包括知名的 bulletProof 和 zk-STARK 等。读完这篇文章相信会对大家深入理解零知识证明的诸多协议有所帮助。

  原文链接：https://www.benthamsgaze.org/2019/05/22/efficient-cryptographic-arguments-and-proofs-or-how-i-became-a-fractional-monetary-unit/

零知识证明迄今为止发展了三十多年，早期一直停留在理论层面，直到近十年才逐渐取得突破。随着越来越多研究人员的进场，相信这个领域未来还会有更多令人惊喜的成果。

## 4. zk-SNARKs 原理

作为零知识证明领域最知名的一类协议，zk-SNARKs 的理论研究和应用也最为广泛。推荐一些介绍 zk-SNARKs 的资料。

* **「推荐文章十三」V 神的 zk-SNARKs 科普文章**

  推荐值：❤️❤️❤️❤️

  难度值：⭐️⭐️⭐️⭐️

  V 神的这几篇文章应该算得上是流传最为广泛的 zk-SNARK 科普文了。不用多说，推荐阅读。

  * [Quadratic Arithmetic Programs: from Zero to Hero](https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649)

    这篇文章详细介绍了 zk-SNARKs 的实现过程。文中将 zk-SNARKs 的实现分为以下几个步骤：

    1. computational problem —> 电路
    2. 电路 —> R1CS
    3. R1CS —> QAP
    4. QAP —> Linear PCP
    5. Linear PCP —> Linear Interactive Proof
    6. Linear Interactive Proof —> zkSNARK

    原文链接：https://medium.com/@VitalikButerin/quadratic-arithmetic-programs-from-zero-to-hero-f6d558cea649

  * [Exploring Elliptic Curve Pairings](https://medium.com/@VitalikButerin/exploring-elliptic-curve-pairings-c73c1864e627)

    这篇文章介绍了椭圆曲线配对。

    原文链接：https://medium.com/@VitalikButerin/exploring-elliptic-curve-pairings-c73c1864e627

  * [Zk-SNARKs: Under the Hood](https://medium.com/@VitalikButerin/zk-snarks-under-the-hood-b33151a013f6)

    这篇文章主要介绍了匹诺曹协议。

    原文链接：https://medium.com/@VitalikButerin/zk-snarks-under-the-hood-b33151a013f6

* **「推荐文章十四」[zcash 官方科普文](https://z.cash/technology/zksnarks/)**

  推荐值：❤️❤️❤️❤️

  难度值：⭐️⭐️⭐️⭐️

  这个系列的文章来自 zCash 官方博客。首先介绍了零知识的基本概念以及其应用到 zcash 中的大致思路。随后 7 篇文章分别对 7 个关键点进行了详细介绍（同态隐藏，多项式盲验证，KCA，完整的多项式盲验证，计算到多项式的转换，匹诺曹协议以及椭圆曲线配对），推荐给想深入了解 zk-SNARKs 实现原理的小伙伴。

  原文链接：

  1. What are zk-SNARKs?：https://z.cash/technology/zksnarks/
  2. Explaining SNARKs Part I: Homomorphic Hidings https://electriccoin.co/blog/snark-explain/
  3. Explaining SNARKs Part II: Blind Evaluation of Polynomials https://electriccoin.co/blog/snark-explain2/
  4. Explaining SNARKs Part III: The Knowledge of Coefficient Test and Assumption https://electriccoin.co/blog/snark-explain3/
  5. Explaining SNARKs Part IV: How to make Blind Evaluation of Polynomials Verifiable https://electriccoin.co/blog/snark-explain4/
  6. Explaining SNARKs Part V: From Computations to Polynomials https://electriccoin.co/blog/snark-explain5/
  7. Explaining SNARKs Part VI: The Pinocchio Protocol https://electriccoin.co/blog/snark-explain6/
  8. Explaining SNARKs Part VII: Pairings of Elliptic Curves](https://electriccoin.co/blog/snark-explain7/
  9. 中文翻译版本链接：https://www.jianshu.com/p/b6a14c472cc1、https://www.jianshu.com/p/92f54fc08d58

* **「推荐文章十五」[Why and How zk-SNARK Works](https://arxiv.org/pdf/1906.07221.pdf)**

  推荐值：❤️❤️❤️❤️❤️

  难度值：⭐️⭐️⭐️

  作者将其学习 zk-SNARK 的经验总结成了一份 PDF 文档并分成 8 篇文章发布到了 Medium 上。与大部分的 zk-SNARK 科普文不同，这个系列的文章没有直接开始讲 zk-SNARK，而是从最基本的数学原理讲起，讲解得非常细致，特别适合数学和密码学基础相对薄弱的小伙伴。

  原文链接：

  1. PDF 完整版：https://arxiv.org/pdf/1906.07221.pdf
  2. Why and How zk-SNARK Works 1: Introduction & the Medium of a Proof：https://medium.com/@imolfar/why-and-how-zk-snark-works-1-introduction-the-medium-of-a-proof-d946e931160
  3. Why and How zk-SNARK Works 2: Proving Knowledge of a Polynomial：https://medium.com/@imolfar/why-and-how-zk-snark-works-2-proving-knowledge-of-a-polynomial-f817760e2805
  4. Why and How zk-SNARK Works 3: Non-interactivity & Distributed Setup：https://medium.com/@imolfar/why-and-how-zk-snark-works-3-non-interactivity-distributed-setup-c0310c0e5d1c
  5. Why and How zk-SNARK Works 4: General-Purpose Computation：https://medium.com/@imolfar/why-and-how-zk-snark-works-4-general-purpose-computation-dcdc8081ee42
  6. Why and How zk-SNARK Works 5: Variable Polynomials：https://medium.com/@imolfar/why-and-how-zk-snark-works-5-variable-polynomials-3b4e06859e30
  7. Why and How zk-SNARK Works 6: Verifiable Computation Protocol：https://medium.com/@imolfar/why-and-how-zk-snark-works-6-verifiable-computation-protocol-1aa19f95a5cc
  8. Why and How zk-SNARK Works 7: Constraints and Public Inputs：https://medium.com/@imolfar/why-and-how-zk-snark-works-7-constraints-and-public-inputs-e95f6596dd1c
  9. Why and How zk-SNARK Works 8: Zero-Knowledge Computation：https://medium.com/@imolfar/why-and-how-zk-snark-works-8-zero-knowledge-computation-f120339c2c55

* **「推荐文章十六」 [zkSNARKs in a nutshell](https://blog.ethereum.org/2016/12/05/zksnarks-in-a-nutshell/)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这篇文章对零知识证明做了总结，分成四个部分：

  1. 编码成一个多项式问题
  2. 简单随机抽样
  3. 同态（Homomorphic）编码 / 加密
  4. 零知识

  文章首先介绍了零知识证明，然后又讲解了zk-SNARKs 的实现，最后分析了将零知识证明结合到以太坊上的作用和方式。

  原文链接：https://blog.ethereum.org/2016/12/05/zksnarks-in-a-nutshell/

  中文翻译版本：https://zhuanlan.zhihu.com/p/31780893

* **「推荐文章十七」[Zero-knowledge proofs, a board game, and leaky abstractions: how I learned zk-SNARKs from scratch](https://medium.com/@weijiek/how-i-learned-zk-snarks-from-scratch-177a01c5514e)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️

  作者坚持一个观点：学习新技能的一个很好的方法是用它建立一些东西。这篇文章就是在介绍作者是如何通过实现一个小的应用来学习 zk-SNARKs 的。文章主要介绍了作者的实现过程和他的思考，文中有很多好的经验时候大家学习。

  原文链接：https://medium.com/@weijiek/how-i-learned-zk-snarks-from-scratch-177a01c5514e

* **「推荐文章十八」[零知识证明 - 从QSP到QAP](https://mp.weixin.qq.com/s/eU8mp81VhpV-g1x89-uZYA)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这篇文章主要介绍了 QSP/QAP ，QAP 和 QSP 问题类似。QAP 问题的zkSNARK 的证明验证过程和 QSP 非常相似。对这部分感兴趣的小伙伴推荐读一读。

  原文链接：https://mp.weixin.qq.com/s/eU8mp81VhpV-g1x89-uZYA

"零知识证明技术就像一个江湖，而 zk-SNARKs 是只是比较著名的门派。而在这个江湖中，还有很多其他的门派，他们风格各异，使用的武器也不尽相同。"[3] zk-SNARKs 协议涉及的技术构件很多，也较为复杂，深入学习这部分确实需要下很多功夫。

## 5. 零知识证明协议

零知识证明协议很多，每个协议的实现也各不相同，有些协议已经应用到了实际的领域，有些还在探索中。推荐几篇介绍不错的文章。

* **「推荐文章十九」STARKs 科普**

  推荐值：❤️❤️❤️❤️

  难度值：⭐️⭐️⭐️⭐️

  V 神的这个科普系列文章，非常详细得介绍了 STARKs 的实现，分成三个部分进行讲解。

  原文链接：

  1. STARKs, Part I: Proofs with Polynomials：https://vitalik.eth.limo/general/2017/11/09/starks_part_1.html

     中文翻译版本：https://ethfans.org/posts/starks_part_1

  2. STARKs, Part II: Thank Goodness It's FRI-day：https://vitalik.eth.limo/general/2017/11/22/starks_part_2.html

     中文翻译版本：https://ethfans.org/posts/starks_part_2

  3. STARKs, Part 3: Into the Weeds：https://vitalik.eth.limo/general/2018/07/21/starks_part_3.html)

     中文翻译版本：https://ethfans.org/posts/starks_part_3_1

     中文翻译版本：https://ethfans.org/posts/starks_part_3_2

* **「推荐文章二十」 [Understanding PLONK](https://vitalik.eth.limo/general/2019/09/22/plonk.html)**

  推荐值：❤️❤️❤️❤️

  难度值：⭐️⭐️⭐️⭐️

  这篇文章同样来自 V 神的博客，介绍了 PLONK 的工作原理。PLONK 是一种全新的零知识证明系统，支持通用或可更新的可信设置（trusted setup），作者是 Filecoin 母公司 Protocol Labs 的研究员 Ariel Gabizon 和以太坊隐私交易协议 Aztec Protocol 的两名研究人员 Zachary J. Williamson、Oana Ciobotaru。

  原文链接：https://vitalik.eth.limo/general/2019/09/22/plonk.html

  中文翻译版本：https://www.8btc.com/article/486086

* **「推荐文章二十一」[Groth09 笔记](https://github.com/huyuguang/zkpblog/blob/master/groth09.md)**

  推荐值：❤️❤️

  难度值：⭐️⭐️⭐️

  这篇文章作者huyuguang，文中对 Groth09 论文[4]的内容进行了总结，对大家学习 Groth09 有所帮助。

  原文链接：https://github.com/huyuguang/zkpblog/blob/master/groth09.md

* **「推荐文章二十二」 零知识证明 - Groth16 算法介绍**

  推荐值：❤️❤️

  难度值：⭐️⭐️⭐️

  Star Li 的这两篇文章主要从工程应用理解的角度介绍了 Groth16 算法的证明和验证过程，推荐给学习 Groth16 算法的小伙伴。

  原文链接

  1. 零知识证明 - Groth16 算法介绍：https://mp.weixin.qq.com/s/SguBb5vyAm2Vzht7WKgzug
  2. 零知识证明 - 有关 Groth16 的zk证明的理解：https://mp.weixin.qq.com/s/x1ggw3VplXAIeL87D5bUfw

对于零知识证明各个协议介绍的文章还比较有限，随着应用的增多，相信这方面的文章也会越来越多。

## 6.  零知识证明在区块链领域的应用

零知识证明技术是随着区块链的发展逐渐走入大众视野的，目前零知识证明结合区块链的研究和应用也越来越多。

* **「推荐文章二十三」 [一文读懂区块链中的零知识证明](https://www.odaily.com/post/5133827)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这篇来自BFTF技术社区联盟的文章介绍了零知识证明在 zcash 和门罗币上的应用。

  原文链接：https://www.odaily.com/post/5133827

* **「推荐文章二十四」[How to prove that you know something, without revealing it? Zero-knowledge proofs, ZCash, Ethereum.](https://medium.com/hackernoon/how-to-prove-that-you-know-something-without-revealing-it-zero-knowledge-proofs-zcash-ethereum-43ce35d4d1c5)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️

  这篇文章介绍了零知识证明在 Zcash 和以太坊上的应用。

  原文链接：https://medium.com/hackernoon/how-to-prove-that-you-know-something-without-revealing-it-zero-knowledge-proofs-zcash-ethereum-43ce35d4d1c5

* **「推荐文章二十五」[Zero-knowledge proofs, Zcash, and Ethereum](https://blog.keep.network/zero-knowledge-proofs-zcash-and-ethereum-f6d89fa7cba8)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️

  这篇文章介绍了零知识证明在 Zcash 和以太坊上的应用。

  原文链接：https://blog.keep.network/zero-knowledge-proofs-zcash-and-ethereum-f6d89fa7cba8

* **「推荐文章二十六」[零知识证明 - zk-SNARK应用场景分析](https://mp.weixin.qq.com/s/9QccZtFcvGwne-NN4BBA5w)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️

  这篇文章介绍了零知识证明在 Zcash，Filecoin项目和 Loopring DEX 3.0 协议中的应用。

  原文链接：https://mp.weixin.qq.com/s/9QccZtFcvGwne-NN4BBA5w

* **「推荐文章二十七」[Zerocoin: making Bitcoin anonymous](https://blog.cryptographyengineering.com/2013/04/11/zerocoin-making-bitcoin-anonymous/)**

  推荐值：❤️❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这篇文章介绍了 Zerocoin 协议是如何利用 zk-SNARKs 在区块链上实现匿名的。

  原文链接：https://blog.cryptographyengineering.com/2013/04/11/zerocoin-making-bitcoin-anonymous/

* **「推荐文章二十八」[不是程序员也能看懂的ZCash零知识证明](https://zhuanlan.zhihu.com/p/24440530)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这篇文章使用比较通俗易懂的语言介绍了 zCash 如何利用零知识证明实现匿名交易的。

  原文链接：https://zhuanlan.zhihu.com/p/24440530

* **「推荐文章二十九」 [Monero to Become First Billion-Dollar Crypto to Implement ‘Bulletproofs’ Tech](https://www.coindesk.com/monero-to-become-first-billion-dollar-crypto-to-implement-bulletproofs-tech)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这篇文章介绍了 Monero 如何使用 Bulletproofs 技术实现隐私特性的。

  原文链接：https://www.coindesk.com/monero-to-become-first-billion-dollar-crypto-to-implement-bulletproofs-tech

* **「推荐文章三十」 [zkPoD:区块链，零知识证明与形式化验证，实现无中介、零信任的公平交易](https://mp.weixin.qq.com/s/TCYDfOAle0K3D69eBm6HNw)**

  推荐值：❤️❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这是安比实验室今年发布的基于零知识证明的公平交易协议。

  原文链接：https://mp.weixin.qq.com/s/TCYDfOAle0K3D69eBm6HNw

* **「推荐文章三十一」 [零知识证明 - Loopring DEX 3.0](https://mp.weixin.qq.com/s/oTbzyqtc-TzJXbMafj28DQ)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这篇文章介绍了 Loopring DEX 3.0 协议的零知识证明部分实现原理。

  原文链接：https://mp.weixin.qq.com/s/oTbzyqtc-TzJXbMafj28DQ

零知识证明的应用正在逐步增加，从最早的公链 zCash，Monero，到最近基于以太坊平台的 zkPoD, Loopring DEX 3.0应用等，零知识证明在区块链领域的应用将越来越多。

## 7. 零知识证明相关的技术和漏洞分析文章

零知识证明技术涉及的知识内容很多，在实际的应用场景中，零知识证明的实现还存在诸多的挑战，协议安全，性能等等问题都有可能限制其发展。这一节推荐一些技术分析和漏洞分析的文章。

* **「推荐文章三十二」 A Marlin is One of the Fastest SNARKs in the Ocean**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️⭐️

  这篇文章来自于博客 [Bentham’s Gaze](https://www.benthamsgaze.org/)，文章观点认为 Marlin 是最快的 SNARKs 方案，并将其与其它的方案进行了比较。

  原文链接：https://www.benthamsgaze.org/2019/09/19/a-marlin-is-one-of-the-fastest-snarks-in-the-ocean/

* **「推荐文章三十三」[How to do Zero-Knowledge from Discrete-Logs in under 7kB](https://www.benthamsgaze.org/2016/10/25/how-to-do-zero-knowledge-from-discrete-logs-in-under-7kb/)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这篇文章同样来自于博客 [Bentham’s Gaze](https://www.benthamsgaze.org/)，文章介绍了Groth09 论文中的优化方案。

  原文链接：https://www.benthamsgaze.org/2016/10/25/how-to-do-zero-knowledge-from-discrete-logs-in-under-7kb/





* **「推荐文章三十四」[zkSNARK 合约「输入假名」漏洞致众多混币项目爆雷](https://sec-bit.github.io/blog/page/2/)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这篇文章的作者是安比实验室 p0n1，文章介绍了大量零知识证明项目由于错误地使用了某个 zkSNARKs 合约库，引入「输入假名 (Input Aliasing) 」漏洞，可导致伪造证明、双花、重放等攻击行为发生，且攻击成本极低。

  原文链接：https://sec-bit.github.io/blog/page/2/

  

* **「推荐文章三十五」[硬核！360高级安全专家彭峙酿以Zcash为例，谈零知识性证明的安全和隐私问题](https://zhuanlan.zhihu.com/p/87690026)**

  推荐值：❤️❤️❤️❤️

  难度值：⭐️⭐️⭐️

  这篇文章是对360高级安全专家彭峙酿博士在 CCF 会议上分享报告的整理。报告中介绍了比特币的隐私问题，零知识证明技术，zk-SNARKs，以及多个实现漏洞。报告干货满满。

  原文链接：https://zhuanlan.zhihu.com/p/87690026

* **「推荐文章三十六」[零知识证明中所涉及的有限域](https://github.com/huyuguang/zkpblog/blob/master/有限域.md)**

  推荐值：❤️❤️❤️

  难度值：⭐️⭐️⭐️⭐️

  有限域的计算是实现零知识证明协议的一个非常重要的环境，这篇文章对零知识证明中所涉及导的有限域的知识进行了介绍，非常有用。

  原文链接：https://github.com/huyuguang/zkpblog/blob/master/有限域.md



## 8. 零知识证明开源仓库及介绍

下面介绍几个热度比较高的零知识证明实现仓库及其源码分析文章，很多的零知识项目都是基于这几个仓库的代码做的。

* [libsnark](https://github.com/scipr-lab/libsnark)

  libsnark 是实现一个 C++ 版本的零知识证明库。

  仓库链接：https://github.com/scipr-lab/libsnark

  * **「推荐文章三十七」[零知识证明 - libsnark源代码分析](https://mp.weixin.qq.com/s/UHqpfl6ImVwa4HtsiksqJA)**

    推荐值：❤️❤️❤️

    难度值：⭐️⭐️⭐️

    原文链接：https://mp.weixin.qq.com/s/UHqpfl6ImVwa4HtsiksqJA

  * **「推荐文章三十八」[零知识证明实战：libsnark](https://zhuanlan.zhihu.com/p/46477111)**

    推荐值：❤️❤️❤️

    难度值：⭐️⭐️⭐️

    原文链接：https://zhuanlan.zhihu.com/p/46477111

* [bellman](https://github.com/zkcrypto/bellman)

  bellman是Zcash团队用Rust语言开发的一个zk-SNARK软件库，实现了Groth16 算法。

  仓库链接：https://github.com/zkcrypto/bellman

  * **「推荐文章三十九」[零知识证明 - bellman源码分析](https://mp.weixin.qq.com/s/NvX11tNSEpV1DR-3PwpIAQ)**

    推荐值：❤️❤️

    难度值：⭐️⭐️⭐️

    原文链接：https://github.com/zcash/librustzcash/tree/master/bellman

* [snarkjs](https://github.com/iden3/snarkjs)

  libsnark 是实现一个 javascript 版本的零知识证明库，实现了 Groth16。

  仓库链接：https://github.com/iden3/snarkjs

  

## 9. 零知识证明相关论文

下面介绍一下零知识证明相关的学术论文，深入学习零知识证明研究成果的小伙伴可以去阅读以下的这些论文。

推荐值：❤️❤️❤️

难度值：⭐️⭐️⭐️⭐️⭐️

* 1985 年，零知识证明技术首次被提出 

   原文链接：[The Knowledge Complexity of Interactive Proof Systems](https://epubs.siam.org/doi/10.1137/0218012)

* BulletProof

   1. [Gro09 ](https://link.springer.com/chapter/10.1007/978-3-642-03356-8_12)提出了一种证明“向量内积”的方法：

      原文链接：https://link.springer.com/chapter/10.1007/978-3-642-03356-8_12

   2. [BCC+16](https://eprint.iacr.org/2016/263) 找到了一种将算数电路编码为向量的方法，从而把电路可满足性的证明转化为向量内积的证明：

      原文链接：https://eprint.iacr.org/2016/263

   3. [BulletProof](https://eprint.iacr.org/2017/1066)继续改进了这种方案：

      原文链接：https://eprint.iacr.org/2017/1066

* zkSNARKs with trusted setup

   1. [Groth10](https://link.springer.com/chapter/10.1007/978-3-642-17373-8_19) 引入了preprocessing的步骤，通过可信第三方生成Common Reference String来实现无交互证明:

      原文链接：https://link.springer.com/chapter/10.1007/978-3-642-17373-8_19

   2. [GGPR13](https://eprint.iacr.org/2012/215) 引入了另一种算数电路编码方式，即Quadratic Arithmetic Program(QAP)，大大提升了证明的效率:

      原文链接：https://eprint.iacr.org/2012/215

   3. [Pinocchio](https://eprint.iacr.org/2013/279) 和 [Groth16](https://eprint.iacr.org/2016/260) 等是在此基础上的改进:

      原文链接：https://eprint.iacr.org/2013/279

      原文链接：https://eprint.iacr.org/2016/260

* [Ligero: Lightweight Sublinear Arguments Without a Trusted Setup](https://acmccs.github.io/papers/p2087-amesA.pdf):

   原文链接：https://acmccs.github.io/papers/p2087-amesA.pdf

* PLONK:

   原文链接：https://eprint.iacr.org/2019/953

* Marlin

   原文链接：https://eprint.iacr.org/2019/1047.pdf

* Sonic

   原文链接：https://eprint.iacr.org/2019/099

* Libra

   原文链接：https://eprint.iacr.org/2019/317

* Hyrax

   原文链接：https://eprint.iacr.org/2017/1132.pdf

* zk-STARKs

    原文链接：https://eprint.iacr.org/2018/046

## 10. 零知识证明学习资料推荐网站

* [awesome-zero-knowledge-proofs](https://github.com/matter-labs/awesome-zero-knowledge-proofs)

  推荐值：❤️❤️❤️❤️

  这是一个 Github 仓库，收录了一系列零知识证明的学习资料

  链接：https://github.com/matter-labs/awesome-zero-knowledge-proofs

* [Zero-Knowledge Proofs](https://zkp.science/)

  推荐值：❤️❤️❤️❤️

  这个网站也收录了一系列零知识证明的学习资料，相对来说学术性更强一些。

  链接：[https://zkp.science](https://zkp.science/)

* [zkproof](https://zkproof.org/workshop2/abstracts.html)

  推荐值：❤️❤️❤️❤️

  ZKProof.org 是为规范零知识证明的使用而形成的一个组织，它的网站上有大量关于零知识证明的资料。

  链接：https://zkproof.org/

* [benthamsgaze.org](https://www.benthamsgaze.org)

  推荐值：❤️❤️❤️

  这是一个来自UCL信息安全研究人员组成的团队的博客，它的博客上会经常发布一些零知识证明的文章。

  链接：https://www.benthamsgaze.org


## 参考文献

[1] https://isi.jhu.edu/~mgreen/

[2] https://www.benthamsgaze.org/about

[3] https://zhuanlan.zhihu.com/p/31651393

[4] https://link.springer.com/chapter/10.1007/978-3-642-03356-8_12

[5] https://github.com/matter-labs/awesome-zero-knowledge-proofs

[6] https://zkp.science/

[7] https://zhuanlan.zhihu.com/p/89386868?utm_source=wechat_session&utm_medium=social&utm_oi=26765481213952

