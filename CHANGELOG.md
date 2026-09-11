# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.1 — 2026-09-08

- **Declares its layer**: `layer = "core"` in the manifest — the public API requires no effects, and `novo pkg publish` now checks the code against that budget.  The layers are described under Design in the [publishing guide](https://novo-lang.org/docs/publishing.html#design).
- `bits.*` calls are the operators they lower to (`&`, `|`, `^`, `<<`, `>>>`, `~`), rewritten by `novo rewrite --bits-to-operators` where the checker proves the operands `Int`; every test vector byte-identical.  Sources reformatted to the canonical form.

## 0.1.0 — 2026-09-07

First release.

- `ip` — `Ipv4` and `Ipv6` as `@value` structs (one 32-bit number and
  two 64-bit halves); `parse_ipv4`, `parse_ipv6` and the `_at` forms
  that read a span of a larger string; `ipv4`, `ipv6`,
  `ipv4_from_int`, `ipv6_from_halves`, `ipv4_mapped` and the two zero
  constructors; octet, group, integer and `Bytes` conversions both
  ways; eight IPv4 and nine IPv6 classification predicates; `ipv4_len`
  / `write_ipv4` and `ipv6_len` / `write_ipv6`, which write into a
  buffer the caller owns. Nothing in the module allocates.
- `cidr` — `Net4` and `Net6`; `net4`, `net6`, `parse_net4` and
  `parse_net6` with a `strict` flag over host bits; `network`,
  `broadcast` (`last_address` for IPv6), `netmask`, `hostmask`,
  `contains`, `first_host`, `last_host`, `num_addresses` and
  `overlaps`. Nothing in the module allocates either.
- `show` — `addr4`, `addr6`, `net4`, `net6` and `reason`: the only
  calls in the package that return a `Str`, and the only ones that
  allocate.
- Formatting follows RFC 5952 § 4 and § 5; parsing takes RFC 4291
  § 2.2's forms and refuses what CPython 3.10's `ipaddress` refuses;
  classification follows RFC 6890's special-purpose registry as
  CPython 3.10 encodes it.
- The value types, the classification and the CIDR arithmetic
  cross-compile for `--target=nrf52-qemu` and are checked under QEMU.
  The parsers and the `write_*` calls carry `@tier(app)` and are
  pruned from a device build: the embedded runtime has no
  `novo_bytes_*` symbol and no `novo_str_byte_at`.
- Zone identifiers are `ZoneUnsupported`, dotted netmasks are
  `NetmaskFormUnsupported`, and `Net6.num_addresses()` answers `-1`
  rather than wrapping. Each is a test with its reason.
