
# Lotus项目源代码导读--项目简介

## 前言

Filecoin可谓是当前区块链的一个明星项目，首先在拼爹的年代它有一个高贵的出身：协议实验室，这个名字或者有人比较陌生，但说起IPFS可就是鼎鼎大名了，IPFS就是协议实验室的大儿子。而Filecoin就是小儿子了，再有它可谓是含着金钥匙出生的：2017年该项目ICO的金额是2.5亿美金，在当时是最高的。

IPFS项目源自2014年，他本身已经很成功项目，代码模块化程度非常高，例如libp2p已经在开源社区很多应用。而Filecoin的项目不仅重用了IPFS的绝大多数代码，而且开发者大多是原先开发IPFS的那批程序员，所以不仅技术起点高，而且团队技术力量雄厚。所以这样的项目不仅得到币/链圈的推崇，也得到技术人员的喜爱。

最早Filecoin的项目是go-filecoin，但2019年下半年团队推出一个新的项目：lotus，这个项目后来居上，率先在12月11日启动测试网。所以可以理解这个项目是之前go-filecoin的重构项目，但官方目前并没有抛弃go-filecoin项目。本人理解其中原因是为保证区块链安全，因为多个代码实现可以避免因为仅有的代码实现的BUG而导致全网瘫痪。

## 项目相关链接

* github代码主仓库：[https://github.com/filecoin-project/lotus](https://github.com/filecoin-project/lotus)
* 使用指南文档：[https://docs.lotu.sh/](https://docs.lotu.sh/)
* 设计规范文档：[https://filecoin-project.github.io/specs](https://filecoin-project.github.io/specs)
* 测试网：[https://stats.testnet.filecoin.io/](https://stats.testnet.filecoin.io/)
* Slack官方频道：[https://app.slack.com/client/TEHTVS1L6/CPFTWMY7N](https://app.slack.com/client/TEHTVS1L6/CPFTWMY7N)


## 项目代码分支

master为0.2.x版本的测试网版本，但最新代码目前是在testnet/3分支上，对应的区块链网络名称为interopnet，相当于是预测试网。本文开始写时是2020年4月23日，使用的版本号为ac92ad36，后续随着版本的版本，文中使用的代码也会跟随更新。

## 项目功能简介

Lotus是一个分布式存储区块链项目，其目标是提供一个存储的平台和市场，一方是提供存储和检索服务的矿工，另一方普通用户通过支付数字货币Fil可上传存储数据，并在需要时又可获取到数据。矿工为了证明自身进行存储的事实，采用零知识证明加密技术对存储的数据进行计算并将结果广播到区块链，一方获取用户支付的Fil，另一方面获得Power（单位与存储的字节一样），当Power到达一定程度还可以有机会参与到区块链的打包，从而再获取得区块打包的奖励。

## 代码基本情况
上层代码开发语言为Golang，版本是1.13或以上。底层涉及到零知识证明等算法相关的逻辑采用Rust语言，toolchain版本为nightly-2020-03-19。底层库先是编译成静态库，上层Golang通过cgo调用底层静态库。底层的代码仓库叫filecoin-ffi，URL为[https://github.com/filecoin-project/filecoin-ffi.git](https://github.com/filecoin-project/filecoin-ffi.git),在主仓库的路径extern/filecoin-ffi之下。

## 二进制文件

代码编译后可能产生多个二进制文件，最主要有两个：lotus和lotus-storage-miner。lotus为全节点，lotus-storage-miner为矿工软件。这两个软件均是命令行方式运行。

## 后续

以上是lotus的基本介绍，后续再进lotus全节点daemon服务启动过程的源代码逻辑进行分析。










