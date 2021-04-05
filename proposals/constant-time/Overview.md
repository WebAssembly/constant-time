# Constant-Time Integer Operations

## Motivation
As WebAssembly grows in popularity as both an in-browser language and as a sandboxing solution, we need some way to run cryptography code safely in Wasm. 

For instance, consider the following issue on the `wasi-crypto` repo ([link](https://github.com/WebAssembly/wasi-crypto/issues/21)):

> WebAssembly doesn't provide any guarantees against resistance to side-channel attacks. The diversity of runtimes makes it impossible to verify that code originally written to be constant-time will actually be constant-time. There isn't any ways to ensure that secrets are cleared from memory either.

> Beyond CT-Wasm, implementers may have expectations, such as the fact that conditional moves, for which WebAssembly has an opcode for, is free of side channels. But this isn't mandated by the specification yet.

In addition, ARM is rolling out a new `PSTATE.DIT` flag that provides microarchitectural guarantees of certain constant-time operations, and we would like some way for security-critical Wasm programs to take advantage of this new feature.

## Goals
With this proposal, we would like to achieve the following:
1. Allow timing-sensitive code to be run securely in browsers and sandboxes.
1. Make it straightforward for runtimes and JIT compilers to support constant-time integer operations.
1. Minimize performance overhead, both as a result of enabling processor features like `PSTATE.DIT` and of missed opportunities for optimization.

## Design
We introduce new .ct instructions that, assuming processor and compiler support, are guaranteed to exhibit runtime independent of the values supplied by their inputs. 

Some exceptions are memory load and store instructions, which exhibit runtime independent of the values being loaded and stored, not of the addresses. 

In addition, `set.ct`, `store.ct`, and `fill.ct` (`fill.ct` not included here yet) operations are guaranteed not to be eliminated in a dead store elimination pass. This ensures that secrets are scrubbed from memory when the user intends.

## `DIT` Support
A hypothetical JIT compiler that supports both this proposal and ARMv8.4+ would, upon finding a function that contains a `.ct` instruction, insert prologue and epilogue procedures into the function that ensures the `DIT` flag is set during the execution of the function, and restores the original value of the `DIT` flag during the epilogue. Then it would ensure that every instruction's `.ct` flag is preserved during each lowering phase, and finally, during the final lowering phase, emit only instructions covered by `DIT` for any IR instruction marked `.ct`.

## Future Work
We would also like to support SIMD instructions in the future, and examine how other architectures' future constant-time guarantees can be leveraged by Wasm. We would also like to add some feature-detection bits so that especially security-critical code can refuse to run when the runtime or the architecture is unable to provide the necessary timing guarantees.

## Index of Additional Instructions
To make it easy to port existing code, we propose adding a new single-byte prefix (shown as CT in the below table) and assigning opcodes that match the preexisting ones with the prefix attached. 

|Instruction              |Opcode |
|-------------------------|-------|
|(reserved)               |CT 0x00|
|(reserved)               |CT 0x01|
|(reserved)               |CT 0x02|
|(reserved)               |CT 0x03|
|(reserved)               |CT 0x04|
|(reserved)               |CT 0x05|
|(reserved)               |CT 0x06|
|(reserved)               |CT 0x07|
|(reserved)               |CT 0x08|
|(reserved)               |CT 0x09|
|(reserved)               |CT 0x0A|
|(reserved)               |CT 0x0B|
|(reserved)               |CT 0x0C|
|(reserved)               |CT 0x0D|
|(reserved)               |CT 0x0E|
|(reserved)               |CT 0x0F|
|(reserved)               |CT 0x10|
|(reserved)               |CT 0x11|
|(reserved)               |CT 0x12|
|(reserved)               |CT 0x13|
|(reserved)               |CT 0x14|
|(reserved)               |CT 0x15|
|(reserved)               |CT 0x16|
|(reserved)               |CT 0x17|
|(reserved)               |CT 0x18|
|(reserved)               |CT 0x19|
|drop.ct                  |CT 0x1A|
|select.ct                |CT 0x1B|
|select.ct t              |CT 0x1C|
|(reserved)               |CT 0x1D|
|(reserved)               |CT 0x1E|
|(reserved)               |CT 0x1F|
|local.get.ct x           |CT 0x20|
|local.set.ct x           |CT 0x21|
|local.tee.ct x           |CT 0x22|
|global.get.ct x          |CT 0x23|
|global.set.ct x          |CT 0x24|
|table.get.ct x           |CT 0x25|
|table.set.ct x           |CT 0x26|
|(reserved)               |CT 0x27|
|i32.load.ct memarg       |CT 0x28|
|i64.load.ct memarg       |CT 0x29|
|(reserved)               |CT 0x2A|
|(reserved)               |CT 0x2B|
|i32.load8_s.ct memarg    |CT 0x2C|
|i32.load8_u.ct memarg    |CT 0x2D|
|i32.load16_s.ct memarg   |CT 0x2E|
|i32.load16_u.ct memarg   |CT 0x2F|
|i64.load8_s.ct memarg    |CT 0x30|
|i64.load8_u.ct memarg    |CT 0x31|
|i64.load16_s.ct memarg   |CT 0x32|
|i64.load16_u.ct memarg   |CT 0x33|
|i64.load32_s.ct memarg   |CT 0x34|
|i64.load32_u.ct memarg   |CT 0x35|
|i32.store.ct memarg      |CT 0x36|
|i64.store.ct memarg      |CT 0x37|
|(reserved)               |CT 0x38|
|(reserved)               |CT 0x39|
|i32.store8.ct memarg     |CT 0x3A|
|i32.store16.ct memarg    |CT 0x3B|
|i64.store8.ct memarg     |CT 0x3C|
|i64.store16.ct memarg    |CT 0x3D|
|i64.store32.ct memarg    |CT 0x3E|
|(reserved)               |CT 0x3F|
|(reserved)               |CT 0x40|
|i32.const.ct i32         |CT 0x41|
|i64.const.ct i64         |CT 0x42|
|(reserved)               |CT 0x43|
|(reserved)               |CT 0x44|
|i32.eqz.ct               |CT 0x45|
|i32.eq.ct                |CT 0x46|
|i32.ne.ct                |CT 0x47|
|i32.lt_s.ct              |CT 0x48|
|i32.lt_u.ct              |CT 0x49|
|i32.gt_s.ct              |CT 0x4A|
|i32.gt_u.ct              |CT 0x4B|
|i32.le_s.ct              |CT 0x4C|
|i32.le_u.ct              |CT 0x4D|
|i32.ge_s.ct              |CT 0x4E|
|i32.ge_u.ct              |CT 0x4F|
|i64.eqz.ct               |CT 0x50|
|i64.eq.ct                |CT 0x51|
|i64.ne.ct                |CT 0x52|
|i64.lt_s.ct              |CT 0x53|
|i64.lt_u.ct              |CT 0x54|
|i64.gt_s.ct              |CT 0x55|
|i64.gt_u.ct              |CT 0x56|
|i64.le_s.ct              |CT 0x57|
|i64.le_u.ct              |CT 0x58|
|i64.ge_s.ct              |CT 0x59|
|i64.ge_u.ct              |CT 0x5A|
|(reserved)               |CT 0x5B|
|(reserved)               |CT 0x5C|
|(reserved)               |CT 0x5D|
|(reserved)               |CT 0x5E|
|(reserved)               |CT 0x5F|
|(reserved)               |CT 0x60|
|(reserved)               |CT 0x61|
|(reserved)               |CT 0x62|
|(reserved)               |CT 0x63|
|(reserved)               |CT 0x64|
|(reserved)               |CT 0x65|
|(reserved)               |CT 0x66|
|i32.clz.ct               |CT 0x67|
|i32.ctz.ct               |CT 0x68|
|i32.popcnt.ct            |CT 0x69|
|i32.add.ct               |CT 0x6A|
|i32.sub.ct               |CT 0x6B|
|i32.mul.ct               |CT 0x6C|
|i32.div_s.ct             |CT 0x6D|
|i32.div_u.ct             |CT 0x6E|
|i32.rem_s.ct             |CT 0x6F|
|i32.rem_u.ct             |CT 0x70|
|i32.and.ct               |CT 0x71|
|i32.or.ct                |CT 0x72|
|i32.xor.ct               |CT 0x73|
|i32.shl.ct               |CT 0x74|
|i32.shr_s.ct             |CT 0x75|
|i32.shr_u.ct             |CT 0x76|
|i32.rotl.ct              |CT 0x77|
|i32.rotr.ct              |CT 0x78|
|i64.clz.ct               |CT 0x79|
|i64.ctz.ct               |CT 0x7A|
|i64.popcnt.ct            |CT 0x7B|
|i64.add.ct               |CT 0x7C|
|i64.sub.ct               |CT 0x7D|
|i64.mul.ct               |CT 0x7E|
|i64.div_s.ct             |CT 0x7F|
|i64.div_u.ct             |CT 0x80|
|i64.rem_s.ct             |CT 0x81|
|i64.rem_u.ct             |CT 0x82|
|i64.and.ct               |CT 0x83|
|i64.or.ct                |CT 0x84|
|i64.xor.ct               |CT 0x85|
|i64.shl.ct               |CT 0x86|
|i64.shr_s.ct             |CT 0x87|
|i64.shr_u.ct             |CT 0x88|
|i64.rotl.ct              |CT 0x89|
|i64.rotr.ct              |CT 0x8A|
|(reserved)               |CT 0x8B|
|(reserved)               |CT 0x8C|
|(reserved)               |CT 0x8D|
|(reserved)               |CT 0x8E|
|(reserved)               |CT 0x8F|
|(reserved)               |CT 0x90|
|(reserved)               |CT 0x91|
|(reserved)               |CT 0x92|
|(reserved)               |CT 0x93|
|(reserved)               |CT 0x94|
|(reserved)               |CT 0x95|
|(reserved)               |CT 0x96|
|(reserved)               |CT 0x97|
|(reserved)               |CT 0x98|
|(reserved)               |CT 0x99|
|(reserved)               |CT 0x9A|
|(reserved)               |CT 0x9B|
|(reserved)               |CT 0x9C|
|(reserved)               |CT 0x9D|
|(reserved)               |CT 0x9E|
|(reserved)               |CT 0x9F|
|(reserved)               |CT 0xA0|
|(reserved)               |CT 0xA1|
|(reserved)               |CT 0xA2|
|(reserved)               |CT 0xA3|
|(reserved)               |CT 0xA4|
|(reserved)               |CT 0xA5|
|(reserved)               |CT 0xA6|
|i32.wrap_i64.ct          |CT 0xA7|
|(reserved)               |CT 0xA8|
|(reserved)               |CT 0xA9|
|(reserved)               |CT 0xAA|
|(reserved)               |CT 0xAB|
|i64.extend_i32_s.ct      |CT 0xAC|
|i64.extend_i32_u.ct      |CT 0xAD|
|(reserved)               |CT 0xAE|
|(reserved)               |CT 0xAF|
|(reserved)               |CT 0xB0|
|(reserved)               |CT 0xB1|
|(reserved)               |CT 0xB2|
|(reserved)               |CT 0xB3|
|(reserved)               |CT 0xB4|
|(reserved)               |CT 0xB5|
|(reserved)               |CT 0xB6|
|(reserved)               |CT 0xB7|
|(reserved)               |CT 0xB8|
|(reserved)               |CT 0xB9|
|(reserved)               |CT 0xBA|
|(reserved)               |CT 0xBB|
|(reserved)               |CT 0xBC|
|(reserved)               |CT 0xBD|
|(reserved)               |CT 0xBE|
|(reserved)               |CT 0xBF|
|i32.extend8_s.ct         |CT 0xC0|
|i32.extend16_s.ct        |CT 0xC1|
|i64.extend8_s.ct         |CT 0xC2|
|i64.extend16_s.ct        |CT 0xC3|
|i64.extend32_s.ct        |CT 0xC4|
|(reserved)               |CT 0xC5|
|(reserved)               |CT 0xC6|
|(reserved)               |CT 0xC7|
|(reserved)               |CT 0xC8|
|(reserved)               |CT 0xC9|
|(reserved)               |CT 0xCA|
|(reserved)               |CT 0xCB|
|(reserved)               |CT 0xCC|
|(reserved)               |CT 0xCD|
|(reserved)               |CT 0xCE|
|(reserved)               |CT 0xCF|
|(reserved)               |CT 0xD0|
|(reserved)               |CT 0xD1|
|(reserved)               |CT 0xD2|
|(reserved)               |CT 0xD3|
|(reserved)               |CT 0xD4|
|(reserved)               |CT 0xD5|
|(reserved)               |CT 0xD6|
|(reserved)               |CT 0xD7|
|(reserved)               |CT 0xD8|
|(reserved)               |CT 0xD9|
|(reserved)               |CT 0xDA|
|(reserved)               |CT 0xDB|
|(reserved)               |CT 0xDC|
|(reserved)               |CT 0xDD|
|(reserved)               |CT 0xDE|
|(reserved)               |CT 0xDF|
|(reserved)               |CT 0xE0|
|(reserved)               |CT 0xE1|
|(reserved)               |CT 0xE2|
|(reserved)               |CT 0xE3|
|(reserved)               |CT 0xE4|
|(reserved)               |CT 0xE5|
|(reserved)               |CT 0xE6|
|(reserved)               |CT 0xE7|
|(reserved)               |CT 0xE8|
|(reserved)               |CT 0xE9|
|(reserved)               |CT 0xEA|
|(reserved)               |CT 0xEB|
|(reserved)               |CT 0xEC|
|(reserved)               |CT 0xED|
|(reserved)               |CT 0xEE|
|(reserved)               |CT 0xEF|
|(reserved)               |CT 0xF0|
|(reserved)               |CT 0xF1|
|(reserved)               |CT 0xF2|
|(reserved)               |CT 0xF3|
|(reserved)               |CT 0xF4|
|(reserved)               |CT 0xF5|
|(reserved)               |CT 0xF6|
|(reserved)               |CT 0xF7|
|(reserved)               |CT 0xF8|
|(reserved)               |CT 0xF9|
|(reserved)               |CT 0xFA|
|(reserved)               |CT 0xFB|
|(reserved)               |CT 0xFC|
|(reserved)               |CT 0xFD|
|(reserved)               |CT 0xFE|
|(reserved)               |CT 0xFF|

