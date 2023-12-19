> **Area where we invalidate TLB**
***
## Description

- When MMU receives `invalidatePort.cmd.valid = True`
and refill not busy
then
	--> Invalidates all ways in DTLB and ITLB

- When it's done
then
	--> Sends `invalidatePort.rsp.valid = True`


`invalidatePort.cmd.valid` become true when the EnvCall Plugin receives a SFENCE.VMA instruction. This instruction is used for a change of context between two process. 
As TLB does not store ASIDs, it must be invalidated when the process changes, in order to avoid collision between virtual addresses.



***
## Code

```scala
val invalidate = new Area{  
  val requested = RegInit(True) setWhen(setup.invalidatePort.cmd.valid)  
  val canStart = True  
  val depthMax = storageSpecs.map(_.p.levels.map(_.depth).max).max  
  val counter = Reg(UInt(log2Up(depthMax)+1 bits)) init(0)  
  val done = counter.msb  
  when(!done){  
    refill.portsRequest := False  
    counter := counter + 1  
    for(storage <- storages;  
        sl <- storage.sl){  
      sl.write.mask := (default -> true)  
      sl.write.address := counter.resized  
      sl.write.data.valid := False  
    }  
  }  
  
  fetch.getStage(0).haltIt(!done || requested)  
  
  when(requested && canStart){  
    counter := 0  
    requested := False  
    refill.portsRequest := False  
  }  
  
  when(refill.busy || getServicesOf[PostCommitBusy].map(_.postCommitBusy).orR){  
    canStart := False  
  }  
  
  setup.invalidatePort.rsp.valid setWhen(done.rise(False))  
}
```