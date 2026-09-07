# ipaddr-nv

IP addresses and CIDR networks as value types. Python's `ipaddress`
and Rust's `std::net`, ported: parse, print, classify and do arithmetic
on IPv4 and IPv6 addresses without touching the heap.

```novo
use cidr
use ip
use show

fn main() [io]
    let p = ip.parse_ipv6("2001:0DB8:0000:0000:0000:0000:0000:0001")
    let a: Ipv6 = p.addr
    println(show.addr6(a))                    // 2001:db8::1
    println("${a.is_documentation()}")        // true

    let n = cidr.parse_net4("10.0.0.0/8", true)
    let block: Net4 = n.net
    println(show.net4(block))                 // 10.0.0.0/8
    println(show.addr4(block.broadcast()))    // 10.255.255.255
    println("${block.contains(ip.ipv4(10, 1, 2, 3))}")   // true
```

```
novo pkg add ipaddr-nv
```

No dependencies, and there could not be: the scanners are hand-written
loops over bytes and the standard library supplies everything they
read. There is no regular expression anywhere in this package.

## An address is a number, not an object

An `Ipv4` is one 32-bit number and an `Ipv6` is two 64-bit halves.
Both are `@value` structs, so they are copied on every binding and
every call, laid out on the stack or in `.bss` with no header and no
reference count.

```novo
use ip

let a = ip.ipv4(192, 0, 2, 1)
a.to_int()          // 3221225985
a.octet(0)          // 192
a.is_documentation()  // true
a.cmp(ip.ipv4(192, 0, 2, 2))   // -1
```

Nothing in `ip` or `cidr` allocates — not the parsers, not the
classification, not the formatters, which write into a `Bytes` their
caller already owns:

```novo
use ip

let a = ip.ipv4(198, 51, 100, 7)
let buf = ip.write_ipv4(bytes.zeros(ip.ipv4_len(a)), 0, a)
```

## The line is a module boundary

Novo has no `[alloc]` effect to declare. SPEC § 5.1's effect vocabulary
is exhaustive and a clause naming `alloc` is `E3005`, so "this
allocates and that does not" cannot be something the compiler checks on
a signature. Here it is a module boundary instead, which a build either
crosses or does not and a reader can see in the `use` lines at the top
of a file:

| Module | What it holds | Allocates |
| --- | --- | --- |
| `ip` | the address types, parsing, classification, byte and integer conversions, formatting into the caller's buffer | never |
| `cidr` | the network types, parsing, masks, membership, iteration bounds | never |
| `show` | `Str` results, and nothing else | always |

A program with no heap writes `use ip` and stops there. The claim is
kept by a test rather than by this paragraph: the suite builds a probe
that reaches all 113 functions in `ip` and `cidr`, reads the emitted
LLVM at `--opt=0`, and fails if any of them can put a cell on the heap.

That check looks for five things, not one. `novo_alloc` is the obvious
one; `novo_some_int`, `novo_some_float` and the boxed
`novo_str_byte_at` / `novo_bytes_byte_at` allocate inside the runtime
with nothing visible in the caller's IR, and a byte scanner is exactly
where that happens — the toolchain filing is
`negated-literal-defeats-the-unboxed-byte-read`.

## What runs on a microcontroller

Everything that is `Int` algebra: construction from octets or from a
32-bit number, the accessors, the comparisons, every classification
predicate, and all of `cidr`'s arithmetic. `tests/embedded_probe.nv`
cross-compiles for `--target=nrf52-qemu`, boots under QEMU and checks
36 answers there.

What does not: `parse_ipv4`, `parse_ipv6`, `cidr.parse_net4`,
`cidr.parse_net6` and the four `write_*` calls. They read a `Str` or
write a `Bytes`, and the embedded runtime defines no `novo_bytes_*`
symbol at all and no `novo_str_byte_at`. They carry `@tier(app)` and a
device build prunes them, so an embedded consumer links the half it can
use and pays nothing for the half it cannot. The two toolchain filings
are `no-embedded-byte-read-of-a-str` and
`no-embedded-str-byte-read-at-rt-tier`; when they close, the
annotations come off.

So on a device an address can be built from the bytes of a packet
header, held, compared, classified and matched against a routing table
— which is the whole of what a link-layer stack does with one — and
cannot be read out of a configuration string. The configuration is the
host's job today.

