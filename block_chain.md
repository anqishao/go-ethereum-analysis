### 一. BlockChain的初始化
服务初始化的时候会调用core.SetupGenesisBlock来加载创始区块。顾名思义.创始区块就是区块链中的第一个区块.number值为0。紧接着调用core.NewBlockChain来加载区块链。

初始化方法做了这么几件事：
1. 创建各种lru缓存(最近最少使用的算法)
2. 初始化triegc（用于垃圾回收的区块number 对应的优先级队列）.初始化stateDb.NewBlockValidator()初始化区块和状态验证器.NewStateProcessor()初始化区块状态处理器
3. NewHeaderChain()初始化区块头部链
4. bc.genesisBlock = bc.GetBlockByNumber(0)  拿到第0个区块.也就是创世区块
5. bc.loadLastState() 加载最新的状态数据
6. 查找本地区块链上时候有硬分叉的区块.如果有调用bc.SetHead回到硬分叉之前的区块头
7. go bc.update() 定时处理future block

### 二. 看看bc.loadLastState()方法

1. 获取到最新区块以及它的hash
2. 从stateDb中打开最新区块的状态trie.如果打开失败调用bc.repair(&currentBlock)方法进行修复。修复方法就是从当前区块一个个的往前面找.直到找到好的区块.然后赋值给currentBlock。
3. 获取到最新的区块头
4. 找到最新的fast模式下的block.并设置bc.currentFastBlock

### 三. 再看看用来回滚区块的bc.SetHead()方法

1. 首先调用bc.hc.SetHead(head. delFn).回滚head对应的区块头。并清除中间区块头所有的数据和缓存。设置head为新的currentHeadr。
2. 重新设置bc.currentBlock.bc.currentFastBlock
3. 调用bc.loadLastState().重新加载状态

### 四. 之前分析Downloader和Fetcher的时候.在同步完区块后会调用InsertChain方法插入到本地BlockChain中。我们看看InsertChain怎么工作的

BlockChain模块插入一个新区块的流程。一个新区块的来源有两种可能性，第一种可能性是本节点挖矿成功，要调用BlockChain模块向本地区块链上插入，第二种可能性是节点从网络上的其他节点收到一个区块，调用BlockChain模块插入本地区块链。将一个区块插入区块链是调用BlockChain的insertChain函数.inertChain函数是功能是将一组区块批量插入区块链，inesrtChain函数会检查这一组区块是否是首尾相接。检查无误后会校验区块头和区块体，然后还需要校验状态是不是正确。最后将区块插入区块链，需要注意的是能插入区块链不一定能插入规范链，在插入的时候会具体判断是否能插入规范链，如果不能插入规范链就是一条分叉。

调用bc.insertChain(chain).并将插入的结果广播给订阅了blockChain事件的对象。
1. 首先确保插入的blocks是安次序排列的
2. 调用共识引擎bc.engine.VerifyHeaders.验证区块链的这些headers。
3. 如果共识验证没有问题.再调用bc.Validator().ValidateBody(block)验证block的Body.这个方法只验证block的叔区块hash和区块交易列表的hash。
4. 根据ValidateBody验证结果.如果是还没有插入本地的区块.但是其父区块在bc.futureBlocks就加入bc.futureBlocks。如果父区块是本地区块.但是没有状态.就递归调用bc.insertChain(winner).直到有状态才插入。
5. 获得父区块的状态.调用processor.Process（）处理block的交易数据.并生成收据.日志等信息.产生本区块的状态。Process()方法.执行了Block里面包含的的所有交易.根据交易的过程和结果生成所有交易的收据和日志信息。（fast模式下收据数据是同步过来的.full模式下是本地重现了交易并生成了收据数据）
6. 调用bc.Validator().ValidateState.对产生的区块交易收据数据于收到的block相关数据进行对比验证。对比消费是否一样.对比bloom是否一致.根据收据生成hash是否一致.对比header.root和stateDb的merkle树的根hash是否一致。
7. 调用bc.WriteBlockWithState(block, receipts, state).将block写入到本地的区块链中.并返回status。根据status判断插入的是主链还是side链.如果是主链bc.gcproc需要加上验证花费的时间。
8. 返回结果事件和日志信息.用于通知给订阅了区块插入事件的对象

### 五. 再分析一下bc.WriteBlockWithState是怎么工作的：

