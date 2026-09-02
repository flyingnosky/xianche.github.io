# Hardware Fences
## Purpose and Scope
Hardware Fences是Gen7硬件调度框架中提供GPU-GPU同步，而不需要CPU参与。本文档包含hardware fences的创建，提交，响应，throttling以及回复机制。

## Overview
Hardware Fences通过允许GMU firmware在硬件中管理fence signal，使能更精细化的GPU工作负载同步，减少CPU参与。当context提交一个包含hardware fences的SYNCOBJ drawobj时，它们通过HFI信息传递给GMU。GMU跟踪
fence completion并signal给下游的consumer，比如TXQueue, GPU retire对应的时间戳。
重要特性：
- Zero CPU overhead：fence signal直接来自硬件GMU，发送给硬件Consumer
- GMU-managed：firmware处理fence的生命周期和signal
- Flow-controlled：限制可以防止大量没有响应的fence涌入GMU
- Recoverable：全面处理fault场景

## Architecture and Components
### Hardware Fence Manager(adreno_hwsched_hw_fence)
跟踪全局hardware fence状态：
- unack_count：已经发送但GMU还未响应的fence数目
- defer_drawctxt：在throttling时deferred的context
- defer_ts：fence被defer的时间戳
- flags： 用于控制throttling和abort的状态位
- unack_wq：在throttling时等待阻塞队列
- seqnum：HFI信息的atomic sequence number
### Hardware Fence Entry(adreno_hw_fence_entry)
Per-fence tracking structure:
- cmd: HFI命令结构体（hfi_issue_hw_fence_cmd）
- drawctxt：拥有的context
- kfence：相关的内核fence object
- node：context fence list的linkage
- reset_node： 在恢复时linkage