## What it follows

**Parsing** takes the forms RFC 4291 § 2.2 defines — the eight-group
form, the `::` elision, and the dotted tail that carries an IPv4
address — and refuses what CPython's `ipaddress` refuses, byte for
byte. A leading zero is `LeadingZero` and not octal; `127.1` is
`WrongPartCount` and not `127.0.0.1`.

**Printing** follows RFC 5952: leading zeros suppressed (§ 4.1), the
longest run of zero groups elided once and never a run of one
(§ 4.2), the leftmost of two equally long runs (§ 4.2.3), lower-case
hex (§ 4.3), and an IPv4-mapped address in mixed notation
(§ 5).

**Classification** follows the special-purpose registry of RFC 6890 as
CPython 3.10 encodes it. Later CPython releases have added blocks to
`is_private`; 3.10 is the release this package is checked against, and
`tests/classify_tests.nv` walks every boundary of every block in it.

## Failure has a reason

There is no `?Ipv4` — SPEC § 14.5 rejects a `@value` struct as an
optional payload, because the payload would live in a heap cell and the
whole point of the type is that it does not. So a parse answers a
struct that carries both the address and the reason:

```novo
use ip
use show

match ip.parse_ipv4("10.0.0.300").error()
    Some(e) => println(show.reason(e))   // an octet above 255
    None    => println("it parsed")
```

Thirteen reasons, each naming what the parser wanted rather than which
check refused it: `Empty`, `BadCharacter`, `WrongPartCount`,
`PartTooLong`, `EmptyPart`, `LeadingZero`, `OctetOutOfRange`,
`RepeatedDoubleColon`, `LoneColon`, `ZoneUnsupported`,
`BadPrefixLength`, `HostBitsSet`, `NetmaskFormUnsupported`.

## Where this is deliberately not Python

Each of these is a test in `tests/narrow_tests.nv`, asserting the
answer this package actually gives, with the reason it gives it.

- **Zone identifiers.** `fe80::1%eth0` is `ZoneUnsupported`. CPython
  3.9 and later keep the zone in `.scope_id`; a `@value` struct holds
  scalars only, so carrying one would mean allocating or holding a span
  into a string the address does not own. Silently dropping it would be
  worse than either: `%eth0` and `%eth1` are different destinations.
- **Dotted netmasks.** `192.0.2.0/255.255.255.0` is
  `NetmaskFormUnsupported`. Python reads it as a /24, and reads
  `/0.0.0.255` as a /24 as well — the same four octets mean two
  different prefixes depending on which way the bits fall.
- **The mixed notation.** `str(IPv6Address('::ffff:192.0.2.1'))` is
  `'::ffff:c000:201'` in CPython; here it is `::ffff:192.0.2.1`, which
  is what RFC 5952 § 5 asks for. Both spellings parse to the same
  address in both libraries.
- **No `ip_interface`.** A `Net4` is always its own network address:
  `strict` decides whether host bits are an error or dropped, never
  whether they are kept. A caller that wants an address and its prefix
  holds both.
- **A count that does not fit is `-1`.** `Net6.num_addresses()` answers
  `-1` for every prefix shorter than 66, where Python answers a bignum.
  A wrapped count would be a plausible-looking wrong number.

Not implemented in 0.1.0: `is_global`, `sixtofour`, `teredo`,
`reverse_pointer`, `exploded`, `supernet`, `subnets`,
`address_exclude`, and iteration over a network's addresses as a
sequence.

## Building and testing

```
novo pkg build                     # type-check and effect-check the library
novo test tests/addr_tests.nv      # CPython's verdict, address by address
novo test tests/format_tests.nv    # RFC 5952, and 256 generated zero runs
novo test tests/classify_tests.nv  # every boundary of every block
novo test tests/net_tests.nv       # CIDR, against ip_network
novo test tests/narrow_tests.nv    # where this is deliberately not Python
novo test src/ip.nv                # the examples in the documentation
```

The vector suites take their INPUTS from CPython's own
`Lib/test/test_ipaddress.py` and their EXPECTED VALUES from what
`ipaddress` answers for them. No CPython source is vendored and none of
its code is copied; CPython is PSF-licensed and this package is
Apache-2.0, and a table of expected answers carries neither.
