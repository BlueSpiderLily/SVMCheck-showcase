# Design decisions

Notes on the choices behind SVMCheck and the problems that hardware testing exposed. Each one was found and verified on real hardware.

## 1. Burst execution, stated up front

The execution model is decided in the ABI header before any interface existed, because it shapes the interface. The driver runs a guest to completion on a pinned thread and returns, so Windows is never the guest and no request can observe a guest mid-run.

A permanent model would need an IPI-reachable, non-blocking handler at elevated IRQL, with no pageable code, no blocking copies and no fast mutex, and reads would race live page-table updates. That is deliberately out of scope, and the ABI says why, so it can't be stretched to it by accident.

## 2. Fail closed

- **Exceptions and shutdown.** The guest's descriptor tables are copied from the host, but the guest runs under a much smaller address space that doesn't map them. A guest exception would have vectored through an IDT it can't read and cascaded toward a machine reset. All 32 exception vectors and the shutdown event are now intercepted, so a guest fault becomes a safe `#VMEXIT` back to the host.
- **Unknown exits.** Anything the dispatcher doesn't recognise stops the guest rather than resuming it.
- **Nested page faults.** The default for any fault is to stop the guest. Only faults inside an allow-listed window, for an allowed access, are ever resolved.

## 3. Re-prime the guest on every entry

`VMRUN` saves the guest's exit RIP back into the VMCB. A second entry without re-priming would resume wherever the guest last stopped, which here is a `jmp $` fallback. That loop never faults and never hits an intercept, and because interrupts are disabled until the host regains control, it would hang that core permanently, with no crash and no timeout. Re-priming the guest on every entry makes its starting state independent of lifecycle history.

## 4. Don't trust "it compiled"

While wiring up the first real `VMRUN`, I found that the I/O and MSR permission map base addresses in the VMCB had never been set, even though `VMRUN`'s consistency checks require them. The fix was per-CPU permission maps (3 pages for I/O, 2 for MSRs), zeroed so nothing extra is intercepted. Reviewing a path that had only been compiled caught a gap that only a real entry would expose.

## 5. MSRs: virtualize, validate, never touch the host

MSR interception uses an explicit permission map. Reads are served from saved guest state. Writes are emulated against the guest's saved fields with architectural validation, and an invalid write injects `#GP(0)`. The exit handler never touches host MSRs. Time-stamp counter and host-save-area registers stay read-only. The injected `#GP` is checked end to end through the guest's IDT (vector, error code, faulting RIP, host MSRs unchanged).

## 6. Nested paging: windows, ceilings, and a retry bound

- Faults are resolved only inside registered identity **windows**, and each window's grant set is a **ceiling**. A fixed-up page carries the union of the accesses made so far, capped by the ceiling.
- Permissions only ever **widen**. Not-present versus protection faults are decided from the handler's own page-table lookup, and the hardware's error code is cross-checked. A disagreement is refused and logged.
- A **retry bound** stops a repeated fault on the same address at the same instruction, so a bad window can't spin the host.
- Every page-table change requests a TLB flush.

## 7. Split a 2 MB leaf instead of widening it

When a fault lands inside a 2 MB leaf, the normal policy runs first (ceiling, restrictions, already-granted, error-code cross-check), and any refusal never splits. If it resolves to a widening, the leaf is split into a page table of 512 entries that replicate the leaf's permissions, and only the faulting page gets the wider permission.

The leaf itself is never widened. The other 511 pages keep exact permissions, so per-page write detection survives. Tests confirm one table page is consumed, the other 511 entries are exact, and another page's first write still faults and is widened alone.

## 8. What real hardware taught me

- **Accessed and Dirty bits.** The CPU sets them on page-table entries behind your back. My first test compared entries exactly and failed (`0x25` against `0x05`; a directory entry `0x07` against `0x27`). The tests now mask bits 5 and 6 when comparing.
- **Buffered IOCTL replies.** For a buffered IOCTL, the I/O manager copies the reply back to the caller only when the request completes successfully. A failing status left the caller's buffer untouched even though the handler had written into it. So a status query reports rejections with success and the real outcome in the reply header, while the direct-I/O read writes straight into the caller's pages and can return both. The ABI documents this asymmetry instead of claiming uniform behavior.
- **A test bug of my own.** A boundary-crossing read test assumed a lone 4 KB page. Once the demo region became a 2 MB leaf, it no longer crossed anything, and the test had to be rewritten to cross the real edge.

## 9. A small, honest control surface

- Memory inspection is gated by the nested page table, not by a list. A guest-physical read succeeds only if the CPU's table maps every page it touches, and bytes come from the host-physical address the table yields, never from an arbitrary one.
- There is **no write primitive**.
- Reads are bounded to 4096 bytes and are all-or-nothing, with no partial data on failure.
- The device is restricted to Administrators and SYSTEM, created only after every guest run has finished, and destroyed first on unload.
- No code path bugchecks on bad input. Each rejection maps to a defined status.
