# Rust `power_supply` abstraction + SMB347 charger driver

A Rust abstraction for the Linux **power supply class** — which does not
currently exist in mainline or `rust-next` — together with the SMBus helpers
needed to drive an I²C charger from Rust, and a Rust port of the Summit
**SMB347** battery charger as the first consumer.

Status: **RFC**, based on **v7.1-rc6**. Not yet submitted upstream.

## Motivation

Rust drivers for power-supply hardware are currently blocked: there is no safe
Rust interface to the power supply class, and the `I2cClient` abstraction
exposes no register I/O. This series provides a minimal, self-contained path
from binding an I²C charger to reporting charging state through sysfs, entirely
in safe Rust, with the `unsafe` FFI confined to the two abstraction layers.

## The series (abstraction-first)

| # | Patch | What it adds |
| - | ----- | ------------ |
| 1 | `rust: i2c: add SMBus byte transfer helpers` | Safe `smbus_read_byte_data` / `smbus_write_byte_data` / `smbus_update_bits` over `i2c::I2cClient`. `rust/kernel/i2c.rs`, +42. |
| 2 | `rust: power_supply: add power supply class abstraction` | A safe `Driver` trait, a generic `get_property` `extern "C"` trampoline, and an RAII `Registration` that owns the descriptor lifetime. `rust/kernel/power_supply.rs`, +133 (new file). |
| 3 | `power: supply: add Rust SMB347 charger driver` | A Rust SMB347 driver consuming both: binds over I²C and reports `STATUS`, `ONLINE`, and `CHARGE_TYPE`. `drivers/power/supply/smb347-charger_rust.rs`, +179 (new file). |

Full diffstat: 8 files changed, 372 insertions(+).

## Layout

```
rust/kernel/power_supply.rs                 # the abstraction (patch 2, browsable)
drivers/power/supply/smb347-charger_rust.rs # the driver       (patch 3, browsable)
patches/                                    # the full RFC series + cover letter
  0000-cover-letter.patch
  0001-rust-i2c-add-SMBus-byte-transfer-helpers.patch
  0002-rust-power_supply-add-power-supply-class-abstraction.patch
  0003-power-supply-add-Rust-SMB347-charger-driver.patch
```

The two `.rs` files above are the new files from the series, included here for
easy reading. To apply the complete change set (including the edits to
`rust/kernel/i2c.rs`, `rust/kernel/lib.rs`, `rust/bindings/bindings_helper.h`,
`MAINTAINERS`, and the `Kconfig` / `Makefile` entries), use the patches:

```sh
cd linux            # a v7.1-rc6 tree
git am /path/to/patches/000{1,2,3}-*.patch
```

## Testing

Built and exercised against an emulated SMB347 using `i2c-stub`: seeding the
chip's status registers and reading back the corresponding sysfs attributes
(`status`, `online`, `charge_type`) confirms the full C→Rust→C path. No physical
hardware or interrupt path has been tested.

## Known limitations (hence RFC)

- `get_property` recovers the driver's private data via the parent device's
  `drvdata`, relying on the observed ordering that callbacks only fire after
  `probe()` has set it; the contract should be made explicit.
- `smbus_update_bits()` is not atomic against concurrent callers; a lock will be
  required before a charger IRQ handler is added (not yet implemented).
- Only `get_property` and a handful of properties are wired up; `set_property`,
  `property_is_writeable`, and IRQ-driven `power_supply_changed()` notification
  are future work.

## License

GPL-2.0, matching the Linux kernel. The source files carry `SPDX-License-Identifier`
headers.

---

*Author: Bruce Robertson <brucer42@gmail.com>.*
