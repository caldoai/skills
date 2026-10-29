---
name: restructure-codebase
description: Restructure a whole codebase in one pass, without changing behavior, so agents working in it read less per change, break less without noticing, and debug faster.
disable-model-invocation: true
argument-hint: "[optional area or concern to focus on]"
---

Start a thorough structural refactor of this codebase and complete it to the end. Ensure behavior stays the same.

Treat this as a goal.

**Why do this at all?**

- **Better output:** agents give worse output in a poorly structured codebase, and produce better code in a well-structured one.
- **Cheaper & faster changes:** less tokens (and time) spent on searching the codebase, letting agents focus on the problem instead.
- **Breaking things silently:** when two pieces must agree and nothing shows it, an agent edits one and the other breaks silently.
- **Faster debugging:** finding the root cause of a symptom becomes much easier for agents.

**Core idea**

**1. Reason: why pieces change**

Pieces that change for the same reason must change together, so the structure should show it. Pieces that change for different reasons must not be tied. A piece with no reason of its own to change shouldn't exist.

When the structure gets this wrong, you get one of three problems:

1. **Hidden link:** the pieces must change together, but the code doesn't show it. Result: silent breaks.
2. **Needless link:** the pieces must not change together, but the code ties them. Result: blast radius, where a small change drags unrelated code along.
3. **Surplus:** the piece has no reason of its own to change. Result: everything in the codebase costs reading and searching, and this gives nothing back.

Some examples:

- **Don't repeat yourself:** copies that must change together are a hidden link. Copies that only look alike aren't linked at all, and merging them creates a needless link.
- **Self-contained code:** a module that quietly relies on something outside it, like a global another module sets or a file another module writes, must change when that does, and nothing in the module shows it. That's a hidden link.
- **Organize by domain:** a feature change touches its route, logic, model and UI. Split by layer, that's four folders for one change, a hidden link. Split by domain, it's one.
- **Colocation:** code that changes together but lives in different folders is a hidden link. Move it next to each other.
- **Explicit over implicit:** a call order or convention callers just have to know is a hidden link.
- **Encapsulation:** callers that depend on internals hold a needless link. The internals change and the callers break.
- **Pass-through wrappers:** a wrapper that changes every time the thing it wraps does has no reason of its own to change. That's surplus.

**2. Rate: how often pieces change**

Some code changes every week, some almost never. Code should depend on what changes less. A structural problem costs something every time the code around it changes, so it costs the most where code changes often.

Some examples:

- **Dependency direction:** stable code has to change whenever the volatile side does, which is a needless link.
- **Speculative generality:** extension points and config for things that never vary. Nothing will ever change them, so they're surplus.
- **Caller vocabulary:** a general function that takes a parameter only one workflow understands has to change whenever that workflow does.

Your goal is to find every reason and rate problem in this codebase and fix it.

**Process**

1. **Map:** Make a structural copy of the codebase in a temporary folder. Imports, exports, signatures, types and module-level constants stay as they are. Every function body becomes a short summary of:
   - **What it does,** in behavior words. One line per step, with its line range.
   - **What it reaches outside itself:** what it calls, reads, writes or emits, and any strings or numbers something else has to match.
   - **What must already be true:** order and shape assumptions.

   ```ts
   export function resumeSession(id: string) {
     // L<start>-<end> <what this step does, in behavior words>
     // L<start>-<end> <what the next step does>
     // outside: <what it calls, reads, writes or emits; strings or numbers something else has to match>
     // needs: <order and shape assumptions>
   }
   ```

   This doesn't need much intelligence, but it touches every file, so use cheaper subagents, each covering a few related files.
2. **Find:** Read the structural copy and list the structural problems in it.
3. **Fix:** Have subagents fix them in the real codebase.
4. **Check:** Confirm behavior is identical to where you started.