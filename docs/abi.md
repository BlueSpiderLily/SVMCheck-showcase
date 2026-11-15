# Controller ABI (v1.1)

A summary of the interface between the user-mode controller and the driver. This is documentation of the interface only, not source.

## Device and posture

- **Device:** `\\.\SvmHv`, created with a security descriptor that allows **Administrators and SYSTEM only**. The controller must run elevated; a non-elevated process gets `ERROR_ACCESS_DENIED` at open.
- **Lifetime:** created at the very end of driver load, after every guest run has completed, and destroyed first on unload. No request can race a running guest.
- **Execution model:** burst. See the [README](../README.md#execution-model-burst).

## IOCTLs

| IOCTL | Buffering | Purpose |
| :-- | :-- | :-- |
| `QUERY_STATUS` | buffered | Processor count, SVM and NPT counts, IOCTL and hypercall counters, and the location of the demo region |
| `READ_GPA` | direct out | Read guest-physical memory through one CPU's nested page table |

## Rules

- **Versioning.** Every request and reply begins with `{Version, Size}`. A different major version, a wrong size, or an unknown IOCTL code is rejected.
- **Limits.** At most 4096 bytes per call. The address plus length is checked for overflow. A read may cross at most one 4 KB boundary, and every page it touches must be mapped, or the whole call fails with no partial data.
- **Gating.** A read succeeds only where the CPU's nested page table maps the memory. Reads only; there is no write operation.
- **Demo region.** A read-only page owned and written by the driver itself, so `READ_GPA` has a stable target. `QUERY_STATUS` reports where it is, so a caller never needs a magic address.

## Status codes

The authoritative result is the `Status` field in the reply header. The Win32 error is only a coarse signal, and several statuses can map to the same Win32 error.

| Status | Meaning |
| :-- | :-- |
| `Ok` | Success |
| `BadVersion`, `BadSize`, `BadCommand` | Malformed request |
| `TooLarge`, `Overflow`, `ZeroLength` | Length or address out of bounds |
| `NotMapped` | A touched page is not mapped; no bytes returned |
| `NotReady`, `BadCpu` | Driver or CPU not available for the request |
| `BufferTooSmall`, `CopyFailed`, `OutOfResources` | Resource problems |
| `Denied` | Not permitted |

For `QUERY_STATUS` (buffered), a rejection returns success at the OS level with the real status in the reply, because the I/O manager copies a buffered reply back only on success. For `READ_GPA` (direct), a rejection returns both a failing OS status and the status field. No path bugchecks.

## CPUID hypercalls

The reserved leaf range `0x40000000`-`0x400000FF` is answered in software, ahead of the real CPUID pass-through. The signature and info leaves are open. The command leaf (`Ping`, `GetVersion`) is gated to CPL 0.

## Controller

```
SvmHvCtl.exe status
SvmHvCtl.exe read <cpuIndex> <gpaHex> <lengthDecimal>
SvmHvCtl.exe negtest          # six negative cases against the live driver
```

`negtest` covers an unknown IOCTL code, a bad ABI version, an oversize length, an address overflow, a read spanning a mapped and an unmapped boundary (no partial data), and a zero length. All six pass, each with a defined status.
