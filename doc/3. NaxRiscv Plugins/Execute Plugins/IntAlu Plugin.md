`IntAluPlugin` is the plugin that executes :
- addition operations
- subtraction operations
- logical operations (xor, or, and)
- load immediate and auipc instructions
- "set if less than" instructions

[code](https://github.com/SpinalHDL/NaxRiscv/blob/main/src/main/scala/naxriscv/execute/IntAluPlugin.scala)

___

#### Parameters

**`euId`**
Like all other Execute plugins, we need to associate it with an `ExecuteUnitBase`, so we give it its name with the `euId` parameter.

**`staticLatency`**
By default, this plugin has static latency operations.

**`aluStage`**
The execute stage where the instruction is executed, by default `aluStage = 0`.

**`writebackAt`**
The execution phase, where the result of the operation is written back into the register, by default `writebackAt = 0`.

___

#### Setup

In this area, we use the `add` function from `ExecutionUnitElementSimple`.
This function : 
- sets the link with the `ExecuteUnitBase` plugin, by adding the microOp supported, their resources, etc
- sets its stageables(`ALU_CTRL`, `ALU_BITWISE_CTRL`) in function of microOp added
- defines the `SrcPlugin` keys to know which operands to choose and which type of operation is

> **Reminder :** Stageables are variables that are transmitted through a pipeline, so a stageable filled in at stage 0 can be read two cycle later at stage 2.

**Micro Operations supported**
- `ADD, ADDI`
- `SUB`
- `SLT, SLTU, SLTI, SLTIU`
- `XOR, XORI`
- `OR, ORI`
- `AND, ANDI`
- `LUI`
- `AUIPC`
if `XLEN = 64`
- `ADDW, SUBW, ADDIW`

**IntAlu stageables**

- `ALU_CTRL` : used in logic area to select the correct result, as the plugin performs all operations at the same time.
	The 3 possible values are :
	- `AluCtrlEnum.ADD_SUB` : select result for operations of type `ADD`, `SUB`, `LUI`, `AUIPC`
	- `AluCtrlEnum.SLT_SLTU` : result for operations of type `SLT`
	- `AluCtrlEnum.BITWISE` : result for logical operations (`AND, OR, XOR`)

- `ALU_BITWISE_CTRL` : used in logic area to select the correct result between logical operations.
	The 3 possible values are :
	- `AluBitwiseCtrlEnum.AND` : select `AND` result
	- `AluBitwiseCtrlEnum.OR` : select `OR` result
	- `AluBitwiseCtrlEnum.XOR` : select `XOR` result

These stageable values are defined during the decode stage of the execute pipeline. We know which value to take thanks to `DecodeList` in the `add` function.

Example for `AND` instruction :
`add(Rvi.AND , List(SRC1.RF, SRC2.RF), DecodeList(ALU_CTRL -> ace.BITWISE , ALU_BITWISE_CTRL -> abce.AND ))`

`ALU_CTRL = AluCtrlEnum.BITWISE` 
`ALU_BITWISE_CTRL = AluBitwiseCtrlEnum.AND`

**Src keys**

- `SRC1` : type of operand 1
	The 2 possible values are :
	- `RF` : operand 1 is a register value
	- `U` : operand 1 is an immediate of type U

- `SRC2` : type of operand 2
	The 4 possible values are :
	- `RF` : operand 2 is a register value
	- `S` : operand 2 is an immediate of type S
	- `I` : operand 2 is an immediate of type I
	- `PC` : operand 2 is the PC value

- `Op` : type of operation, used by `SrcPlugin` to find out what operation it should perform, and to define some intern stageables.
	In fact, some operations such as `ADD` or `SUB` are calculated in the `SrcPlugin`, and `IntAluPlugin` takes the result directly.
	The 5 possible values are :
	- `ADD` : operation of type `ADD` (`ADD, ADDI, AUIPC, ADDW, ADDIW`)
	- `SUB` : operation of type `SUB` (`SUB, SUBW`)
	- `SRC1` : operation with only operand 1 (`LUI`)
	- `LESS` : operation of type `SLT` (`SLT, SLTI`)
	- `LESS_U` : operation of type `SLTU` (`SLTU, SLTIU`)
	As logical operations (`XOR, OR, AND`) are calculated in `IntAluPlugin`, we do not specify these keys for these operations.
	
	These stageable values are defined thanks to `Srckeys` list in the `add` function.

___

#### Logic

Thanks to the setup area, we've set up our plugin properly and it will run using the stageables we've defined.

1. `SrcPlugin` sets the values of `ss.SRC1` and `ss.SRC2` according to the instruction decoded using `Srckeys`. The plugin does this 1 stage before the first pipeline execution stage, if `earlySrc = True`, which is the default.
2. `SrcPlugin` calculates `ss.ADD_SUB` and `ss.LESS` according to the instruction decoded. In the same time, `IntAluPlugin` calculates the logical operation. Finally, the final result is stored in `ALU_RESULT` according to `ALU_BITWISE_CTRL` and `ALU_CTRL` values. These operations are performed by default during execution stage(0).
3. `IntAluPlugin` sends the result to `IntFormatPlugin` which sends it to `ExecuteUnitBase`, to be stored in the registry file. This is also done in stage(0) by default.

<svg version="1.1" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1115.3130136358536 478" width="1115.3130136358536" height="478">
  <!-- svg-source:excalidraw -->
  
  <defs>
    <style class="style-fonts">
      @font-face {
        font-family: "Virgil";
        src: url("https://excalidraw.com/Virgil.woff2");
      }
      @font-face {
        font-family: "Cascadia";
        src: url("https://excalidraw.com/Cascadia.woff2");
      }
    </style>
  </defs>
  <rect x="0" y="0" width="1115.3130136358536" height="478" fill="#ffffff"></rect><g stroke-linecap="round"><g transform="translate(433 92) rotate(0 0 89.5)"><path d="M0 0 C0 53.51, 0 107.03, 0 179 M0 0 C0 71.44, 0 142.88, 0 179" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(475 113) rotate(0 0 70)"><path d="M0 0 C0 52.28, 0 104.56, 0 140 M0 0 C0 50.57, 0 101.14, 0 140" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(433 92) rotate(0 21 10.5)"><path d="M0 0 C10.15 5.08, 20.3 10.15, 42 21 M0 0 C15.8 7.9, 31.6 15.8, 42 21" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(433 271) rotate(0 20.5 -8.5)"><path d="M0 0 C15.63 -6.48, 31.26 -12.96, 41 -17 M0 0 C12.01 -4.98, 24.02 -9.96, 41 -17" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(663 212.5) rotate(0 0 89.5)"><path d="M0 0 C0 49.27, 0 98.54, 0 179 M0 0 C0 67.59, 0 135.18, 0 179" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(748 233.5) rotate(0 0 70)"><path d="M0 0 C0 40.91, 0 81.81, 0 140 M0 0 C0 52.14, 0 104.28, 0 140" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(663 212.5) rotate(0 42.5 10)"><path d="M0 0 C22.38 5.27, 44.76 10.53, 85 20 M0 0 C25.48 5.99, 50.95 11.99, 85 20" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(663 391.5) rotate(0 42.5 -9)"><path d="M0 0 C27.28 -5.78, 54.55 -11.55, 85 -18 M0 0 C28.56 -6.05, 57.12 -12.1, 85 -18" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(430 184) rotate(0 -100.5 0)"><path d="M0 0 C-63.29 0, -126.57 0, -201 0 M0 0 C-78.69 0, -157.38 0, -201 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(430 127) rotate(0 -100.5 0)"><path d="M0 0 C-47.14 0, -94.29 0, -201 0 M0 0 C-79.02 0, -158.04 0, -201 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(430 240) rotate(0 -100.5 0)"><path d="M0 0 C-72.46 0, -144.92 0, -201 0 M0 0 C-69.8 0, -139.6 0, -201 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(364 42) rotate(0 46.5 30)"><path d="M0 0 C36.14 0, 72.28 0, 93 0 M0 0 C33.25 0, 66.49 0, 93 0 M93 0 C93 18.74, 93 37.47, 93 60 M93 0 C93 13.7, 93 27.4, 93 60" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(662 247) rotate(0 -92.5 -30)"><path d="M0 0 C-46.49 0, -92.98 0, -134 0 M0 0 C-35.87 0, -71.73 0, -134 0 M-134 0 C-134 -22.38, -134 -44.76, -134 -60 M-134 0 C-134 -22.77, -134 -45.54, -134 -60 M-134 -60 C-144.89 -60, -155.78 -60, -185 -60 M-134 -60 C-150.46 -60, -166.92 -60, -185 -60" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(619.5 164) rotate(0 46.5 30)"><path d="M0 0 C19.28 0, 38.56 0, 93 0 M0 0 C21.2 0, 42.4 0, 93 0 M93 0 C93 23.63, 93 47.26, 93 60 M93 0 C93 23.12, 93 46.25, 93 60" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g transform="translate(238 96) rotate(0 91.5 11.5)"><text x="91.5" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">ss.SRC1 &amp; ss.SRC2</text></g><g transform="translate(246 152.5) rotate(0 87.5 11.5)"><text x="87.5" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">ss.SRC1 | ss.SRC2</text></g><g transform="translate(242 208.5) rotate(0 89.5 11.5)"><text x="89.5" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">ss.SRC1 ^ ss.SRC2</text></g><g transform="translate(257.5 10) rotate(0 98.5 11.5)"><text x="98.5" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">ALU_BITWISE_CTRL</text></g><g transform="translate(436.5 118.5) rotate(0 17.5 9)"><text x="17.5" y="14" font-family="Helvetica, Segoe UI Emoji" font-size="16px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">AND</text></g><g transform="translate(440.5 173) rotate(0 12.5 9)"><text x="12.5" y="14" font-family="Helvetica, Segoe UI Emoji" font-size="16px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">OR</text></g><g transform="translate(436 230) rotate(0 18 9)"><text x="18" y="14" font-family="Helvetica, Segoe UI Emoji" font-size="16px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">XOR</text></g><g transform="translate(587.5 214.5) rotate(0 31 11.5)"><text x="31" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">bitwise</text></g><g stroke-linecap="round"><g transform="translate(660 303) rotate(0 -174.5 0)"><path d="M0 0 C-101.24 0, -202.48 0, -349 0 M0 0 C-97.47 0, -194.93 0, -349 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(660 359) rotate(0 -175 0)"><path d="M0 0 C-83.95 0, -167.91 0, -350 0 M0 0 C-76.54 0, -153.08 0, -350 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g transform="translate(353.5 328) rotate(0 148 11.5)"><text x="148" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">S(U(ss.LESS, Global.XLEN bits))</text></g><g transform="translate(528.5 273) rotate(0 60.5 11.5)"><text x="60.5" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">ss.ADD_SUB</text></g><g transform="translate(670.5 237.5) rotate(0 33.5 9)"><text x="33.5" y="14" font-family="Helvetica, Segoe UI Emoji" font-size="16px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">BITWISE</text></g><g transform="translate(667 351) rotate(0 39 9)"><text x="39" y="14" font-family="Helvetica, Segoe UI Emoji" font-size="16px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">SLT_SLTU</text></g><g transform="translate(667.5 294) rotate(0 38.5 9)"><text x="38.5" y="14" font-family="Helvetica, Segoe UI Emoji" font-size="16px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">ADD_SUB</text></g><g transform="translate(604.5 132.5) rotate(0 51.5 11.5)"><text x="51.5" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">ALU_CTRL</text></g><g transform="translate(755.8959111569475 272.02081776861047) rotate(0 64 11.5)"><text x="64" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">ALU_RESULT</text></g><g stroke-linecap="round"><g transform="translate(750 305) rotate(0 126.42694077978103 0)"><path d="M0 0 C80.19 0, 160.37 0, 252.85 0 M0 0 C60.12 0, 120.24 0, 252.85 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g transform="translate(895.666915702232 272.02081776861047) rotate(0 51 11.5)"><text x="51" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">wb.payload</text></g><g stroke-linecap="round" transform="translate(1004.3130136358536 258.9791822313896) rotate(0 50.5 45)"><path d="M0 0 C22.57 0, 45.14 0, 101 0 M0 0 C31.97 0, 63.94 0, 101 0 M101 0 C101 34.95, 101 69.9, 101 90 M101 0 C101 20.58, 101 41.16, 101 90 M101 90 C65.19 90, 29.39 90, 0 90 M101 90 C67.97 90, 34.94 90, 0 90 M0 90 C0 68.72, 0 47.43, 0 0 M0 90 C0 65.6, 0 41.2, 0 0" stroke="#000000" stroke-width="4" fill="none"></path></g><g transform="translate(1011.3130136358536 280.9791822313896) rotate(0 43.5 23)"><text x="43.5" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">IntFormat</text><text x="43.5" y="41" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">Plugin</text></g><g stroke-linecap="round" transform="translate(236 420) rotate(0 434.5 24)"><path d="M12 0 M12 0 C336.3 0, 660.6 0, 857 0 M12 0 C221.69 0, 431.39 0, 857 0 M857 0 C865 0, 869 4, 869 12 M857 0 C865 0, 869 4, 869 12 M869 12 C869 19.06, 869 26.12, 869 36 M869 12 C869 17.41, 869 22.82, 869 36 M869 36 C869 44, 865 48, 857 48 M869 36 C869 44, 865 48, 857 48 M857 48 C542.87 48, 228.74 48, 12 48 M857 48 C591.36 48, 325.73 48, 12 48 M12 48 C4 48, 0 44, 0 36 M12 48 C4 48, 0 44, 0 36 M0 36 C0 28.89, 0 21.77, 0 12 M0 36 C0 27.95, 0 19.89, 0 12 M0 12 C0 4, 4 0, 12 0 M0 12 C0 4, 4 0, 12 0" stroke="#000000" stroke-width="4" fill="none"></path></g><g stroke-linecap="round" transform="translate(10 420) rotate(0 111.5 24)"><path d="M12 0 M12 0 C55.91 0, 99.83 0, 211 0 M12 0 C57.21 0, 102.41 0, 211 0 M211 0 C219 0, 223 4, 223 12 M211 0 C219 0, 223 4, 223 12 M223 12 C223 17.92, 223 23.84, 223 36 M223 12 C223 17.93, 223 23.86, 223 36 M223 36 C223 44, 219 48, 211 48 M223 36 C223 44, 219 48, 211 48 M211 48 C144.28 48, 77.57 48, 12 48 M211 48 C158.67 48, 106.34 48, 12 48 M12 48 C4 48, 0 44, 0 36 M12 48 C4 48, 0 44, 0 36 M0 36 C0 28.01, 0 20.01, 0 12 M0 36 C0 26.49, 0 16.98, 0 12 M0 12 C0 4, 4 0, 12 0 M0 12 C0 4, 4 0, 12 0" stroke="#000000" stroke-width="4" fill="none"></path></g><g transform="translate(72 432.5) rotate(0 49.5 11.5)"><text x="49.5" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">fetch stage</text></g><g stroke-linecap="round"><g transform="translate(12.37144998176177 12) rotate(0 0 186)"><path d="M0 0 C0 62, 0 310, 0 372 M0 0 C0 62, 0 310, 0 372" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(310 279) rotate(0 0 51)"><path d="M0 0 C0 17, 0 85, 0 102 M0 0 C0 17, 0 85, 0 102" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(12.675827471917728 384) rotate(0 148.66208626404114 0)"><path d="M0 0 C49.55 0, 247.77 0, 297.32 0 M0 0 C49.55 0, 247.77 0, 297.32 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(11.78564095033721 11) rotate(0 32.607179524831395 0)"><path d="M0 0 C10.87 0, 54.35 0, 65.21 0 M0 0 C10.87 0, 54.35 0, 65.21 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(77 279) rotate(0 116.375 0)"><path d="M0 0 C38.79 0, 193.96 0, 232.75 0 M0 0 C38.79 0, 193.96 0, 232.75 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(77 12) rotate(0 0 132)"><path d="M0 0 C0 44, 0 220, 0 264 M0 0 C0 44, 0 220, 0 264" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g transform="translate(115.24525257872165 318.2218951902719) rotate(0 46 11.5)"><text x="46" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">Src Plugin</text></g><g stroke-linecap="round"><g transform="translate(190.40678730878574 153.2058193234052) rotate(0 -55.70339365439287 0)"><path d="M0 0 C-41.1 0, -82.19 0, -111.41 0 M0 0 C-29.71 0, -59.42 0, -111.41 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(190.06001643880546 217.3087289851078) rotate(0 -56.03000821940273 0)"><path d="M0 0 C-36.7 0, -73.4 0, -112.06 0 M0 0 C-26 0, -52.01 0, -112.06 0" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g transform="translate(92.10158305448124 121.42825333504192) rotate(0 40 11.5)"><text x="40" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">ss.SRC1</text></g><g stroke-linecap="round" transform="translate(190 93) rotate(0 18.5 84.97122373723454)"><path d="M0 0 L37 0 L37 169.94 L0 169.94" stroke="none" stroke-width="0" fill="#ffffff"></path><path d="M0 0 C11.53 0, 23.06 0, 37 0 M0 0 C11.2 0, 22.4 0, 37 0 M37 0 C37 53.67, 37 107.33, 37 169.94 M37 0 C37 41.66, 37 83.31, 37 169.94 M37 169.94 C24.39 169.94, 11.77 169.94, 0 169.94 M37 169.94 C25.97 169.94, 14.95 169.94, 0 169.94 M0 169.94 C0 119.99, 0 70.04, 0 0 M0 169.94 C0 116.97, 0 63.99, 0 0" stroke="#000000" stroke-width="4" fill="none"></path></g><g stroke-linecap="round"><g transform="translate(189.01970382886464 153.09856917434635) rotate(0 18.55224154394557 -13.003907624260933)"><path d="M0 0 C9.63 -6.75, 19.26 -13.5, 37.1 -26.01 M0 0 C7.99 -5.6, 15.97 -11.2, 37.1 -26.01" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(189.7132455688252 153.09856917434635) rotate(0 18.5522415439456 15.778074584103265)"><path d="M0 0 C13.03 11.08, 26.06 22.16, 37.1 31.56 M0 0 C10.58 9, 21.16 18, 37.1 31.56" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(190.06001643880546 153.09856917434635) rotate(0 17.68531436899488 43.69312961751673)"><path d="M0 0 C10.62 26.23, 21.24 52.46, 35.37 87.39 M0 0 C12.4 30.64, 24.81 61.29, 35.37 87.39" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(189.36647469884485 216.90440925071997) rotate(0 18.552241543945627 -15.604689149113113)"><path d="M0 0 C7.72 -6.49, 15.44 -12.99, 37.1 -31.21 M0 0 C14.33 -12.05, 28.66 -24.11, 37.1 -31.21" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(190.06001643880546 217.59795099068057) rotate(0 17.85869980398502 11.443438709349621)"><path d="M0 0 C8.3 5.32, 16.6 10.64, 35.72 22.89 M0 0 C9.24 5.92, 18.47 11.84, 35.72 22.89" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g stroke-linecap="round"><g transform="translate(189.36647469884485 217.25118012070027) rotate(0 18.205470673965323 -44.38667135747731)"><path d="M0 0 C12.52 -30.52, 25.03 -61.04, 36.41 -88.77 M0 0 C12.31 -30.01, 24.61 -60.01, 36.41 -88.77" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask><g transform="translate(92.89924463368231 185.8620967147499) rotate(0 40 11.5)"><text x="40" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">ss.SRC2</text></g><g transform="translate(498.4362656202085 431.8305083313313) rotate(0 63 11.5)"><text x="63" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">execute stage</text></g><g transform="translate(932.4921900906481 431.4689089339777) rotate(0 70 11.5)"><text x="70" y="18" font-family="Helvetica, Segoe UI Emoji" font-size="20px" fill="#000000" text-anchor="middle" style="white-space: pre;" direction="ltr">writeback stage</text></g><g stroke-linecap="round"><g transform="translate(898.2139255647032 421.07970554327244) rotate(0 0 23.172077127294102)"><path d="M0 0 C0 17.59, 0 35.17, 0 46.34 M0 0 C0 11.92, 0 23.85, 0 46.34" stroke="#000000" stroke-width="4" fill="none"></path></g></g><mask></mask></svg>
  
  
  
  