WriteBlockWithState方法的功能是将一个区块写入区块链，同时处理可能发生的分叉。能够执行到WriteBlockWithState这个函数说明区块本身是被验证过的没有问题，所以这个方法一定能将区块写入数据库，但是能不能写入规范链，需要进一步判断，假设写入的是规范链，是在原有规范链基础上追加一个呢? 还是将数据库中的一个分叉升级成规范链呢? 这个也要进一步判断。所以WriteBlockWithState方法将区块写入区块链的同时还会处理可能的分叉。WriteBlockWithState函数将一个区块写入区块链的过程其实就是将一个区块写入数据库，一个区块在数据库中是几个部分单独存储的，分别是区块头、区块体、interlinks、收据、区块号、状态，所以WriteBlockWithState函数就是将区块的这几个部分按特定的键值对写入数据库。

1. 从数据库中获取到parent的interlinks。计算新的interlinks值.并写入数据库。
2. 调用WriteBlock(batch, block) 把block的body和header都写到数据库
3. 调用state.Commit(bc.chainConfig.IsEIP158(block.Number()))把状态写入数据库并获取到状态root。
4. 按规则处理bc.stateCache缓存.并清理垃圾回收器
5. 调用WriteBlockReceipts.把收据数据写入数据库
6. 如果发现block的父区块不是本地当前最新区块.调用bc.reorg(currentBlock, block).如果新区块比老区块高度高.则把高出来的区块一一insert进blockChain。
7. 调用WriteTxLookupEntries.根据交易hash建立数据库的索引。调用WritePreimages.把Preimages写入数据库.Preimages在evm执行sha3指令时产生。
8. 如果新的高度大于或等于本地高度.说明是主链区块.调用bc.insert(block).更新blockChain的currentBlock.currentHeader.currentFastBolck等信息。如果不是主链区块则不会。

### 六. insertTail方法

将区块写入规范链，从下面的代码可以看出将一个区块写入到规范链其实就是3步：

1. 调用WriteCanonicalHash函数将数据库中‘h’ + num + ‘n’标记成待插入区块的hash。

2. 调用WriteHeadBlockHash函数将数据库中“LastBlock”标记成待插入区块的hash。

3. 调用bc.currentBlock.Store(block)将BlockChain的currentBlock更新成待插入区块

### 七. reorg方法
    reorg函数功能是处理区块插入时引起的分叉。也就是说待插入区块是插入规范链，但是它的父区块指向的又不是当前规范链的头区块，说明数据中的一个分叉链要升级成规范链。reorg的原理是向下追溯，找到这两条链的共同祖先区块，这个共同祖先区块就是分叉点，然后将新链（待插入区块所在的链）上从分叉点后面所有区块全部重新调用上面的insert方法更新一下规范链，这样原来的规范链就变成了一条分叉链。待插入区块所在的链变成了新的规范链。

上面的代码还涉及到一个概念叫交易查找入口（TxLookupEntry），所谓的交易查找入口就是在数据库中记录了所有规范链上区块中每一个交易的信息，存储这个一个结构有很大的意义，如果你知道一个交易的hash，这个hash就能在数据库中查找有没有这个查找入口，如果没有，说明交易没有上链，如果有，就能快速确定这个交易在哪个区块中，已经在区块中的偏移。存储的数据结构包括3部分内容，这个交易所在的区块的hash（BlockHash）和区块号（BlockIndex）、交易在区块中的偏移（Index）。在数据库中存储的时候是以这个交易的hash值编码后的结果为key，这个数据结构编码后的结果为value存储的。当然只有规范链上的区块中的交易会存储，分叉链上的区块中的交易是不会存储的。所以reorg函数中在回溯老链和新链的时候会把两条链上所有的交易收集起来，然后找出那些老链中不存在于新链的交易，将他们的TxLookupEntry从数据库中删除。

### 八. 在分析Downloader的时候.只有full模式才会调用InsertChain()方法.而fast模式是InsertReceiptChain()方法。我们来看看InsertReceiptChain()方法做了什么.它和InsertChain()方法有什么区别。

1. 首先确保插入的blocks是安次序排列的
2. 调用SetReceiptsData(bc.chainConfig, block, receipts)把收到的交易收据数据加入到block中
3. 调用WriteBody().把blockbody数据写入数据库
4. 调用WriteBlockReceipts().把收据数据写入数据库
5. 调用WriteTxLookupEntries.根据交易hash建立数据库的索引。
6. 更新BlockChain的currentFastBlock信息

