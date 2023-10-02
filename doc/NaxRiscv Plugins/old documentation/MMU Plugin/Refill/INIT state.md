> **State that init variables using address**

***
## Description

It initializes `load.address` with `satp.ppn` and a part of virtual address.

<table>
  <tr >
    <th>SATP Mode</th>
    <th style="text-align: center;" colspan="10">load.address</th>
  </tr>
 <tr style="text-align: center;" >
	<th style="text-align: center;" rowspan="2">SV32</th>
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
    <td colspan="3">satp.ppn</td>
    <td colspan="3">portsAddress(31 downto 22)</td>
    <td colspan="3">00</td>
  </tr>
   <tr style="text-align: center;" >
	<th style="text-align: center;" rowspan="2">SV39</th>
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
    <td colspan="3">satp.ppn</td>
    <td colspan="3">portsAddress(38 downto 30)</td>
    <td colspan="3">000</td>
  </tr>
</table>

Then if it's SV32, it goes to CMD(1) state, else if it's SV39, it goes to CMD(2) state.

***
## Code

```scala
INIT whenIsActive{  
  virtual := portsAddress  
  load.address := (satp.ppn @@ spec.levels.last.vpn(portsAddress) @@ U(0, log2Up(spec.entryBytes) bits)).resized //It relax timings to do it there instead than in IDLE  
  goto(CMD(spec.levels.size - 1))  
}

```






