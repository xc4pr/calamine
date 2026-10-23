# Queue entry 01: xlsb BrtBundleSh panic

- Type: issue+pr
- Target repo: tafia/calamine
- Base branch: master (checked against upstream commit c922fc0a0c3cc9901e7d0b7a40507140478e5912)
- Head branch: xc4pr:fix/xlsb-bundle-sheet-panic
- Head commit: a5500fd0e2165c25bdda21d3802397cf7d743407
- Order: open the issue first, then replace `#ISSUE` in the PR text with its number.

---

## Issue

### Title

xlsb: opening a workbook panics on a malformed BrtBundleSh record or a missing sheet relationship

### Body

### Current behavior

`open_workbook` for an `.xlsb` file panics, instead of returning an `Err`, when a `BrtBundleSh` (sheet) record in `xl/workbook.bin` is malformed:

1. The record's `strRelID` (here `rId2`) is not defined in `xl/_rels/workbook.bin.rels`:

   ```
   panicked at src/xlsb/mod.rs:312:66:
   no entry found for key
   ```

2. The record body is shorter than the fixed `hsState` and `iTabID` fields (6 bytes here):

   ```
   panicked at src/xlsb/mod.rs:306:48:
   range start index 8 out of range for slice of length 6
   ```

3. The record body ends after `strRelID`, without the `strName` length:

   ```
   panicked at src/utils.rs:38:25:
   range end index 4 out of range for slice of length 0
   ```

The panic happens while the workbook is opened, so callers cannot handle it as an error. The xlsx reader returns `XlsxError::RelationshipNotFound` for the equivalent of case 1 (see #252).

### Expected behavior

`open_workbook` returns an `XlsbError`.

### Sample code to reproduce

```rust
use calamine::{open_workbook, Xlsb};
use std::fs::File;
use std::io::BufReader;

fn main() {
    let res: Result<Xlsb<BufReader<File>>, _> = open_workbook("bundle_sheet.xlsb");
    println!("{:?}", res.err());
}
```

### Test file

The files are single-sheet workbooks made from `tests/issue_666_panic.xlsb`, with `xl/workbook.bin` replaced by `BrtBeginBook`, `BrtWbProp`, `BrtBeginBundleShs`, one `BrtBundleSh`, `BrtEndBundleShs` and `BrtEndBook`. The `BrtBundleSh` body is:

1. `hsState=0`, `iTabID=0`, `strRelID="rId2"`, `strName="Sheet1"` (the rels file only defines `rId1`),
2. 6 zero bytes, or
3. `hsState=0`, `iTabID=0`, `strRelID="rId1"` and nothing after it.

The three files will be added as test fixtures (`tests/xlsb_bundle_sheet_*.xlsb`) in a PR.

### Environment

calamine 0.37.0 and current `master` (c922fc0), Linux x86_64, Rust 1.97.0.

---

## PR

### Title

xlsb: return an error for malformed BrtBundleSh records

### Body

## Summary

Opening an xlsb workbook panicked when a `BrtBundleSh` record in `xl/workbook.bin` was truncated, or when its `strRelID` was missing from `xl/_rels/workbook.bin.rels`. It now returns an `XlsbError`.

Fixes #ISSUE

## Root cause

`Xlsb::read_workbook` decoded `BrtBundleSh` with unchecked slices (`buf[8..len]`, `buf[12..12 + rel_len]`) and looked the relationship id up with `relationships[&*relid]`, which panics for a missing key. `wide_str()` checked the string length against the buffer, but called `read_u32()` first, which panics when fewer than 4 bytes are left.

## Fix

- Read `strRelID` with `wide_str()`, which already returns `XlsbError::WideStr` when the string does not fit in the record, and read `strName` from the bytes after it.
- Make `wide_str()` return `XlsbError::WideStr` when the buffer is too short for the 4 byte length prefix. Other callers of `wide_str()` get the same check.
- Report a `strRelID` that is not in the relationships as `XlsbError::Unrecognized { typ: "BrtBundleSh:strRelID", val }`. I used an existing variant because `XlsbError` is not `#[non_exhaustive]`, so a new variant would be a breaking change. Happy to change this if you prefer a dedicated variant.

A null `strRelID` (length `0xFFFFFFFF`) is still skipped as before.

The three new fixtures were made by replacing `xl/workbook.bin` in `tests/issue_666_panic.xlsb` with a minimal record stream that holds one crafted `BrtBundleSh` record. I did not edit `Changelog.md`, since it looks like it is updated at release time.

## Tests

- `cargo test --test test xlsb_bundle_sheet` with `src/xlsb/mod.rs` from `master` (c922fc0): the 3 new tests fail with the panics from the issue. With this change: 3 passed.
- `cargo test --all-features`: all passed (47 unit, 182 integration with 1 ignored, 68 doc tests).
- `cargo fmt -- --check`: clean.
- `cargo clippy --all-targets --all-features -- -Dwarnings`: clean.
- `cargo check --target wasm32-unknown-unknown --all-features`: clean.
