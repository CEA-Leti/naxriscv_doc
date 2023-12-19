> Function that stores Page Table Entry into TLB and returns to IDLE state

***
## Description

The for loop goes through the `storages` list, for each `storage` element (TLB) :

1) With `levelId`, the function selects the right `storageLevelId` (TLB level). It selects the maximum level that is less than or equal to `levelId`.
	For example, with a 2-ways TLB configuration (id = 0 or 1) :
	- if `levelId = 2` (doneLogic after RSP(2)), `storageLevelId = 1` 
	- if `levelId = 1` (doneLogic after RSP(1)), `storageLevelId = 1` 
	- if `levelId = 0` (doneLogic after RSP(0)), `storageLevelId = 0` 

2) With `storageLevelId`, it selects the right `storageLevel` area in each TLB

3) We have the right TLB level selected in each TLB, but we have now to select the right TLB, in order to store the data. We manage it thanks to `sel` variable.
	To do this, we first filter the TLBs by their associated ports, and we see if it's the associated port that leads to this refill operation. In this case, `sel = True`.
	For example, if it's LSU load that leads to the refill operation :
	- ports(0) (LSU load) is associated to DTLB, so `sel = True` for DTLB
	- ports(2) (Fetch load) is associated to ITLB, so `sel = False` for ITLB

4) With `storageLevel.write`, we write in the selected level in each TLB. But with `storageLevel.write.mask`, we store the data, only in the right TLB, with `sel`, and the selected way, with `allocId`. 
	So, we write as many times as there are TLBs, but in fact we store the data in one TLB.

5) It increments `allocId`, to store in the next way, next time we write in the same level.

6) Finally, we return to IDLE state to manage another refill operation.

#### `storageLevel.write` fields

Name | Type | Description
:----: | :----: | :----: 
`mask` | Bits(number of ways bits) | Select in which way we store, 1 bit is set
`address` | Bits(log2Up(TLB depth) bits) | address where we store the data, it's a part of virtual address that leaded to refill operation

The virtual address part selected depends on the TLB level where we store and TLB depth.
For example, for SV39 and depth = 32, if :
- TLB level = 2, `address = virtual(34 downto 30)`
- TLB level = 1, `address = virtual(25 downto 21)`
- TLB level = 0, `address = virtual(16 downto 12)`

TLB data fields | Description
:----: | :----:
valid | TLB line validity
pageFault | load.exception or load.levelException(levelId) or !load.flags.A
accessFault | !storageLevel.write.data.pageFault && load.levelToPhysicalAddress(levelId).drop(physicalWidth) =/= 0
virtualAddress | virtual(specLevel.virtualOffset + log2Up(storageLevel.slp.depth), widthOf(storageLevel.write.data.virtualAddress) bits)
physicalAddress | (load.levelToPhysicalAddress(levelId) >> specLevel.virtualOffset).resized
allowRead | load.flags.R
allowWrite | load.flags.W && load.flags.D
allowExecute | load.flags.X
allowUser | load.flags.U

##### virtualAddress

<table>
  <tr >
    <th>SATP Mode</th>
    <th>levelId</th>
    <th style="text-align: center;">virtual address stored (offset, width bits)</th>
  </tr>
  <tr style="text-align: center;" >
    <th style="text-align: center;" rowspan="2">SV32</th>
	<td >1</td>
    <td >virtual(22 + log2Up(way depth), 10 - log2Up(way depth) bits)</td>
  </tr>
   <tr style="text-align: center;" >
	<td >0</td>
    <td >virtual(12 + log2Up(way depth), 20 - log2Up(way depth) bits)</td>
  </tr>
  <tr style="text-align: center;" >
    <th style="text-align: center;" rowspan="3">SV39</th>
    <td >2</td>
    <td >virtual(30 + log2Up(way depth), 9 - log2Up(way depth) bits)</td>
  </tr>
  <tr style="text-align: center;" >
    <td >1</td>
    <td >virtual(21 + log2Up(way depth), 18 - log2Up(way depth) bits)</td>
  </tr>
  <tr style="text-align: center;" >
    <td >0</td>
    <td >virtual(12 + log2Up(way depth), 27 - log2Up(way depth) bits)</td>
  </tr>
</table>

For example, with SV39, way depth = 32, and levelId = 0 :
`virtualAddress` = virtual(12 + 5, 27 - 5 bits) = virtual(17, 22 bits) = virtual(38 downto 17)

##### physicalAddress

