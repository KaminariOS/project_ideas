This video is a Diamond Sponsor Talk from RustConf 2025 by **Alessandro Decina** from Anza (Solana's performance team), titled **"Blazing-Fast Magic Beans."**



The presentation provides an in-depth look at how Solana is leveraging **Rust** for high-performance, system-level programming to build a high-throughput blockchain platform.



---



## Solana's Vision and Architecture



Alessandro Decina describes Solana not as a traditional blockchain, but as a "financial transactions database" [[00:43](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=43)] with the goal of becoming the platform for the **"internet capital markets"** [[01:18](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=78)].



* **Scale and Impact:** In the first half of the year, exchanges on Solana generated over **$1.2 trillion in volumes** [[02:02](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=122)], and applications deployed on the network generated over **$1.6 billion** [[02:09](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=129)].

* **System Overview:** The network is a **spicy distributed database** where time is divided into **400-millisecond slots** [[02:47](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=167)]. Leaders rotate, execute transactions, and non-leader nodes replay them before voting on the final state to reach consensus [[02:56](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=176)].

* **On-Chain Programs:** The logic that runs on the blockchain is written in **pure Rust**, compiled to **eBPF** (Extended Berkeley Packet Filter), and executed in a user-space virtual machine [[03:52](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=232)].

* **Parallelization:** The system achieves efficiency through parallelization by utilizing **read-write locking on accounts** [[05:08](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=308)]. If multiple programs only **read** an account, they can run in parallel, but a program **writing** to an account is guaranteed exclusive access [[05:20](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=320)].



---



## Performance Enhancements with Rust



A major focus of the talk is on the **IBRL (Increase Bandwidth, Reduce Latency)** effort, showcasing several high-performance system-level changes:



* **QUIC Protocol (Quinn Crate):** Solana implemented and contributed several changes to the open-source Rust QUIC crate, **Quinn**, to improve UDP ingestion, reduce memory allocations, and reduce transaction fragmentation for lower latency [[07:05](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=425)].

* **Storage Layer Rewrite (IO_uring):** The team rewrote the storage layer using a new, purely Rust, non-async wrapper around **IO_uring** (a Linux kernel interface for asynchronous I/O). This change allows for processes like unpacking a 100 GB tarball during a cold boot to be done nearly **three times as fast** [[08:05](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=485)].

* **Networking Layer (XDP):** The networking layer is being rewritten to use **XDP** (eXpress Data Path) for **kernel bypass**, with a new open-source crate, **Agami XDP** [[08:41](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=521)]. This implementation has been shown to be extremely fast, generating less than three context switches per second while sending 2.5 million packets per second [[09:16](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=556)].



---



## High-Powered Profiling



The team created a new, powerful profiler called **Zolana**, written completely in Rust, which is based on **eBPF** [[09:42](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=582)].



* **Deep Visibility:** Zolana can profile execution that regular profilers miss, such as code that is **off-CPU** (like threads that are sleeping/waiting), disk I/O, context switches, and memory allocations [[10:01](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=601)].

* **Detailed Analysis:** It supports custom markers (like "slot spans") to narrow down a profile to a specific time range [[10:36](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=636)] and provides line-by-line and disassembly views, integrating with the **profiler.firefox.com** UI [[10:57](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=657)].



---



## Performance Results and Call to Action



The talk concludes by highlighting the raw performance numbers achieved by the team:



* The public cluster has been pushed to execute over **100,000 transactions per second** [[12:21](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=741)].

* Internal scheduler tests have been shown to execute over **1 million transactions per second** [[12:33](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=753)].



The speaker encourages Rust developers to be open-minded and consider joining the Solana ecosystem, noting that the platform's core is powered by cutting-edge, low-level Rust technology and that there are over 500 open jobs available [[12:44](http://www.youtube.com/watch?v=AqKFBEvwqqg&t=764)].



You can view the video here: [http://www.youtube.com/watch?v=AqKFBEvwqqg](http://www.youtube.com/watch?v=AqKFBEvwqqg)




