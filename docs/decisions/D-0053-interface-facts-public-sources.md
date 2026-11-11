# D-0053 — Where the interface facts come from: public sources and the library's linking facts

- Date: 2026-11-11
- Contract: §50
- Status: accepted by the maintainer on 2026-11-11, after a review of every item of the public interface.

## Context

A prospective contributor pointed out that two of the header files in VMware's VDDK package carry a "VMware
Confidential" marking, while PROVENANCE.md named VDDK's headers as the source of OpenVDDK's interface facts. They asked
whether every fact OpenVDDK uses is also available in public sources.

## The review

On 2026-11-11, 421 items of OpenVDDK's public interface were matched against public sources: 43 function prototypes, 28
types, 57 constants, 266 error codes and 27 facts about the library file.

- **400 items have a public source.**
  - Broadcom's open-source open-vm-tools (`lib/include/vix.h`, `lib/include/vm_basic_types.h`, LGPL-2.1) has every
    error code with its value, and every basic type name. `vm_basic_types.h` is the marked header in question;
    Broadcom publishes it as open source.
  - The API reference VMware ships with VDDK has 33 of the 43 prototypes and the error list.
  - VMware's *VDDK Programming Guide* (8.0 U3) has the datastore specification and the sector size.
  - nbdkit's VDDK plugin has the structure layouts, enumerations, open flags and limits.
- **About 15 of those have only weaker public sources.** For these, the only public sources are VMware's
  *NAS VAAI Development Kit 7.0.1* API reference, as posted publicly by NetApp, and projects that reproduce VMware's
  header. The items:
  - two prototypes: `GetConnectParams` and `IsAttachPossible`;
  - four constant values: `MAX_CHUNK_SIZE`, `FLAG_OPEN_COMPRESSION_MASK`, `HWVERSION_CURRENT` and `SPEC_UNKNOWN`;
  - four opaque type names;
  - four field names of the connection parameters.
- **21 items are facts of the library file VMware distributed.** They were read from its dynamic tables with
  `readelf`, `nm -D` and `objdump -T`, never from its code sections:
  - the SONAME;
  - the twelve symbol version names, their order, and which function carries which version;
  - three internal exports;
  - for the record only, the libraries it needs, its glibc floor and its build flags.

  No public document lists them. A program records the SONAME and the symbol versions when it is linked, so a
  replacement has to carry them to be loaded in its place.

No header text, code or comment was reproduced. Nothing was decompiled or disassembled.

## Decision

1. PROVENANCE.md and docs/CLEAN-ROOM.md cite the public sources above instead of VDDK's header files. A note in
   PROVENANCE.md says what they named before.
2. The linking facts stay. Binary compatibility needs them, and anyone can read them with standard tools from the
   library VMware distributed.
3. The three internal exports (`IsFaultEnabled_Func`, `SetConfigOption`, `SetInjectedFault`) stay recorded under
   `abi/`. The tracer keeps forwarding them, because VMware's own library needs them when the tracer sits in front of
   it. OpenVDDK itself does not export them (D-0008).
4. **The reference packages.** The maintainer downloaded them from VMware's public download page; until August 2026 it
   required no account. The terms that came with them are VMware's General Terms of 22 November 2023, shipped in the
   8.0.3 package as `doc/vmware-general-terms.pdf`. Their section 4.2 excludes publicly available information from
   the confidentiality duty.
