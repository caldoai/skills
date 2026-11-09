---
name: untangle-codebase
description: 'Untangle a whole codebase in one pass, so agents working in it read less per change, break less without noticing, and debug faster. Manual only — never auto-invoke. Use when the user asks to untangle, de-duplicate, or re-architect a codebase, or says the structure is fighting them ("this repo is a mess", "untangle this", "clean up the architecture", "this code is fighting me"). Do NOT use it for a single bug, a feature, or a one-file cleanup: those need a local edit, not a repo-wide restructure.'
disable-model-invocation: true
argument-hint: "[optional area or concern to focus on]"
---

Start a thorough refactor of this codebase and complete it to the end.

Treat this as a goal.

**Why do this at all?**

- **Better output:** agents write better code in a well-structured codebase.
- **Cheaper & faster changes:** agents spend fewer tokens searching and more on the problem.
- **Fewer silent breaks:** agents break less without noticing.
- **Faster debugging:** agents find root causes faster.

**0. Set up the safety net first**

Everything below depends on being able to tell "I broke it" from "it was already broken". Do this before you read any code.

- **Baseline the tests.** Run the suite and save the list of what already fails. Without that list, the first red test after your refactor is unattributable, and you will spend more time deciding whether you caused it than the refactor saved.
- **Isolate the diff.** Work in a branch or a worktree, one commit per fix, and use the fix sentence from the decision log as the commit message. Ten structural changes in one commit cannot be reviewed or reverted, which makes the whole pass unsafe to run.
- **Check coverage where you are about to cut.** If nothing would fail when you break the code you are restructuring, capture its behavior first (run it on a handful of real inputs and save the output). If you cannot capture the behavior — a queue consumer, a webhook handler, a cron job with no harness — do not restructure it. Write it down as a finding and move on. Restructuring code you cannot observe is how a refactor silently deletes a feature.

**1. Ask**

Besides untangling the codebase, there are two other things you may do:

| Addition | What it does |
| --- | --- |
| **Comments** | Remove comments that only restate the code. |
| **Tests** | Remove tests that could not catch a real bug, and rewrite the ones that assert internals instead of behavior. |

List these (with their exact descriptions) for the user and ask which ones to do. Then do only those. A test that already fails for a reason unrelated to this refactor is not yours to fix: report it, leave it red, and keep the baseline honest.

**2. Find, before you read**

Do not read files top to bottom — on a real codebase that costs about three times what it returns. Start where change concentrates, and let the greps tell you which files deserve a full read.

- **Repeated symbols first.** Search for the same function name, the same constant, the same regex, the same `switch` over the same enum, the same object shape. Duplication is the cheapest, highest-yield finding in any codebase and one grep finds more than an hour of reading. Do this before anything else.
- **Then the dependency graph.** Script what imports what and look for the two shapes that matter: a cycle, and a leaf that half the repo depends on.
- **Then git.** Files that change in the same commits are coupled even when the imports hide it, and files that change in every commit are where a bad seam hurts most.
- **Then read.** Read in full only the files on the path of the highest-leverage flow (the one thing the product is for), plus what the greps pointed at. Skim by signatures elsewhere.

**3. List every problem**

Don't change anything yet.

