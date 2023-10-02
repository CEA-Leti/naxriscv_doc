> **State that sends load command to cache memory in order to refill TLB with MMU entry**

There is a CMD state for each TLB level, so for :
-  SV32 : 2 levels --> 2 CMD states
-  SV39 : 3 levels --> 3 CMD states
***
## Description

1. Send `load.cmd.valid` when cache is not in a refill operation because of a previous command.
2. Move to RSP state when `load.cmd.ready` received

#### Reset

After reset, `cacheRefill = 0` and `cacheRefillAny = False` so it can send `load.cmd.valid` in CMD state.

#### Cache refill operation

If MMU receives a `load.rsp.redo` in RSP state, it will come back to CMD state, but this time either:
-  `cacheRefill != 0` because MMU receives `load.rsp.refillSlot != 0`, as cache must perform a refill operation,  cache sends the refill slot. `cacheRefill` remains different from 0 until MMU receives `setup.cache.refillCompletions` for the same refill slot. So, MMU waits for cache refill completion to resend load command.
-  `cacheRefillAny = True` because MMU receives `load.rsp.refillSlotAny = True`,  as cache must perform a refill operation but all cache refill slots are full. So, MMU waits until a cache refill slot is free to resend load command. After resending the command, it will be in the previous case.

***
## Code

```scala
CMD(levelId) whenIsActive{  
  when(cacheRefill === 0 && cacheRefillAny === False) {  
    load.cmd.valid := True  
    when(load.cmd.ready) {  
      goto(RSP(levelId))  
    }  
  }  
}
```