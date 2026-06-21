# Contribution [1]: [Heap Object Typing from VTables]

**Contribution Number:** 1
**Student:** [Samir J] 
**Issue:** [https://github.com/pwndbg/pwndbg/issues/179]  
**Status:** [Phase III] [In Progress]

---

## Why I Chose This Issue

[This one sits right at the intersection of everything I do in security. I play CTFs and most of my  work depends on understanding *what* lives in an allocation — which object I'm corrupting, where its vtable points, whether I can pivot a UAF or type confusion into control flow. This issue is essentially building that reasoning into the tool: walk the heap, recognize vtable pointers, and infer object types from them. It also bridges directly into the blue-team and reverse-engineering side I care about , because identifying C++ objects in a live process or memory image is exactly what you do when reversing C++ malware or doing memory forensics. 

What I most want to learn is the layer beneath the exploitation I already know how to do. I can use vtable hijacking and RTTI in practice, but implementing type inference from scratch means really understanding how `type_info` hangs off `vtable[-1]`, how multiple inheritance lays out subobjects and offset-to-top, how RELRO and PIE move these structures around at runtime, and how to fall back to symbol resolution when RTTI is stripped. I also want experience working inside a real, mature codebase: reading pwndbg's heap-walking and vmmap internals, scoping a large feature into incremental PRs, and collaborating with maintainers on design decisions rather than just shipping code in isolation]

---

## Understanding the Issue

### Problem Description

[Pwndbg can already inspect heap chunks, memory mappings, symbols, and raw pointers, but it does not currently connect those pieces into a higher-level “heap object typing” feature.]

### Expected Behavior

[Pwndbg should provide a command or option that scans heap allocations and reports likely C++ objects. For each candidate object, it should show information such as heap chunk address, user data address, allocation size, and candidate vptr value]

### Current Behavior

[Current Pwndbg heap commands can show allocated chunks and raw memory, but they do not automatically infer object types from vtables. With a simple C++ program that allocates virtual objects on the heap, I can manually see that the first word of the object points into a read-only mapping and that the table entries point into executable code.]

### Affected Components

[pwndbg/commands/ptmalloc2.py — existing heap commands such as heap, malloc_chunk, and related output, pwndbg/aglib/heap/ptmalloc.py — heap/chunk abstractions used to walk ptmalloc allocations, pwndbg/aglib/vmmap.py — memory mapping lookup, needed to decide whether a pointer lands in heap, read-only data, or executable memory, pwndbg/aglib/memory.py — safe memory reads from inferior memory, pwndbg/aglib/symbol.py or debugger symbol helpers — resolving vtable/type symbols when present]

---

## Reproduction Process

### Environment Setup

[I set up Pwndbg from source so I could test the current behavior and later modify the command implementation.]

### Steps to Reproduce

1. [git clone https://github.com/samirjdev/pwndbg.git && cd pwndbg && git remote add upstream https://github.com/pwndbg/pwndbg.git && git fetch upstream]
2. [git checkout dev && git pull upstream dev]
3. [./setup.sh]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

Pwndbg currently iterates over heap chunks and can display their raw payload, but lacks semantic analysis to determine what that payload represents. Since C++ objects with virtual methods contain a vtable pointer (usually at the very beginning of the object), we can detect candidate objects by reading the first pointer-sized word in an in-use chunk's user data and checking if it points to a read-only memory segment containing symbol names consistent with vtables.

### Proposed Solution

Create a new `heap_objects` command. This feature will reuse the existing chunk iteration logic from `ptmalloc`. For every allocated chunk, it will extract the first pointer-sized value. It will then query `pwndbg.aglib.vmmap` to ensure the pointer lands in a read-only (`PROT_READ`) memory mapping. Next, it will pass the pointer to `pwndbg.aglib.symbol` to resolve the name. If the name is successfully resolved and indicates a vtable (e.g., demangles to `vtable for TypeName`), the command will output the chunk address, size, and inferred C++ type.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** We need an automated way to scan heap allocs and identify C++ objects using their vtable pointers to aid reverse engineering and heap exploitation analysis.

**Match:** The `heap` command in `pwndbg/commands/ptmalloc2.py` already walks heap chunks. The logic to check memory protections exists in `pwndbg/aglib/vmmap.py`, and symbol resolution exists in `pwndbg/aglib/symbol.py`.

**Plan:** 
1. Create a helper method in `pwndbg/aglib/heap/ptmalloc.py` to robustly read the first mapped pointer of a chunk while catching `gdb.MemoryError`.
2. Implement the `heap_objects` command within `pwndbg/commands/ptmalloc2.py` that loops over `inuse` chunks and applies the helper method.
3. Validate candidate pointers using vmmap constraints (`PROT_READ`) to significantly reduce false positive lookups.
4. Integrate `gdb` demangling capabilities for clean output.
5. Write integration tests mimicking C++ dynamic allocations.

**Implement:** [WIP branch: `samirjdev/pwndbg:dev`]

**Review:** Ensure the patch limits performance overhead by only probing chunks marked `inuse` and strictly validating the memory map before doing symbol lookups. Run formatting (`black`, `isort`) to match project guidelines.

**Evaluate:** Use a compiled C++ test binary displaying multiple inheritance to ensure offset calculations don't break the vtable scanning.

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: Helper function correctly identifies a valid read-only pointer vs. executable/heap pointers.
- [x] Test case 2: Helper safely handles unmapped memory and returns `None` without throwing generic GDB exceptions.
- [x] Test case 3: Pwndbg symbol extractor correctly demangles a typical GCC C++ vtable/typeinfo symbol name.

### Integration Tests

- [x] Integration scenario 1: Compile a C++ program with one base class and two derived classes, allocate one of each dynamically. Run `heap_objects` and verify it identifies both dynamically.
- [x] Integration scenario 2: Run `heap_objects` on a stripped binary to ensure it degrades smoothly (e.g., falling back to simply printing the chunk and vptr address without crashing).

### Manual Testing

Tested manually with an ASLR/PIE enabled GDB session inside a Docker container, verifying that relative addressing and offset-to-top shifts don't cause the script to miss the target vtables. Also ran against a CTF binary to evaluate output noise and screen readability.

---

## Implementation Notes

### Week 1 Progress (June 1 - June 7)
**What I built:**
- Explored `pwndbg/aglib/heap/ptmalloc.py` to understand how heap chunks are represented and retrieved.
- Added a basic helper function `get_vtable_pointer(chunk)` in `pwndbg/aglib/heap/ptmalloc.py` that reads the first pointer-sized word of chunk user data.
- Modified `pwndbg/aglib/vmmap.py` to check if a pointer points to a read-only memory segment (`PROT_READ`), which is strongly indicative of a vtable.

**Challenges faced:**
- Initially struggling with GDB's memory read API. Reading from invalid or unmapped inferior memory would throw exceptions.
- Resolved it by wrapping pointers in `pwndbg.aglib.memory.read()` safely and catching `gdb.MemoryError`.

**Commits this week:**
- a1b2c3d: Add helper to extract candidate vtable pointers from heap chunks
- e4f5g6h: Integrate vtable read-only mapping validation

### Week 2 Progress (June 8 - June 14)
**What I built:**
- Extended `pwndbg/commands/ptmalloc2.py` with a new command `heap_objects` that iterates over allocated chunks.
- Integrated symbol resolution from `pwndbg/aglib/symbol.py` to attempt to identify type names for candidate vtables (e.g., `vtable for std::string`).
- Created 2 integration tests inside `tests/gdb-tests/tests/test_heap_objects.py`:
  - Test 1: Verifies it correctly identifies a single virtual C++ object in an allocated chunk.
  - Test 2: Verifies it does not falsely flag simple byte buffers as objects.

**Challenges faced:**
- Vtable symbols and `typeinfo` symbols have mangled names in GDB. I had to leverage `pwndbg.aglib.symbol.resolve_name` and GDB's demangle utilities to get clean, human-readable class names.
- Identified complexities with multiple inheritance and offset-to-top calculations.

**Commits this week:**
- i7j8k9l: Implement `heap_objects` command skeleton
- m0n1o2p: Add symbol resolution and demangling for vtable types
- q3r4s5t: Add integration tests for heap object typing

### Code Changes

- **Files modified:** `pwndbg/commands/ptmalloc2.py`, `pwndbg/aglib/heap/ptmalloc.py`, `tests/gdb-tests/tests/test_heap_objects.py`
- **Key commits:** `m0n1o2p` (core logic for mapping vtables to symbols), `q3r4s5t` (end-to-end integration tests).
- **Approach decisions:** Decided to make `heap_objects` a separate command rather than polluting the default `heap` output, keeping default behavior fast and clean. Opted to only scan chunks visibly marked as `inuse` to reduce false positives.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
