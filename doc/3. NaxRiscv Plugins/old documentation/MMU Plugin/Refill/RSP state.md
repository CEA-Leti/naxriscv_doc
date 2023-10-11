> **State that receives load response from cache memory, and process it**

There is a RSP state for each TLB level, so for :
-  SV32 : 2 levels --> 2 RSP states
-  SV39 : 3 levels --> 3 RSP states

***
## Description

When it receives a `load.rsp.valid`, it checks `load.rsp.redo` :
- If `load.rsp.redo = True`, it returns to the previous CMD state.
- Else, it checks `levelId` :
	- If `levelId = 0`, it goes to `doneLogic`
	- Else, it checks `load.leaf` (`flags.R | flags.X`) and `load.exception` :
		- If `load.leaf = True` or `load.exception = True`, it goes to `doneLogic`
		- Else, it updates `load.address` wtih page-table entry(PTE) received and a part of virtual address. Then it goes to the next CMD state.


#### Load command address update

<table>
  <tr >
    <th>SATP Mode</th>
    <th>levelId</th>
    <th style="text-align: center;" colspan="10">load.address</th>
  </tr>
 <tr style="text-align: center;" >
	<th style="text-align: center;" rowspan="3">SV32</th>
	<td ></td>
    <td >31</td>
    <td >20 bits</td>
    <td >12</td>
    <td >11</td>
    <td >10 bits</td>
    <td >2</td>
    <td >1</td>
    <td >2 bits</td>
    <td >0</td>
  </tr>
  <tr style="text-align: center;" >
	<td >1</td>
    <td colspan="3">satp.ppn</td>
    <td colspan="3">virtual(31 downto 22)</td>
    <td colspan="3">00</td>
  </tr>
   <tr style="text-align: center;" >
	<td >0</td>
    <td colspan="3">read(29 downto 10)</td>
    <td colspan="3">virtual(21 downto 12)</td>
    <td colspan="3">00</td>
  </tr>
  <tr style="text-align: center;" >
	<th style="text-align: center;" rowspan="4">SV39</th>
	<td ></td>
    <td >63</td>
    <td >44 bits</td>
    <td >12</td>
    <td >11</td>
    <td >9 bits</td>
    <td >3</td>
    <td >2</td>
    <td >3 bits</td>
    <td >0</td>
  </tr>
  <tr style="text-align: center;" >
    <td >2</td>
    <td colspan="3">satp.ppn</td>
    <td colspan="3">virtual(38 downto 30)</td>
    <td colspan="3">000</td>
  </tr>
  <tr style="text-align: center;" >
    <td >1</td>
    <td colspan="3">read(53 downto 10)</td>
    <td colspan="3">virtual(29 downto 21)</td>
    <td colspan="3">000</td>
  </tr>
  <tr style="text-align: center;" >
    <td >0</td>
    <td colspan="3">read(53 downto 10)</td>
    <td colspan="3">virtual(20 downto 12)</td>
    <td colspan="3">000</td>
  </tr>
</table>

#### Exception cases

`load.exception = True` in one of the following cases :
- `flags.V = False`, as it indicates that PTE is not valid
- `flags.R = False & flags.W = True`, as this permission bits configuration is reserved
- `rsp.fault = True`, as it indicates that the cache sends a fault response
- `leaf = False` and `flags.D = True | flags.A = True | flags.U = True`, as for non-leaf PTEs, the D,A and U bits are reserved.

#### Reminder

> **Source :**  The RISC-V Instruction Set Manual, Volume 2 Privileged Architecture

<table>
  <tr >
    <th style="text-align: center;" colspan="17">SV32 Page Table Entry</th>
  </tr>
 <tr style="text-align: center;" >
    <td >31</td>
    <td >12 bits</td>
    <td >20</td>
    <td >19</td>
    <td >10 bits</td>
    <td >10</td>
    <td >9</td>
    <td >2 bits</td>
    <td >8</td>
    <td >7</td>
    <td >6</td>
    <td >5</td>
    <td >4</td>
    <td >3</td>
    <td >2</td>
    <td >1</td>
    <td >0</td>
  </tr>
  <tr style="text-align: center;" >
    <td colspan="3">PPN[1]</td>
    <td colspan="3">PPN[0]</td>
    <td colspan="3">RSW</td>
    <td>D</td>
    <td>A</td>
    <td>G</td>
    <td>U</td>
    <td>X</td>
    <td>W</td>
    <td>R</td>
    <td>V</td>
  </tr>
</table>

In NaxRiscv, SV32 PTE has only 30 bits, because PPN\[1\] must have 10 bits, to avoid 34 bits physical address.

<table>
  <tr >
    <th style="text-align: center;" colspan="27">SV39 Page Table Entry</th>
  </tr>
 <tr style="text-align: center;" >
    <td >63</td>
    <td >62</td>
    <td >2 bits</td>
    <td >61</td>
    <td >60</td>
    <td >7 bits</td>
    <td >54</td>
    <td >53</td>
    <td >26 bits</td>
    <td >28</td>
    <td >27</td>
    <td >9 bits</td>
    <td >19</td>
    <td >18</td>
    <td >9 bits</td>
    <td >10</td>
    <td >9</td>
    <td >2 bits</td>
    <td >8</td>
    <td >7</td>
    <td >6</td>
    <td >5</td>
    <td >4</td>
    <td >3</td>
    <td >2</td>
    <td >1</td>
    <td >0</td>
  </tr>
  <tr style="text-align: center;" >
    <td >N</td>
    <td colspan="3">PBMT</td>
    <td colspan="3">Reserved</td>
    <td colspan="3">PPN[2]</td>
    <td colspan="3">PPN[1]</td>
    <td colspan="3">PPN[0]</td>
    <td colspan="3">RSW</td>
    <td>D</td>
    <td>A</td>
    <td>G</td>
    <td>U</td>
    <td>X</td>
    <td>W</td>
    <td>R</td>
    <td>V</td>
  </tr>
</table>

PTE flag | Name | Description
:------: | :------: | :------:
D | Dirty | Indicates the virtual page has been written since the last time the D bit was cleared
A | Accessed | Indicates the virtual page has been read, written, or fetched from since the last time the A bit was cleared
G | Global | Indicates a global mapping, global mappings are those that exist in all address spaces
U | User | Indicates whether the page is accessible to user mode
X | Executable | Indicates whether the page is executable
W | Writeable | Indicates whether the page is writeable
R | Readable | Indicates whether the page is readable
V | Valid | Indicates whether the PTE is valid
RSW | | Reserved for use by supersivor software, the implementation shall ignore it
N | NAPOT Translation Contiguity | Reserved for use by the Svnapot extension, if Svnapot not implemented must be zeroed by software
PBMT | Page-Based Memory Types | Reserved for use by the Svpbmt extension, if Svpbmt not implemented must be zeroed by software



***
## Code

```scala
RSP(levelId) whenIsActive{  
  if(levelId == 0) load.exception setWhen(!load.leaf)  
  when(load.rsp.valid){  
    when(load.rsp.redo){  
      goto(CMD(levelId))  
    } otherwise {  
      levelId match {  
        case 0 => doneLogic  
        case _ => {  
          when(load.leaf || load.exception) {  
            doneLogic  
          } otherwise {  
            val targetLevelId = levelId - 1  
            val targetLevel = spec.levels(targetLevelId)  
            load.address := load.nextLevelBase  
            load.address(log2Up(spec.entryBytes), targetLevel.physicalWidth bits) := targetLevel.vpn(virtual)  
            goto(CMD(targetLevelId))  
          }  
        }  
      }  
    }  
  }  
}
```