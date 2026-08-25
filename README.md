# cpacker — MBA-obfuscated executable packer (C)

A minimal UPX-style packer that compresses an executable and hides the
decompression key material with Mixed Boolean-Arithmetic (MBA) obfuscation.
Pure educational / reverse-engineering study project.

## Files

- `mathcrypt.h` — the math layer. Byte-wise affine cipher over Z/256
  (`encode`) and its MBA-obfuscated inverse (`decode_mba`). All identities are
  verified exhaustively over the 256 possible byte values.
- `packer.c` — build-time tool: deflate + encode the input, append it to the
  stub with a footer.
- `stub.c` — runtime loader: locate the appended payload, decode (MBA) +
  inflate it, write to a temp file, and exec it.

## Layout of a packed file

```
[ stub binary ][ encoded+compressed payload ][ orig u64 ][ comp u64 ][ "MBApack!" ]
```

The stub reads the 24-byte footer from the end of its own executable to find
and restore the payload.

## Build

```sh
clang -O2 -Wall stub.c   -o stub   -lz
clang -O2 -Wall packer.c -o packer -lz
```

(gcc works too: swap `clang` for `gcc`.)

## Use

```sh
./packer stub ./your_program ./your_program.packed
./your_program.packed          # runs exactly like the original
```

## What "obfuscated" means here

Only the stub ships in the packed file, and the stub only runs `decode`. So
`decode` is what an attacker disassembles — that's where the MBA lives. The
clean affine inverse compiles to four obvious instructions
(`ror; xor; sub; imul`); the MBA version compiles to ~30 tangled
`and/or/xor/add/shift` instructions with the same input→output behavior.
Verify with:

```sh
objdump -d stub | sed -n '/decode/,/ret/p'
```

## Known limitations (next steps)

- The final `imul $0x9d` (= A^{-1}) is still visible; a full MBA multiply
  would hide it.
- Byte-wise (Z/256) is weak — effectively a 256-entry substitution. Moving to
  32-bit words (Z/2^32) is much stronger.
- The MBA pattern is fixed, so it has a signature. Randomizing the identities
  per build (see plzin/mba) defeats pattern matching.
- The stub writes the unpacked binary to /tmp; a real packer would run it from
  memory (memfd_create + execveat on Linux) to avoid touching disk.
