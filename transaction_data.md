### 交易数据 | Transaction Data

每一个区块必须包含一笔或者多笔交易。这些交易的第一笔必须是一个 coinbase 交易，也被称作 generation 交易，它包含了这个区块所有花费和奖励（由一个区块的补贴和这个区块任何交易产生的交易费用构成）。

一个 coinbase 交易的 UTXO 有一个特殊的条件，那就是在之后的 100 个区块内都不能被花费（被用作输入）。这临时性的杜绝了一个矿工花费掉因为分叉而可能被判定为陈旧区块（这个 coinbase 交易因此被销毁掉）中所包含的补贴和交易费。

区块并不要求一定包含非 coinbase 的交易，但是矿工们为了把他们的交易手续费包含其中总是会带上额外的交易。

所有的交易，包括 coinbase 交易，都会被编码为二进制的 rawtransaction 格式存入区块。

通过对 rawtransaction 格式做哈希得到了交易标识符（txid）。从这些 txids 中，通过将一个 txid 与另一个 txid 配对然后做哈希运算，最终构建了 merkle 树。如果 txids 的个数为奇数，那么没有被配对的那个 txid 将会与他自身的一个副本配对做哈希。

以上得到的哈希值再一一配对，让一个哈希值与另一个再做哈希运算。任何没有可配对的哈希值与自身配对做哈希。这个过程迭代进行直到只剩下一个哈希值，这就是 merkle 根节点。

例如，如果交易只是连接（没有做哈希运算）在一起，那么具有 5 个交易的 merkle 树应该如下图所示：

```
       ABCDEEEE .......Merkle 根节点
      /        \
   ABCD        EEEE
  /    \      /
 AB    CD    EE .......E 与它自身配对
/  \  /  \  /
A  B  C  D  E .........交易
```

在[简化支付验证（SPV）](https://bitcoin.org/en/glossary/simplified-payment-verification)提案中指出，merkle 树允许客户端通过一个完整节点从一个区块头中获取其 merkle 根节点和中间哈希值列表来验证一个交易被包含在这个区块中。这个完整节点并不需要是可信的，因为伪造区块头十分困难而中间哈希值是不可能被伪造的，否则验证就会失败。

例如，要验证交易 D 被加到了区块中，一个 SPV 客户端只需要一份 C，AB 和 EEEE 的副本进而做哈希运算得到 merkle 根节点，客户端并不需要知道任何其他交易的信息。如果这 5 个交易的大小都是一个区块的最大上限，那么下载整个区块需要下载 500,000 个字节，但下载树的哈希值和区块头仅仅需要 140 个字节。

注意：如果在同一个区块中找到了相同的 txids，那么 merkle 树可能会出现与另一个区块的碰撞，归因于 merkle 树对非平衡的实现（对孤立的哈希值做复制）会将一个区块中一些或全部的重复内容移除掉。从对于单独的交易具有相同的 txid 是不现实的角度来看，merkle 树的实现不会对诚实的软件增加负担，但如果一个区块的无效状态要被缓存那么就必须做检查，否则，一个移除了重复交易的有效区块会因为具有相同的 merkle 根节点和区块哈希而被缓存的无效状态拒绝，导致了编号为 [CVE-2012-2459](https://en.bitcoin.it/wiki/CVEs#CVE-2012-2459) 的安全问题。