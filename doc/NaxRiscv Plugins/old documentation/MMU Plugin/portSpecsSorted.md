> List of ports sorted by priority (highest number = highest priority)

***
## Description

In NaxRiscv current implementation, there is 3 ports(in order of priority) :
- Load port with priority = 1
- Store port with priority = 1
- Fetch port with priority = 0

For each port, the variable `storage` stores the storage spec associated, so the TLB associated.

Each port is associated to a new Composite. In this Composite, there is various Area.

#### read 

1) Read ways in each storage level with virtual address, and store the page table entires in `ENTRIES`.
2) Compare virtual address with virtual address stored in each entry, to detect TLB hits. We store the results into `HITS`.

#### ctrl

1) With `HITS` check if there is a hit.
2) If yes, extract data from entry. If there is more than one hit, select the first entry than gives a hit.
3) Check if it's a memory access that needs MMU (peripheral,...)
4) Ask refill, if `hit = False` and need MMU translation
5) Send response to the port by Stageable variables shared with pipeline 

<table>
  <tr >
    <th style="text-align: center;" >Stageable name</th>
    <th style="text-align: center;" >need MMU translation</th>
    <th style="text-align: center;" >Value</th>
  </tr>
  <tr style="text-align: center;" >
    <td >REDO</td>
    <td style="text-align: center;" rowspan="8">Yes</td>
	<td >If hit, False, else True</td>
  </tr>
  <tr style="text-align: center;" >
	<td >TRANSLATED</td>
    <td >If hit, physical address, else 0</td>
  </tr>
  <tr style="text-align: center;" >
	<td >ALLOW_EXECUTE</td>
    <td >If hit, X flag & !(U flag & current priv = S) , else False
    False if physical address not in instruction range</td>
  </tr>
  <tr style="text-align: center;" >
	<td >ALLOW_READ</td>
    <td >If hit, R flag | (MXR flag & X flag), else False</td>
  </tr>
  <tr style="text-align: center;" >
	<td >ALLOW_WRITE</td>
    <td >If hit, W flag, else False</td>
  </tr>
  <tr style="text-align: center;" >
	<td >PAGE_FAULT</td>
    <td >If hit, page fault flag | (U flag & current priv = S) | (!SUM flag | !U flag & current priv = U) , else False</td>
  </tr>
  <tr style="text-align: center;" >
	<td >ACCESS_FAULT</td>
    <td >If hit, access fault flag, else False</td>
  </tr>
    <tr style="text-align: center;" >
	<td >BYPASS_TRANSLATION</td>
    <td >False</td>
  </tr>
  
    <tr style="text-align: center;" >
    <td >REDO</td>
    <th style="text-align: center;" rowspan="8">No</th>
	<td >False</td>
  </tr>
  <tr style="text-align: center;" >
	<td >TRANSLATED</td>
    <td >virtual address( = physical address)</td>
  </tr>
  <tr style="text-align: center;" >
	<td >ALLOW_EXECUTE</td>
    <td >True else if physical address not in instruction range</td>
  </tr>
  <tr style="text-align: center;" >
	<td >ALLOW_READ</td>
    <td >True</td>
  </tr>
  <tr style="text-align: center;" >
	<td >ALLOW_WRITE</td>
    <td >True</td>
  </tr>
  <tr style="text-align: center;" >
	<td >PAGE_FAULT</td>
    <td >False</td>
  </tr>
  <tr style="text-align: center;" >
	<td >ACCESS_FAULT</td>
    <td >False</td>
  </tr>
      <tr style="text-align: center;" >
	<td >BYPASS_TRANSLATION</td>
    <td >True</td>
  </tr>
  <tr style="text-align: center;" >
	<td >IO</td>
	<td >Yes/No</td>
    <td >If physical address is in io range, True, else False</td>
  </tr>
</table>


***
## Code