<table>
  <tr >
    <th>SATP Mode</th>
    <th>levelId</th>
    <th style="text-align: center;" >Physical Address stored</th>
  </tr>
  <tr style="text-align: center;" >
    <th style="text-align: center;" rowspan="4">SV32</th>
	<td rowspan="2">1</td>
    <td >PPN[1](9 downto 0)</td>
  </tr>
  <tr style="text-align: center;" >
    <td >leaf PTE read(29 downto 20)</td>
  </tr>
   <tr style="text-align: center;" >
	<td rowspan="2">0</td>
    <td >PPN[1](9 downto 0) # PPN[0]</td>
  </tr>
  <tr style="text-align: center;" >
    <td >leaf PTE read(29 downto 20) # leaf PTE read(19 downto 10)</td>
  </tr>
  <tr style="text-align: center;" >
    <th style="text-align: center;" rowspan="6">SV39</th>
    <td rowspan="2">2</td>
    <td >PPN[2]</td>
  </tr>
  <tr style="text-align: center;" >
    <td >leaf PTE read(53 downto 28)</td>
  </tr>
  <tr style="text-align: center;" >
    <td rowspan="2">1</td>
    <td >PPN[2] # PPN[1]</td>
  </tr>
  <tr style="text-align: center;" >
    <td >leaf PTE read(53 downto 28) # leaf PTE read(27 downto 19)</td>
  </tr>
  <tr style="text-align: center;" >
    <td rowspan="2">0</td>
    <td >PPN[2] # PPN[1] # PPN[0]</td>
  </tr>
  <tr style="text-align: center;" >
    <td >leaf PTE read(53 downto 28) # leaf PTE read(27 downto 19) # leaf PTE read(18 downto 10)</td>
  </tr>
</table>

Moreover, as `physicalAddress` is resized, we remove the `(spec.physicalWidth - physicalWidth)` MSBs. Because the total width of the physical address must be `physicalWidth`.

For example, with SV39, `spec.physicalWidth = 56`, MMU `physicalWidth` parameter = 32, and `levelId = 0` :
`spec.physicalWidth - physicalWidth` = 56 - 32 = 24
`physicalAddress` = leaf PTE read(53 downto 10).resized = leaf PTE read(29 downto 10)

Thus, the physical address formed by `physicalAddress` and page offset, will have a width of 32 bits (20 + 12).


##### pageFault

`load.exception = True` in one of the following cases :
- `flags.V = False`, as it indicates that PTE is not valid
- `flags.R = False & flags.W = True`, as this permission bits configuration is reserved
- `rsp.fault = True`, as it indicates that the cache sends a fault response
- `leaf = False` and `flags.D = True | flags.A = True | flags.U = True`, as for non-leaf PTEs, the D,A and U bits are reserved.

`load.levelException(levelId) = True`, if a megapage is not virtually and physically aligned, for example :
- In SV39, with doneLogic called at `levelId = 1`, we have a megapage of 2 MiB, the 21 LSBs of physical address must be equal to zero, so PPN\[0\] must be equal to zero in leaf PTE read from memory.

`load.flags.A = False`, accessed flag must be set by the software.

##### accessFault

We have an access fault, if there is a page fault or the physical address is greater than `physicalWidth`.

***
## Code

```scala
def doneLogic() : Unit = {  
  for(storage <- storages){  
    val storageLevelId = storage.self.p.levels.filter(_.id <= levelId).map(_.id).max  
    val storageLevel = storage.sl.find(_.slp.id == storageLevelId).get  
    val specLevel = storageLevel.level  
  
    val sel = (portOhReg.asBools, ports).zipped.toList.filter(_._2.storage == storage).map(_._1).orR  
    storageLevel.write.mask                 := UIntToOh(storageLevel.allocId).andMask(sel)  
    storageLevel.write.address              := virtual(storageLevel.lineRange)  
    storageLevel.write.data.valid           := True  
    storageLevel.write.data.pageFault       := load.exception || load.levelException(levelId) || !load.flags.A  
    storageLevel.write.data.accessFault     := !storageLevel.write.data.pageFault && load.levelToPhysicalAddress(levelId).drop(physicalWidth) =/= 0  
    storageLevel.write.data.virtualAddress  := virtual(specLevel.virtualOffset + log2Up(storageLevel.slp.depth), widthOf(storageLevel.write.data.virtualAddress) bits)  
    storageLevel.write.data.physicalAddress := (load.levelToPhysicalAddress(levelId) >> specLevel.virtualOffset).resized  
    storageLevel.write.data.allowRead       := load.flags.R  
    storageLevel.write.data.allowWrite      := load.flags.W && load.flags.D  
    storageLevel.write.data.allowExecute    := load.flags.X  
    storageLevel.write.data.allowUser       := load.flags.U  
    storageLevel.allocId.increment()  
  }  
  
  goto(IDLE)  
}

```