Go through the codebase asking one thing: **should these pieces change together?** Write down every place the code gets the answer wrong, as in the [decision log example](#decision-log-example). There are six ways to get it wrong.

**These six are a search aid, not a filter.** A closed taxonomy only finds the defects it can name, and the worst ones often have no structural smell at all. If something is wrong and none of the six describes it, it still goes in the log under `other`, with the same `path:line`. Forcing a finding into the nearest category, or dropping it because it doesn't fit, is worse than a messy log — the next reader will find the same thing and assume you were careless.

**Hidden links:** pieces that must change together, where the code doesn't show it.

- Copies of logic or values: change one and the others must change too, but nothing says so.
- A module that reads state another module set — a global, a file, or ambient context injected by `AsyncLocalStorage`, a thread-local, or a row created as a side effect. The signature doesn't say the value is required, so nothing warns you.
- A feature split across folders by layer, or related code in different folders: one change needs all of them, but the layout hides that.
- A call order or convention callers just have to know: change it and callers break, but nothing warns them.

**Needless links:** pieces tied together that change for different reasons.

- Callers that depend on internals: the internals change for their own reasons, and the callers break anyway.
- Stable code that imports something more volatile: the stable code has to change whenever the volatile code does.
- A general function with a parameter only one workflow understands: the function has to change whenever that workflow does.
- A shallow module: the interface is nearly as complex as the implementation, so using it means knowing what's inside. Test it by deleting the module and inlining its body — if the callers are no less readable, it was only forwarding.

**Surplus:** pieces with no reason of their own to change.

- Wrappers: they only change when the thing they wrap does.
- Extension points and config for things that never vary: nothing will ever change them.
- Dead code: nothing uses it, so nothing will ever need it to change.

**Missing pieces:** the requirement is real, the code that serves it was never wired up.

- A helper, validator or template with no caller: usually a rule that was written and forgotten, not surplus. Check which before deleting — the interesting finding is the missing call, not the dead function.
- A queue, table or config value nothing consumes or reads.
- A field the type promises and the code never fills.

**Failure handling:** the system cannot tell the caller what actually happened.

- A `catch {}` that swallows infrastructure errors and reports "not found", or "no data".
- A retry or sleep that hides an intermittent bug instead of surfacing it. A retry that is the queue's declared policy is fine; one that exists because something is flaky is a finding.
- An error type that collapses distinct causes, so callers cannot react differently.

Also write down every **special case**: code that exists for one situation instead of following from how the system works. Each one is a requirement the design never accounted for, so use them to work out what the system needs to be.

- An `if` for one value
- A parameter for a specific caller
- A swallowed exception
- A retry or sleep that exists to paper over a race
- A disabled test

And one thing that is not a design smell but is always worth writing down: **wherever a credential, token or key is read, sent or compared**, say where it lives. A secret in the browser bundle, a shared token in a request body, a role compared by string instead of by permission — report it, mark it blocked if fixing it changes behavior, and never quietly keep it or quietly remove it.

**Findings have no budget; fixes do.** Write down everything you see, then say which items you are deferring and why. A silent drop is indistinguishable from a miss, and the next reader will find the same thing and assume you were careless.

**4. Walk the product, with a different question**

The six categories ask "should these pieces change together?". They cannot see a bug that lives *inside* a correctly structured piece: a filter gated by another filter's output, a counter computed on the wrong slice, a confirmation message deleted by the refresh it triggers. Nothing is coupled, nothing is duplicated, no import is wrong — and the screen lies. Those only surface if you stop asking about structure and ask about behavior.

Pick the three to five things the product exists for — open the case, resolve the payment, find the resident — and follow **one realistic input** through the code end to end. A real name, a real case number, the second page, the first day of the month. At every step ask: **what will the user see, and what did the code claim it would be?**

Two oracles are free and catch most of it:

- **Labels are claims.** Every button, column header and counter asserts something about behavior. Check the claim, not the implementation. A button that says "Delete" and calls a deactivate endpoint. A KPI labelled "total" that counts one page of 25. A search box that empties the table when you type a person's name.
- **Compose the inputs the UI lets you compose.** Search plus filter, page size plus total, an empty field plus a validation. Bugs cluster in the combinations nobody tried by hand.

Write those findings down too, then merge both sweeps into one log — a fix can serve a structural finding and a behavioral one at once, and that is usually the best fix in the list.

**5. Redesign from first principles**

Work out the fixes as if the system had known today's requirements from the start. Look for the one fix behind several problems, which often looks nothing like them. Don't fix special cases one by one — and don't assume the fix adds something. When a requirement was never accounted for, the right move is often to delete the mechanism that was standing in for it.

Ask the same thing of every fix, even one that looks better in the code: pieces that change together end up together, pieces that don't aren't tied, and everything depends on what changes less.

When more than one fix qualifies, order by what breaks loudest: widest blast radius first, then the ones where divergence is already observable (two copies that already return different answers), then the rest.

Write each fix above the problems it solves, as in the [decision log example](#decision-log-example). For any finding that claims a behavior is wrong, name the input that proves it — "search a resident's name", "a case created from the admin editor", "open page two" — before designing the fix; if you cannot name the input, it is a hunch, so label it as one. Say how two copies actually differ, in the log line itself: "identical" that isn't identical sends the next reader to delete the wrong one, and a fix built on it quietly drops a behavior. If a fix needs a product decision — which of two behaviors is correct, whether a rule still applies — mark it blocked and ask. Guessing there is how a refactor changes the product.

**6. Fix in batches**

Remove the special cases the redesign makes unnecessary, and keep the ones the system still needs. Don't go looking for bugs separately. A structural fix may fix some along the way, which is fine.

Fix in batches of three to five, and after each batch run the **full** suite, not the tests you touched: a structural change moves seams, and the callers you never opened are where it breaks. Then re-run your discovery greps — if the same duplicated symbol is still there in two places, the fix isn't finished. Then commit.

If a batch turns the suite red, the fix that caused it goes back out: revert it, write down in the log why it could not land, and keep going. Never leave the codebase worse than the baseline you recorded — a pass that trades three improvements for one broken behavior has made things worse, not better.

Some fixes change behavior no test covers: which element a UI script reads, the copy in a confirmation nobody asserts, the order two side effects happen in. Mark those in the log together with what a person should check, so they get verified by hand instead of trusted.

If the user chose Comments:

- Comments that restate the implementation, like `// sort first so the binary search works`, only change when it does, so they have no reason of their own to change. Remove them, even if they're phrased as a why.
- Comments about something outside the code, like `// the proxy drops websockets, so we poll`, change when that does, and the code can't show it. Keep them.
- Comments that explain a workaround in the codebase, like `// wait 100ms because the cache isn't ready yet`, only change when the problem behind the workaround does. They go when the redesign removes that problem.

If the user chose Tests, do this last, because tests are your regression safety net:

- Tests that repeat the implementation only change when it does. They check one interpretation against itself, so remove them.
- Tests tied to internals break when the code changes but behavior doesn't. Rewrite them to depend on behavior.
- Tests that cannot fail pass whatever the code does. Remove them.

**7. Check, and repeat until done**

Have subagents search with **both** questions — the six categories over the finished code, and the user walkthrough over the flows you touched. Partition by directory, one owner per directory, no two agents editing the same file; they report, the main thread merges and re-runs the suite (subagents declare success with red tests). Without subagents, run the greps and the walkthrough yourself, pass by pass, directory by directory.

You're done when:

- No duplicated symbol or value appears twice, and every import is reasonable — nothing imports anything more volatile than itself.
- A typical change lands in one place.
- Each fix either has a test that would fail if it were reverted, or is listed for the user with what to check by hand. If you could not observe its behavior even manually, it should not have been touched at all (step 0).
- No finding is left unaddressed without a written reason: fixed, deferred, or blocked. "As few special cases left as possible, and every remaining one named" is the best single measure of how close you are.
- Intended behavior is the same as when you started, apart from bugs the refactor fixed — proven by the suite running green against the baseline from step 0.

Stop there. "Nothing worth fixing left" is not a real stopping condition on a large codebase; you will always find one more thing, and chasing it is how the pass ends in a giant unreviewable diff. The real one is: every item in the log is closed or explained, and the suite is green against a baseline you recorded.

**Tips**

- **Start from a skeleton:** script away the function bodies, so the whole codebase fits in context.
- **Enter where leverage is highest:** the flow the product exists for, not `index.ts`.
- **Keep a decision log:** compaction is less of a problem that way, and it is the artifact that survives the session.
- **Use Git as evidence:** for how often pieces change, and what changes together.
- **Never mix the pass with a feature, a dependency bump or a rename sweep.** Those hide the diff and make the revert impossible.

<a id="decision-log-example"></a>**Decision log example**

The list of problems:

```md
- `ui/LogoutDialog.tsx:30` waits 100ms after closing and hopes the message input can take focus by then.
- `ui/Toolbar.tsx:40` and `ui/Menu.tsx:22` call the same actions, each with its own check for when they're disabled.
- `checkout/submit.ts:9` depends directly on `payments/stripe.ts`.
- `shortcuts/keymap.ts:18` has a switch with one case per action, separate from the menu's list.
- `ui/CommandPalette.tsx:55` keeps its own list of actions, and it's missing the newer ones.
- `ui/SettingsDialog.tsx:61` sets a global `skipRefocus` flag to stop the logout dialog it opens from sending focus to the message input.
- `history/undo.ts:30` has a special case for every action that can be undone.
- `refunds/refund.ts:22` wraps every Stripe call in `if (order.provider === "paypal")`.
- Every action handler has to call `track()` by hand, and some don't.
- `ui/Toolbar.test.tsx` and `ui/Menu.test.tsx` test the same actions separately.
- `mail/templates.ts:88` has an `isWithinWindow` helper with no caller, so the throttling rule it encodes is never applied.
- `jobs/sync.ts:41` catches everything and logs "no records", so a database outage looks like an empty result.
- `directory/Search.tsx:141` gates the resident list on the ids the *unit* search produced, so searching a resident's name empties the table.
```

Then with the fixes:

```md
- Make the general `Dialog` give focus back to whatever had it before it opened, so the wait and the flag can both go.
  - `ui/LogoutDialog.tsx:30` waits 100ms after closing and hopes the message input can take focus by then.
  - `ui/SettingsDialog.tsx:61` sets a global `skipRefocus` flag to stop the logout dialog it opens from sending focus to the message input.
- Define a `PaymentProvider` interface that Stripe and PayPal both implement, so checkout and refunds no longer know which one they're using.
  - `checkout/submit.ts:9` depends directly on `payments/stripe.ts`.
  - `refunds/refund.ts:22` wraps every Stripe call in `if (order.provider === "paypal")`.
- Make every user action a command with a name, a handler and an undo, so the toolbar, menu, shortcuts, palette and undo history all run from one list.
  - `ui/Toolbar.tsx:40` and `ui/Menu.tsx:22` call the same actions, each with its own check for when they're disabled.
  - `shortcuts/keymap.ts:18` has a switch with one case per action, separate from the menu's list.
  - `ui/CommandPalette.tsx:55` keeps its own list of actions, and it's missing the newer ones.
  - `history/undo.ts:30` has a special case for every action that can be undone.
  - Every action handler has to call `track()` by hand, and some don't.
  - `ui/Toolbar.test.tsx` and `ui/Menu.test.tsx` test the same actions separately.
- Call `isWithinWindow` where records are created, so the throttling rule the helper encodes actually applies.
  - `mail/templates.ts:88` has an `isWithinWindow` helper with no caller, so the throttling rule it encodes is never applied.
- Separate "the query returned nothing" from "the query failed", so an outage stops looking like an empty result.
  - `jobs/sync.ts:41` catches everything and logs "no records", so a database outage looks like an empty result.
- Give the resident list its own filter and let the unit filter only narrow units, so one search box stops emptying the other list.
  - `directory/Search.tsx:141` gates the resident list on the ids the *unit* search produced, so searching a resident's name empties the table.
```
