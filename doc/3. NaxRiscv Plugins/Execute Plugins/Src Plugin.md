`SrcPlugin` is the plugin which defines the operand values for the following plugins `IntAluPlugin, BranchPlugin, ShiftPlugin`.
The operand values come from the register file or from the instructions themselves for the immediate.
It also calculates addition, subtraction, and"set if less than" operations, and sends the result to `IntAluPlugin`

[code](https://github.com/SpinalHDL/NaxRiscv/blob/main/src/main/scala/naxriscv/execute/SrcPlugin.scala)

___

#### Parameters

**`euId`**
Like all other Execute plugins, we need to associate it with an `ExecuteUnitBase`, so we give it its name with the `euId` parameter.

**`earlySrc`**
Used to know when the operand values must be set, during the execution stage(0), or 1 step before, if `earlySrc = True`. So when does it read the registry file.

___

#### Logic

With the `specify` function used by `IntAluPlugin, BranchPlugin, ShiftPlugin` plugins, fill the following arrays :
- `spec` : array linking microOps to their `srcKeys`

- `keys` : list of all `srcKeys` used by plugins connected to `SrcPlugin`. As this is a `LinkedHashSet`, there is no duplication of elements, so if plugins use the same `srcKeys` several times, it will only appear once in the list.

- `opKeys` : list of all `srcKeys` of type `OpKeys` (`ADD, SUB, SRC1, LESS, LESS_U`)
- `src1Keys` : list of all `srcKeys` of type `Src1Keys` (`RF, U`)
- `src2Keys` : list of all `srcKeys` of type `Src2Keys` (`RF, I, S, PC`)

- `src1ToEnum` : array linking `srcKeys` of type `Src1Keys` to a number. Because we've used `keys`, we don't have duplicate elements, and so a key is associated with a single number.This table is used to define the `SRC1_CTRL` value, which will be used in the `ss.SRC1` multiplexer. The number gives the value that `ss.SRC1` should take, because it gives the type of operand 1.
- `src2ToEnum` : Same as `src1ToEnum` but for `srcKeys` of type `Src2Keys`

Then, thanks to all these arrays, we can use the `addDecoding` function to define the stageable values of `SrcPlugin` according to the decoded instruction in `ExecuteUnitBase`.
The stageables defined are :
- `ss.REVERT = True` if the second operand must be inverted
- `ss.ZERO = True` if the second operand must be set to zero
- `ss.UNSIGNED = True`, if the instruction it's an unsigned compare 
- `SRC1_CTRL` : Multiplexer select signal for `ss.SRC1`
- `SRC2_CTRL` : Multiplexer select signal for `ss.SRC2`

Once all the stageables have been defined, we can carry out the necessary operations depending on the instruction being processed.

> [!NOTE] 
> Stageables are variables that are transmitted through a pipeline, so a stageable filled in at stage 0 can be read two cycle later at stage 2.

Firstly, depending on `SRC1_CTRL` and `SRC2_CTRL` which have their value set during the decode stage of `ExecuteUnitBase`, `ss.SRC1` and `ss.SRC2` take the needed value (register file value, if the instruction has a register as an operand, or immediate value if the instruction has an immediate as an operand, etc).

These two stageables (`ss.SRC1` and `ss.SRC2`) are then passed to all the plugins that need them via the execute pipeline.

Secondly, depending on the others stageables, we can calculate the following stageables :
- `ss.ADD_SUB` : stores the result for `Op.ADD`, `Op.SUB` and `Op.SRC1` operations (`ADD`, `ADDI`, `ADDW`, `ADDIW`, `SUB`, `SUBW`, `AUIPC` or `LUI` instructions)

- `ss.LESS` : stores the result for `Op.LESS` and `Op.LESS_U` operations (`SLT`, `SLTU`, `SLTI` or `SLTIU` instructions)

These two stageables are then passed to `IntAluPlugin` via the execute pipeline.
In this way, only the `XOR`, `OR`, `AND`, `XORI`, `ORI`, `ANDI` instructions are calculated in `IntAluPlugin`.