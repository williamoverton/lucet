# Hostcalls

Hostcalls are how Lucet guests interact with the world outside the WebAssembly
VM. For example, all functions in [WASI](https://github.com/bytecodealliance/wasmtime/blob/main/docs/WASI-intro.md) are implemented in terms of
hostcalls that can be exposed to a WebAssembly guest. This chapter discusses
implementation details of hostcalls as Lucet implements them.

Lucet implements hostcalls as imports of symbols specified by the
`bindings.json` provided to `lucetc`. This maps namespaced functions provided
to the WebAssembly module to symbol names the module should import when loaded.
Functionally, `lucet-runtime` currently relies on the dynamic linker to be able
to locate and fix up refereneces to these imported functions, and will fail to
load a module if the dynamic linker can't resolve all imports.

Hostcalls have an important intersection with safety properties Lucet seeks to
uphold: if a fault occurs in a WebAssembly guest, it should be isolated to that
guest, and the host should, generally, be able to continue execution. However,
a fault in a hostcall is a fault _outside_ the WebAssembly guest, back in
whatever code the host running `lucet-runtime` has provided. A fault here is
well outside any guarantees WebAssembly can offer, and the only sound option
Lucet has is to raise that issue in the host, if it has the option, and hope
the host knows what to do with it.

## Stack overflows

Generally, "kick the problem to the host and hope they know what to do" works.
For a general memory fault in host code, that gets handled no differently.
Language-specific features, like Rust panics, work too; if a `ud2` is present
in the hostcall's body, the `SIGILL` still causes the same kind of panic - not
a recoverable issue, but Lucet doesn't cause new problems here.

Unfortunately, memory faults like `SIGBUS` and `SIGSEGV` are typically fatal to
host applications. Memory faults in unpredictable locations even moreso. A
naive hostcall implementation scheme by calling import functions raises a real
risk here: if a WebAssembly guest consumes most, but not all, of the Lucet
guest's stack, _then_ makes a hostcall, the hostcall may consume the rest of
the guest's stack space and experience a stack overflow.

To mitigate the risk of an unknown WebAssembly guest being able to cause host
faults essentially on-demand, Lucet guards hostcalls by a trampoline function
that performs safety checks before actually making the call into host code.
Currently, there is one check: is there enough guest stack space remaining to
uphold some guaranteed amount available for hostcalls?

By guaranteeing some minimum available space, the problem of hostcall stack use
becomes the same as not overflowing stacks generally; if a host expects to
handle stack overflows in some manner, it probably still can, and if it allows
the system to do what it will on stack overflows, a hostcall overflowing the
guaranteed space will still observe a normal stack overflow. Hostcalls must
conform to the same requirements code would have without Lucet inolved, except
that the Lucet hostcall stack reservation may be more or less than the system's
configured thread size.

The good news is that while the hostcall stack reservation is a fixed size, it
is customizable: the field
[hostcall_reservation](https://docs.rs/lucet-runtime/0.7.0/lucet_runtime/struct.Limits.html#structfield.heap_memory_size)
in `Limits` specifies the space Lucet will require to be available, with a
default of 32KiB. Lucet requires that `hostcall_reservation` is between zero
and the guest's entire stack. Finally, a `hostcall_reservation` equal to the
entire guest stack size is allowed, and a de facto denial of hostcalls to the
guest - a Lucet guest will always have some stack space reserved for the
runtime-required backstop, so the availability check would always fail.

In terms of implementation, hostcall checks are done in entirely synthetic
functions generated by lucetc, prefixed with `trampoline_`. The trampoline
functions themselves are very simple, and have a shape like the following
Cranelift IR:
```
; A trampoline has the same signature as its hostcall - args plus a vmctx.
function %trampoline_$HOSTCALL($HOSTCALL_ARGS.., i64 vmctx) -> $HOSTCALL_RESULT {
    gv_vmctx = vmctx
    heap0 = static gv0

block0($HOSTCALL_ARGS.., vmctx: i64):
  ; The stack limit is recorded as part of the instance, just before the start of the guest heap.
  stack_limit_addr = heap_addr.i64 heap0, gv_vmctx, -$STACK_LIMIT_OFFSET
  stack_limit = load.i64 stack_limit_addr
  ; Compare the current stack pointer to stack_limit.
  ; The stack pointer is the LHS of this comparison, `stack_limit` the RHS.
  stack_cmp = ifcmp_sp stack_limit
  ; If the limit is greater than or equal to the stack pointer, there is
  ; insufficient space for the hostcall. Branch to the fail block and trap.
  ;
  ; "greater than or equal" may be surprising - it might be more natural to
  ; consider this comparison with arguments reversed; if the stack pointer is
  ; less than the limit, there is insufficient space.
  ;
  ; Even phrased like this, "less than" may be surprising, but is correct since
  ; the stack grows downward. Given an example layout:
  ; 0x0000: start of stack - start of the stack's allocation
  ; 0x2000: hostcall limit - reserve 0x2000 bytes
  ; 0x8000: base of stack - initial guest stack pointer
  ;
  ; and knowledge that the stack grows downward, the space from 0x2000 to
  ; 0x0000 is the reserved space, and a stack pointer below 0x2000 is in the
  ; reserved area, and thus voids the guarantee of reserved space by Lucet.
  brif ugte stack_cmp, stack_check_fail
  jump do_hostcall($HOSTCALL_ARGS.., vmctx)

do_hostcall($HOSTCALL_ARGS.., vmctx: i64):
  $HOSTCALL_RESULT.. = call $HOSTCALL($HOSTCALL_ARGS.., vmctx)
  return $HOSTCALL_RESULT

stack_check_fail():
  ; If the stack check fails, it's raised as a stack overflow in guest code.
  ; This is reasonably close to the actual occurrance, and allows the guest to be
  ; unwound.
  trap stk_ovf
```

### Lucet implementation considerations

Why do this trampoline and stack usage test, instead of observing failures in
guest stack guards? The issue here is twofold: first, we can't robustly
distinguish a stack overflow from an unlucky errant memory access for other
reasons. If hosts are built using stack probes, stack overflows will probably
be observed in places we expect, with patterns we can expect. But this is by no
means a guarantee, and stack accesses might not be through the stack pointer
directly (perhaps the address is loaded into an alternate register, and a fault
occurs without referencing the stack pointer at all). Second, if a hostcall
fault were to be recovered by Lucet, the runtime may have to unwind a guest that
already has exhausted its stack space. This would require temporarily making
the stack guard writable to support instigating a guest unwind, and probably
motivate a second "real" guard page for safety against unforseen circumstances
where unwinding might consume significant amounts of stack space.

Given the complexity and fallibility of trying to recover from a stack overflow
in guest code, we've concluded that pushing the error out of host code, with
hostcalls constrained to the same kind of boundaries as would otherwise be
expected, is a reasonable compromise.