# isomacprog <br />


**Tool for Modifying ISO Files for Macs with 64-bit Processors but 32-bit EFI (e.g., MacBook2,1 2006)**

Convert 64Bit EFI ISO's to work for 32Bit EFI Mac Pro in Bios only mode.<br />
<br />
Use Prepackaged Bins or Compile. <br />
cc -g -Wall isomacprog.c.txt. -o isomacprog <br />
 <br />
cp original.iso macversion.iso <br />
./isomacprog macversion.iso <br />
 <br />
Original Author: <br />
https://mattgadient.com/linux-dvd-images-and-how-to-for-32-bit-efi-macs-late-2006-models/ <br />
mattgadient.com <br />


**Important:** By default, **input** and **output** files must always be specified. The program first copies the input ISO to the output ISO and modifies the output file—**the original remains untouched**.

---

## EFI Detection
The tool now attempts to detect the content at the write target for common EFI formats. When executed, the program reads the target sector (the sector it intends to write) from the input image, analyzes the first bytes, and prints a short message, such as:
- `"EFI Detected: efi32"`
- `"EFI Detected: efi64"`
- `"EFI Detected: fat32"`
- `"EFI Detected: macho32"`
- `"EFI Detected: macho64"`
- `"EFI Detected: corrupted"`
- `"EFI Detected: FAT32 boot sector"`
- `"EFI Detected: Mach-O 32-bit/64-bit"`
- `"EFI Detected: Corrupted or truncated MZ/PE header"`
- `"EFI Detected: Unknown or not present"`

### How Detection Works (Summary)
- The program reads the target block (`LBA * 2048 + 64`) and checks its beginning for a **DOS-MZ signature**.
- If a DOS header is found, it parses the `e_lfanew` offset and checks for the `"PE\0\0"` signature and the **OptionalHeader magic** (`0x10b` for PE32, `0x20b` for PE32+).
- If the buffer contains **FAT32** or **Mach-O magic bytes** at well-known offsets, it reports those as well.
- If the **MZ header** is found but the `e_lfanew` leads outside the buffer or the `"PE"` signature is missing, it indicates **"Corrupted"**.

### Limitations & Caveats
- The detection only examines the sector that will be written (**currently 1984 bytes**). If EFI data is spread across multiple sectors or inside a filesystem and not at this exact sector, the detection will miss it.
- This is a **heuristic**: The presence of an MZ/PE signature in the target area is a strong hint, but it does not guarantee the image is a full, well-formed EFI binary or that it is the EFI boot binary used by a boot runner.
- Overwriting the sector may break bootability if it contains boot-critical structures. **Always keep a backup** (`cp input.iso input.iso.bak`) or test with a non-production copy.
- For better detection, an extended scan of neighboring sectors or parsing of the **ISO9660/FAT filesystem** to find `EFI/BOOT/` files would be more accurate—this is left as an enhancement.

### Example Run & Detection Output
You can test the detection with the supplied commands. A sample run shows detection like:
```
EFI Detected: efi32
```

---

## Flags
- `--check-only`: Run detection only, do not copy or modify files.
- `--scan N`: Scan **N sectors** around the calculated write target for EFI signatures.
- `--scan-all`: Scan the **entire ISO** for EFI signatures (can be slow on large ISOs).
- `--backup`: Create a backup of the input file as `input.iso.bak` before copying.

> If desired, I can add CLI flags like `--scan` to search across the ISO for `EFI` files or `--full-check` to scan multiple sectors and print file paths from ISO9660/FAT tables.

---

# isomacprog
**Brief Tool for Modifying ISO Files**

**Important:** By default, **input** and **output** files must always be specified. The program first copies the input ISO to the output ISO and modifies the output file—**the original remains untouched**.

---

## Usage
### Compilation (Standard, Host Architecture)
```bash
cc -g -Wall isomacprog.c -o isomacprog
```

### Commands / Example
```bash
# Normal usage: specify input and output
./isomacprog input.iso output.iso
```

---

## Notes
- The tool **requires** 2 arguments: `input.iso` and `output.iso`. It refuses to execute if both are identical (no in-place operation, for safety reasons).
- The program reads the **LBA address** at a fixed location (offset `32768 + 2048 + 71`) and writes a **1984-byte null sequence** (sector area) to `LBA * 2048 + 64`.
- The program checks file size, LBA validity, and whether the write area is within the file. If errors occur, execution is aborted.
- After writing, the program executes `fsync()` and reads back the written block to verify the change.

---

## Safety & Repetition
- If you apply the tool multiple times in succession to the same ISO (e.g., `out.iso`)—by using `out.iso` as the new `input` and specifying a new `out2.iso` as the `output`—the behavior is as follows:
  - The program copies the current input file to the new output and overwrites the same block with null bytes.
  - If the block is already null, the file content remains unchanged (only filesystem metadata may change).
  - Repeatedly setting the block to null typically does not further alter the file, as long as you overwrite the same area.
- **Risks:**
  - If the area the program overwrites contains critical ISO image structure/metadata (e.g., a sector with boot information), the ISO image may become invalid or unbootable. This applies whenever you overwrite content—regardless of the number of runs.
  - **Always create a backup first.** Example:
    ```bash
    cp input.iso input.iso.bak
    ```

---

## Recommended Procedure
- Use the tool only on **non-production ISOs** or after creating a backup.
- Run the program with **test ISOs** if you are unsure what data is at the target location.
- If you need automatic backups and dry runs, extend the program with `--dry-run` and `--backup` switches.
