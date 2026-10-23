# Queue entry 02: xlsb shared string index panic

- Type: issue+pr
- Target repo: tafia/calamine
- Base branch: master (checked against upstream commit c922fc0a0c3cc9901e7d0b7a40507140478e5912)
- Head branch: xc4pr:fix/xlsb-shared-string-index
- Head commit: bddd9019b327395efa9e3256c8c021ff7e70c7ff
- Order: open the issue first, then replace `#ISSUE` in the PR text with its number.

---

## Issue

### Title

xlsb: reading a worksheet panics when a cell's shared string index is out of range

### Body

### Current behavior

`worksheet_range()` (or `next_cell()` on a `worksheet_cells_reader()`) on an `.xlsb` file panics when a `BrtCellIsst` cell refers to an index beyond the shared strings table. This happens for example when the workbook has no `xl/sharedStrings.bin` part, since calamine then treats the table as empty:

```
panicked at src/xlsb/cells_reader.rs:144:44:
index out of bounds: the len is 0 but the index is 0
```

The xlsx reader returns `XlsxError::Unexpected("Cell string index not found in shared strings table")` for the same case, and the xls reader skips the cell.

### Expected behavior

`worksheet_range()` returns an `XlsbError` instead of panicking.

### Sample code to reproduce

```rust
use calamine::{open_workbook, Reader, Xlsb};
use std::fs::File;
use std::io::BufReader;

fn main() {
    let mut xlsb: Xlsb<BufReader<File>> =
        open_workbook("xlsb_missing_shared_string.xlsb").unwrap();
    println!("{:?}", xlsb.worksheet_range("Sheet1").err());
}
```

### Test file

A single-sheet workbook without `xl/sharedStrings.bin`, whose `xl/worksheets/sheet1.bin` holds one `BrtRowHdr` and one `BrtCellIsst` record with `isst=0` at A1. It will be added as `tests/xlsb_missing_shared_string.xlsb` in a PR.

### Environment

calamine 0.37.0 and current `master` (c922fc0), Linux x86_64, Rust 1.97.0.

---

## PR

### Title

xlsb: return an error for an out of range shared string index

### Body

## Summary

Reading an xlsb worksheet panicked when a `BrtCellIsst` cell referred to an index beyond the shared strings table, for example in a workbook without `xl/sharedStrings.bin`. It now returns an `XlsbError`.

Fixes #ISSUE

## Root cause

`XlsbCellsReader::next_cell` indexed the shared strings slice directly with the `isst` value from the record (`&self.strings[isst]`). `worksheet_range()` and `worksheet_cells_reader()` both go through this code.

## Fix

Look the string up with `get()` and return `XlsbError::Unrecognized { typ: "BrtCellIsst:isst", val }` when it is missing. This matches the xlsx reader, which returns an error for a `t="s"` cell with an unknown index. I used an existing variant because `XlsbError` is not `#[non_exhaustive]`, so a new variant would be a breaking change. Happy to change this if you prefer a dedicated variant, or to skip the cell like the xls reader does.

The new fixture was made from `tests/issue_666_panic.xlsb`, by replacing `xl/workbook.bin` with a minimal record stream for one sheet and `xl/worksheets/sheet1.bin` with a sheet that holds a single `BrtCellIsst` cell. I did not edit `Changelog.md`, since it looks like it is updated at release time.

## Tests

- `cargo test --test test xlsb_shared_string` with `src/xlsb/cells_reader.rs` from `master` (c922fc0): the new test fails with the panic from the issue. With this change: 1 passed.
- `cargo test --all-features`: all passed (47 unit, 180 integration with 1 ignored, 68 doc tests).
- `cargo fmt -- --check`: clean.
- `cargo clippy --all-targets --all-features -- -Dwarnings`: clean.
- `cargo check --target wasm32-unknown-unknown --all-features`: clean.
