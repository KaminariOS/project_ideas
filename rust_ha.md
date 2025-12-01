# Rust High Availability for blockchain

- [超越内存安全：高可用 Rust 基础设施的挑战与实践](https://www.bilibili.com/video/BV1MUnLztErN)
## **Summary**

The talk discusses real-world security and availability issues in Rust-based blockchain infrastructure, showing that *memory safety alone* does not guarantee system safety.

### **1. Rust Is Widely Used in Blockchain**

Rust is now a dominant language for blockchain infrastructure—especially for smart-contract virtual machines (VMs) like Solana VM, Move VM, and Cosmos WASM. Rust’s memory safety reduces many classic vulnerabilities, but this creates new challenges: bug hunters struggle to find memory-safety bugs, yet systems remain vulnerable in *other* ways.

### **2. Smart-Contract VMs and Their Security Requirements**

A smart-contract VM is a core component of the blockchain “world computer.”
Its main security requirements:

* Prevent memory-safety bugs
* Ensure **availability** — the blockchain must never crash

Even with safe Rust, VMs still crash due to logic errors, panics, resource exhaustion, and dependency issues.

---

## **3. Problem Category 1: Panics**

### **(a) Developer-introduced panics**

Some projects accidentally or carelessly include `assert!`, `panic!`, or `unwrap`.
Example:
In the Move VM verifier, developers reused code that included `assert!`, unintentionally creating a panic path. A panic on-chain causes *all validator nodes* to crash, jeopardizing the entire blockchain.

### **(b) Panics inherited from dependencies**

Large Rust blockchain stacks depend on complex third-party crates.
Example:

* Cosmos WASM → Wasmer → rkyv
* rkyv contains a documented `unwrap` panic
* Cosmos developers were unaware of this indirect panic source

A carefully crafted malicious input can bypass multiple layers of checks and trigger the hidden panic, crashing all nodes. These dependency chains make panics difficult to control or even detect.

### **Catching panics is limited**

* Panics implemented as `abort` cannot be caught
* If panics occur while holding locks, locks become poisoned, breaking system operation even if panic is caught

### **Strategies**

* Fuzzing to find unexpected panic paths
  (e.g., using `arbitrary`)
* Using `cargo clippy` to detect `panic!`, `unwrap`, `expect`, etc.
* Documenting panic conditions clearly
* Checking panic conditions before calling risky functions

---

## **4. Problem Category 2: Resource Exhaustion**

Even without memory-unsafe bugs, Rust systems can still be DOS’ed via CPU or memory exhaustion.

### **(a) Query APIs**

Without limits, users can request extremely large datasets, causing:

* Heavy CPU usage
* Large memory usage → OOM kill → process termination

### **(b) Smart-contract execution**

Smart contracts are often Turing-complete, so blockchain VMs use **gas metering** to cap CPU and memory usage.

### **Case study: Incorrect memory-usage accounting**

Suite/Aptos VM miscalculated memory for `Vec<T>` as:

```
vec.len() * size_of::<T>()
```

But even an empty vector (`len = 0`) allocates memory (capacity > 0).
Therefore malicious contracts could create many empty vectors:

* Consuming **100+ GB** RAM
* Without consuming gas

Broadcasting such transactions could make all validators run OOM and crash the network.

### **Prevention**

* Avoid infinite loops and extremely long loops
* Model memory and CPU usage accurately
* Use meters that correctly track resource usage
* Ensure metering cannot be bypassed
* Use profiling/visualization tools (e.g., ByteHound, perf) to identify hotspots

---

## **5. Other Availability Risks**

Rust avoids memory-safety bugs but does **not** eliminate:

* `unsafe` code issues
* Memory leaks
* Deadlocks / livelocks / starvation
  These can also break blockchain node availability.

---

## **6. Conclusion**

Rust is powerful for infrastructure, but real-world blockchain systems still face serious reliability challenges. Key takeaways:

* Avoid panic-prone constructs (`unwrap`, `assert!`) in high-availability systems
* Document panic conditions clearly
* Be aware of and audit dependency chains
* Apply strict and accurate resource limits
* Continuously test with fuzzing and profiling

Even simple mistakes—like an unnecessary `assert!` or incorrect gas calculation—can crash an entire blockchain network. Improving panic hygiene, dependency quality, and resource metering is essential for safer Rust ecosystems, especially in blockchain infrastructure.



- 
  ```
    00:00:04	大家好,今天我来给大家分享一下RAST在超越内存安全这个视角上的一些实践与经验。目前RAST在基础设施这个领域是非常广泛的应用。因为RAST有一个很核心的特性,就是它很安全。而这对于开发者来说,我们可以更轻松地写出一些比较安全的程序。

    00:00:33	可以省下很多与这些内存安全漏洞做斗争的时间。但是对于安全研究人员来说,他们去在这个Rust的系统里面寻找bug就是非常困难,因为Rust已经做得很安全。它的编译器可以排除很多内存安全的问题。

    00:00:55	目前在区块链这个领域﹐很多基础设施是使用RUST开发的。我可以看到这个右边有一个表格﹐很多排名比较靠前的这些区块链项目﹐他们是使用RUST去开发的。RUST开发的区块链的基础设施里面﹐有一个比较重要的基础设施类型就是﹐

    00:01:18	这就是智能合约虚拟机。智能合约虚拟机就是为区块链提供智能合约知识的一个组件。目前有很多新的智能合约虚拟机,比如说SolanaVM,MoveVM,CosmosVM,这些VM都是使用RUST去开发的。所以说这个目前内存安全已经成为区块链里面一个相当于标准的一个东西。另外就是RUST是区块链智能合约虚拟机这些基础设施的一个趋势。

    00:01:50	区块链的智能合约虚拟机在区块链里面是一个什么角色?在说这个问题的时候,我们可以把区块链看作一个执行智能合约的世界计算机。智能合约虚拟机就是这个世界计算机里面一个核心的基础设施。

    00:02:06	它的工作流程可以简化为﹐用户提交一个Transaction﹐然后这个Transaction在区块链网络中进行广播﹐广播到所有的Validator里面。然后Validator收到交易之后﹐去在智能合约虚拟机里面﹐执行这些智能合约。在执行过程中﹐有些不同的VM有不同的执行过程﹐大概可以分为预处理的过程﹐还有执行的过程。

    00:02:33	这个就是智能合约虚拟机在区块链里面的简单的工作流程。它有什么安全的需求和风险呢?首先,它最基础的安全要求就是保证management safety。比如说,我们要避免内存安全漏洞带来一些危害,就是这些传统的一些安全问题。比如说,

    00:02:57	进一步,我们还有一个需求,就是要保证区块链的可用性。

    00:03:16	区块链网络始终是可用的,不能当机,一当机就会出很多的严重的安全问题。比如说用户就无法执行任何的交易,这会在市场上引起恐慌,导致数字资产的价格下跌之类的事情。很多crash是由内存安全漏洞导致的,但是rust又避免了一些内存安全漏洞。

    00:03:43	并不等于保证的可用性。在RAS里面,即使你完全使用SafeRAS去编写代码,这个系统仍然会

    00:03:55	我发现了很多严重的安全漏洞,累计获得了超过一百万美元的bug bounty。

    00:04:22	这个发现是在区块链领域被广泛认可的。首先介绍第一个问题就是可以触发的panic。

    00:04:33	Panic就是Rust处理不可恢复错误的一个措施。它是可以强制的并且安全的终止执行,阻止不可恢复错误的传播。这个影响会导致程序强制退出,其实和Crash的影响是差不多的,都是导致程序退出。

    00:04:57	而panic在RAS里面又是一个比较常见﹖有一个很简单的panic例子。可能大家在写RAS的代码的时候都会写出这样的代码﹖或者遇到panic这样的情况。随着我们的项目越来越大﹖然后参加到项目的contributor越来越多﹖panic的情况就会从简单的情况变成很复杂的一种场景。这就增加了当机的风险。

    00:05:26	现在我要介绍一个例子。在介绍这个例子之前,首先介绍一些背景。在Blockchain的链上的组件里面有一个组件叫Wirefare。这是MOO虚拟机里面引入的一个组件。它可以在链上去验证智能合约是否满足某些安全属性。这个可以理解为把RUST的编译检查挪到链上。因为用户通过MOO编译器编译出来的

    00:05:56	我们发现了这个panic的问题就是在verifier里面的。

    00:06:20	这个问题就是,开发者自己引入了一个Panic,自己调了一个Azure的红,然后在这里引入了Panic。但是他为什么会这样做呢?按理说这个是一个对可能性要求非常高的系统,但他却在这里面引入了一个会触发Panic的代码。因为他直接复用了一些代码,而这个代码最初不是说要在链上使用的。他复用了这些代码并且没有删除Azure。这就会导致,

    00:06:48	这就是第一类问题,开发者主动引入的panic。

    00:07:13	它就是开发者在码里面引入一些众所周知会punic的红或者是functions,比如说punic红,unwrap这些function。

    00:07:23	有些是开发者他知道﹐他故意引入﹐他知道是这个地方会PANIC﹐他甚至会写PANIC DOC﹐就告诉你这个什么时候会PANIC。第二种就是他﹐他也不知道他会不会PANIC﹐但是反正就是写上去了﹐我也不知道这个会不会PANIC﹐我只是想去排除一些不应该发生的情况。这个就是刚才那个﹐刚才那个Azure的例子。那么第二类PANIC﹐

    00:07:50	在讲这个之前,也是有一些背景的知识介绍。Cosmos WASM是另一个智能合约虚拟机。它支持了WASM智能合约的执行。

    00:08:03	为了让WASM执行变得更加的高效,他做了一个AOT。用户可以发布WASM Code到链上﹐YZiter可以编译这些WASM Code变成X86并且存到硬盘中。在执行的时候就加载这些已经编译过的数据。

    00:08:26	这里面涉及到一些RUS的开源项目﹖比如说WASMR提供了WebAssembly的Runtime﹖并且里面有AOT的支持﹖它可以编译并且执行WASMR代码。在它里面使用了一个叫RKYAV的库﹖这个库在WASMR里面负责把编译后的代码保存到硬盘中。

    00:08:49	这些都是不同的开发者,不同的team去开发的项目。整个依赖的图就像这个图中显示的,他们是层层嵌套了一个依赖关系,并且项目的代码量也是比较多的。

    00:09:04	对于RKYV和WASMR来说,他们并没有必要去保证自己是不会PANIC的,因为他们对可用性的要求不是很高。但对于Cosmos WASMR来说,他是应该保证自己不会PANIC,但他并不知道他的依赖这些项目中的这些PANIC风险。

    00:09:23	那就会出现这样的一个问题﹖无意中引入的panic﹖在RKVV里面﹖开发者有意引入了一个unwrap﹖并且写了panic的道口﹖告诉开发者﹖这个地方什么时候会panic﹖但是对于Wasmr和Cosmos Wasm﹖甚至Cosmos Wasm Proxing这些项目来说﹖他并不知道这里面有一个潜在的panic﹖

    00:09:49	由于依赖关系的层层线道非常复杂,这种PANIC就非常难以注意并且难以控制。即使层层依赖导致这个恶意输入的传播路径充满了检查,但是我们仍然可以去触发这个PANIC,就是用一个很小的输入绕过所有的检查去触发这个PANIC,

    00:10:18	感谢观看﹗

    00:10:34	他也分为两类,一个是没有写文档,就开发者根本就不知道从哪知道这个panic了,因为他依赖那个项目也没有写panic文档,比如说这个wasmer里面这种panic,他根本就不知道这里面会panic。然后第二就是虽然开发者写了panic,但是在调用这些函数的时候,那些开发者他并没有去

    00:10:57	关注这个PanicDoc,它没有去排除Panic的条件,却调用了这些函数,最终导致一个可以触发Panic的输入进入到这个函数中。这是两类的Panic。我们在避免这个Panic造成严重影响的时候,会使用Cash and Win的这个东西去防止Panic直接导致进城退出。

    00:11:22	这个在很多情况是有用﹐但是它有两个局限性﹐一个是如果说panic等于about这种实现方法﹐那它是无法catch的﹐因为程序直接调用about,系统调用推出了。第二种情况就是﹐如果这个程序中有很多的lock﹐那么在panic之后﹐即使是被catch and win的捕获﹐那么这个lock会进入pointing的状态﹐必须要进行处理﹐否则就无法正常的运行。

    00:11:56	那么如何去在这个项目中避免这个panic造成很严重的影响?首先我们可以去尝试去找到这些panic,就是利用魔物测试这种方法。

    00:12:10	这样就可以发现一些非预期的输入。我们可以用Abitrary这个项目去构造输入。可以构造任何类型的输入。从开发者的角度来说,我们要尽量避免在代码中引入panic。

    00:12:27	这里可以用cargo clip的一些选项去检查代码中是否有panic﹖比如说enrive used这些东西﹖去尽量让代码中减少引入这些会导致panic的函数或者红。如果说我不得不写panic﹖也是要尽量在文档里面写明白这个函数是会panic的﹖并且它在什么情况会panic﹖这也为后续的开发者提供了便利﹖不至于让大家不知道这个函数会panic﹖

    00:12:56	当我们调用有panic条件的函数的时候﹐我们应该去检查panic条件﹐把这些条件给排除掉﹐这样函数不会触发panic。

    00:13:10	那么这个就是第二个问题,资源耗尽的问题。资源耗尽这个问题首先有一个简单的例子,就是我们在写query API的时候,用户可以输入一个数量,query API会返回出这么多数量的数据。那么如果没有做任何的限制,用户可以输入很大的一个数字,让query API去查询。那么就有两个问题,一个是查询的过程,它会耗尽CPU。

    00:13:37	这就会导致你的系统的CPU耗尽,整个系统变得非常的缓慢,甚至无法正常的运行。第二就是,你查询到的结果数量很多,它会占用很大的内存。那最终会导致内存耗尽,占满了系统内存,这个进程就会被OMQ了,给强制终止。那如果缺少或者说没有资源的限制,就会导致资源耗尽。

    00:14:06	那么在区块链里面,由于区块链是一个执行智能合约的世界计算机,而智能合约,很多智能合约是图灵完备的,也就是说它可以无限循环。

    00:14:16	可以干很多的占用内存﹐或者说是占用CPU的事情。所以对于智能合约的执行﹐这个方面是一定要进行资源限制的。因为首先CPU为了防止无线循环耗尽CPU﹐CPU是要被限制的。然后防止无线增长的内存耗尽内存资源的内存也是要被限制的。在区块链的智能合约虚拟机里面﹐大家会使用GET这种机制去限制资源的使用。

    00:14:47	Gaiss就是这个智能合约可以使用的资源上限。当Gaiss耗尽的时候,这个智能合约就会被强制中止执行,避免它消耗更多的资源。

    00:15:00	在数据中就会对每种不同的操作进行盖茨插着去,去限制它的资源。比如说一些这个指令,站指令啊,计算指令啊,都会计算出这个指令消耗的盖茨。对于一些需要内存分配的指令,

    00:15:20	这种质量需要分配内存,所以从内存消耗的角度去计算它的消耗。

    00:15:32	在数据里面就会出现一个问题。它在charge gas的时候,比如charge vector消耗了多少内存资源的时候,它出现了一个错误。它用vector的长度去计算vector的长度。

    00:15:50	可以看到这个例子里面,他用了长度乘上U8的大小去计算Vector占用的内存。如果长度是零,那算出来就是零。但是Vector即使是长度为零,他也分配了内存。

    00:16:09	这里就会导致它的Gas和实际的内存占用是不符的。比如说,如果说构造大量的空的Vector在一个智能活跃里面去执行,那么就可以在不消耗Gas的情况下消耗大量内存。我们的实验就是它可以超过100GB的内存。这样就会导致一个恶意的

    00:16:33	在区块链网络中广播,就会导致所有的Validator Node占用超过100GB的内存,最终导致网络当机。

    00:16:48	在资源耗尽这一块,我们在开发的时候首先应该避免一些逻辑错误会导致死循环这种情况出现。因为有些循环条件的计算一旦出错,

    00:17:05	请不吝点赞 订阅 转发 打赏支持明镜与点点栏目

    00:17:21	还有就是要避免这个长时间循环的执行。虽然说它不是死循环,但是如果说这个操作过于的复杂,循环执行的次数过于多的话,那它仍然在一定程度上可以说是一种耗尽了CPU资源。另外对于内存资源的时候,就要在对内存资源建模的时候是要,

    00:17:41	是要准确的建模﹖而不是说虽然做了一些计算﹖或者说做了一些内存资源的建模﹖但是却错了﹖比如说苏伊刚才计算Vector内存占用头﹖他使用了Lens去乘上大小﹖那这个显然是一种错误﹖在他修复的时候﹖他加上一个常数﹖那这样这个计算就变得正确了﹖

    00:18:02	还有一种方法就是Meter,目前在Aptos和Suite里面都在使用,对一些比较复杂的操作用一个Meter去计算这个操作的资源使用情况,并且在耗尽资源之前强制终止这个过程。

    00:18:19	但是这个meter一定要保证它必须是准确﹖它计算不能出现错误﹖它不能没有反应资源的使用﹖另外它也不能被绕过﹖就是一些工艺者可能会提交一些输入去绕过这个meter的限制﹖然后耗尽它的资源。

    00:18:37	另外就是我们在调试的时候﹐可以使用一些可视化的工具﹐这个比如是ByteHunt、Perf这些工具﹐这个是我们在分析漏洞的时候﹐也会用到这种工具﹐它可以在一个可视化的环境下面﹐就告诉你哪个函数﹐或者哪个结构体消耗了内存﹐或者占用CPU比较多﹐它对于定位这些问题是比较有用的。

    00:19:03	虽然说今天讲了这两个问题﹖可能性仍然是一个非常难以保证的一个问题﹖我们仍然任重而道远。我们只讨论这两个问题﹖但是还有很多别的角度需要考虑﹖比如说unsave﹖这个unsave就在RAS里面引入了那层安全问题﹖但是unsave是不可避免的﹖比如说我们说有些FFI接口﹖但是unsave是不可避免的﹖

    00:19:32	还有就是Memory Leak。这个Rust是并不保证没有Memory Leak的。所以Memory Leak也可能会导致内存资源耗尽的问题。下面这个例子就是Rust文档里面一个经典的例子,就是一个循环引用的问题。还有就是如果说要写并发的代码的时候,比如说会出现死锁、活锁或者饥饿的问题,这也会导致程序无法正常运行,导致可用性被破坏。有很多与可用性相关的问题需要我们考虑。

    00:20:08	目前Rust在基础设施领域是被广泛的应用的。我们在构建Rust的基础设施的时候﹐是有很多独特的挑战﹐比如说刚才我们提到的﹐主要讲了Panic的问题﹐还有资源耗尽的问题﹐以及我们后面补充了Unsafe﹐还有并发这些问题。

    00:20:38	我们的这些发现主要是在panic和资源耗尽这两部分。它展示出了一些可用性的,在RAS里面常见的一些缺陷。

    00:20:51	这些分析都来自于真实的区块链像﹖比如说Aptos、Suite、Cosmos。我们今天给的例子是Aptos、Suite、Cosmos、VM里面的一些例子。其实都不是很复杂的例子﹖它都是一些很简单的错误导致的﹖比如说在Aptos的例子里面﹖就是开发者他自己在RAS代码里面

    00:21:18	他写了一个Azure,他完全没必要这样写。他可以用Error,或者用别的什么方法。就是没有必要在这里面写一个Azure,但是他却在这里面写了一个Azure。而这个Azure,这个代码又是在链上的逻辑。所以一旦这个Azure被触发,所有的节点都会panic。这就是一个很简单的一个代码的,没有符合最佳实践的问题,但却会导致很严重的后果。

    00:21:47	Cosmos 那个问题,开发者其实并没有想主动引入 Panic,但由于依赖关系过于复杂,他没有办法去保证他依赖的项目里面没有 Panic。这样就会导致他在无意间在自己的代码中引入了 Panic 的这种安全风险。最终也是会导致所有的区块链节点当机。而且 Cosmos 又是一个

    00:22:15	有一个类似于VM基础设施的东西﹖很多区块链可以基于Convice去构建﹖它里面出现了panic问题﹖所有基于它构建的区块链都会出现这种问题。这个也是一个很严重的安全问题﹖它是由依赖的项目导致﹖而不是他们的开发者自己在里面引入panic导致的。

    00:22:39	在资源耗尽方面,我举了一个资源耗尽的例子。这个就是在术语里面的计算内存资源使用不准确的情况。在计算内存资源使用的时候,术语使用了vector的长度乘上vector元素的大小这种方法,

    00:23:04	这种计算方法在空的时候,长度为零,乘上数据的大小,算出来的内存使用就是零。这就会导致它计算出来的内存使用实际上是小于真实的内存使用。用这种错误的方法去限制内存的使用,就会导致

    00:23:29	如果我们想构建更加安全的RAS基础设施﹖我们可以考虑在系统中避免panic﹖

    00:23:54	例如CPU资源耗尽和内存资源耗尽这些情况。对于PANIC,在对可能性要求比较高的系统中,我们不要主动引入PANIC红、THIRD红,或者掉UNWRAP这种方法,应该有更优雅的方式去处理这些错误。

    00:24:15	对于无意间从依赖向引入的panic,我只能说希望RUS的社区能够重视这个问题。在开发这些可能被很多项目依赖的这些比较基础的creator的时候,

    00:24:33	尽量不要引入punic﹖我们即使不得不引入punic﹖也要从文档中告诉后来的开发者﹖这个地方是会punic的﹖并且这个函数会在什么情况上punic﹖便于让后续的开发者﹖知道如何在自己的项目中﹖排除这些punic条件﹖并避免punic的触发。

    00:24:59	对于资源耗尽这个问题,我们在开发RAT项目的时候,应该时刻考虑资源是否应该限制的问题。有些时候不做资源限制可能也不会出什么问题,比如说资源使用量非常的小,这个时候你不做资源限制可能也不会出什么问题。但是一旦这个项目变得复杂,

    00:25:28	项目的需求变得越来越多,越来越复杂。里面使用的结构体或者接口越来越复杂,提供了很多更为复杂的功能。这也会使资源耗尽的风险不断地增加。在开发这样的需求的时候,我们也要考虑内存和CPU这些资源是否应该限制,或者是否准确地限制。

    00:25:57	这样的话就会尽可能在项目中避免资源耗尽的这种风险,让我们的项目更加的安全。最后谢谢大家,希望RUST生态更加的安全。
  ```