```scala
val portSpecsSorted = portSpecs.sortBy(_.ss.p.priority).reverse  
val ports = for(ps <- portSpecsSorted) yield new Composite(ps.rsp, "logic", false){  
  import ps._  
  
  val storage = storages.find(_.self == ps.ss).get  
  
  
  readStage(ALLOW_REFILL) := (if(allowRefill != null) readStage(allowRefill) else True)  
  val allowRefillBypass = for(stageId <- pp.readAt to pp.ctrlAt) yield new Area{  
    val stage = ps.stages(stageId)  
    val reg = RegInit(True)  
    stage.overloaded(ALLOW_REFILL) := stage(ALLOW_REFILL) && !storage.refillOngoing && reg  
    reg := stage.overloaded(ALLOW_REFILL)  
    when(stage.isRemoved || !stage.isStuck){  
      reg := True  
    }  
  }  
  
  val read = for (sl <- storage.sl) yield new Area {  
    val readAddress = readStage(ps.preAddress)(sl.lineRange)  
    for ((way, wayId) <- sl.ways.zipWithIndex) {  
      readStage(sl.keys.ENTRIES)(wayId) := way.readAsync(readAddress)  
      hitsStage(sl.keys.HITS_PRE_VALID)(wayId) := hitsStage(sl.keys.ENTRIES)(wayId).hit(hitsStage(ps.preAddress))  
      ctrlStage(sl.keys.HITS)(wayId) := ctrlStage(sl.keys.HITS_PRE_VALID)(wayId) && ctrlStage(sl.keys.ENTRIES)(wayId).valid  
    }  
  }  
  
  
  val ctrl = new Area{  
    import ctrlStage._  
  
    val hits = Cat(storage.sl.map(s => ctrlStage(s.keys.HITS)))  
    val entries = storage.sl.flatMap(s => ctrlStage(s.keys.ENTRIES))  
    val hit = hits.orR  
    val oh = OHMasking.firstV2(hits)  
  
    def entriesMux[T <: Data](f : StorageEntry => T) : T = OhMux.or(oh, entries.map(f))  
    val lineAllowExecute = entriesMux(_.allowExecute)  
    val lineAllowRead    = entriesMux(_.allowRead)  
    val lineAllowWrite   = entriesMux(_.allowWrite)  
    val lineAllowUser    = entriesMux(_.allowUser)  
    val lineException    = entriesMux(_.pageFault)  
    val lineTranslated   = entriesMux(_.physicalAddressFrom(ps.preAddress))  
    val lineAccessFault  = entriesMux(_.accessFault)  
  
    val requireMmuLockup  = satp.mode === spec.satpMode  
    val needRefill        = isValid && !hit && requireMmuLockup  
    val askRefill         = needRefill && overloaded(ALLOW_REFILL)  
    val askRefillAddress = ctrlStage(ps.preAddress)  
  
    requireMmuLockup clearWhen(!status.mprv && priv.isMachine())  
    when(priv.isMachine()) {  
      if (ps.usage == LOAD_STORE) {  
        requireMmuLockup clearWhen (!status.mprv || priv.logic.machine.mstatus.mpp === 3)  
      } else {  
        requireMmuLockup := False  
      }  
    }  
  
    import ps.rsp.keys._  
    IO := ioRange(TRANSLATED)  
    when(requireMmuLockup) {  
      REDO          := !hit  
      TRANSLATED    := lineTranslated  
      ALLOW_EXECUTE := lineAllowExecute && !(lineAllowUser && priv.isSupervisor())  
      ALLOW_READ    := lineAllowRead || status.mxr && lineAllowExecute  
      ALLOW_WRITE   := lineAllowWrite  
      PAGE_FAULT    := lineException || lineAllowUser && priv.isSupervisor() && !status.sum || !lineAllowUser && priv.isUser()  
      ACCESS_FAULT  := lineAccessFault  
    } otherwise {  
      REDO          := False  
      TRANSLATED    := ps.preAddress.resized  
      ALLOW_EXECUTE := True  
      ALLOW_READ    := True  
      ALLOW_WRITE   := True  
      PAGE_FAULT    := False  
      ACCESS_FAULT  := ps.preAddress.drop(physicalWidth) =/= 0  
    }  
  
    ALLOW_EXECUTE clearWhen(!fetchRange(TRANSLATED))  
  
    BYPASS_TRANSLATION := !requireMmuLockup  
    WAYS_OH       := oh  
    (WAYS_PHYSICAL, entries.map(_.physicalAddressFrom(ps.preAddress))).zipped.foreach(_ := _)  
  }  
  ps.rsp.pipelineLock.release()  
}
```