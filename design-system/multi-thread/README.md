# 👌 MULTI THREAD

* [INTRODUCE](introduce.md)
* [Serial and concurency queue](serial-vs-concurency-queue.md)
* [Async vs Sync](asynchronous-vs-synchronous-tasks.md)
* [GCD](gcd.md)
* [Operation Queue](operation-queue.md)
* [Dispatch Group](dispatchgroup.md)
* [Dispatch Semaphore](dispatchsemaphore.md)
* [Async/Await](async-await.md)
* [Question](../question-interview.md)
* [Swift Concurrency](structured-concurrency.md)

## Câu hỏi phỏng vấn tổng quan

**Q: Sync/async vs serial/concurrent?** Sync/async nói caller chờ hay không; serial/concurrent nói scheduling/overlap. Async không đồng nghĩa background.

**Q: GCD vs OperationQueue vs Swift Concurrency?** GCD queues; Operation dependency/state/cancel/limit; Swift Concurrency structured tasks, actor/Sendable.

**Q: Data race vs deadlock?** Data race unsafe concurrent memory access; deadlock chờ vòng tròn. Isolation/lock đúng, tránh blocking main/cooperative executor.
