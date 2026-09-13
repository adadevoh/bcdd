# BCDD — Behaviour Contract Driven Development

The instruction manual for anyone — human or agent — developing on these projects. It is the North
Star for code generation, validation, and delivery. It is the method; the project's own documents
are the content.

## 0. Status of this document

**This file does not change from project to project.** It is the method, and it is deliberately free
of product, stack, and domain detail. Copy it to the root of a repo unmodified.

Everything that varies lives in the project's own artifacts:

- the product, domain, stack, and guardrails → `<project>.design.md`
- the observable behaviour of each slice → `bc/<slice>/<slice>.bc.md`
- the mechanism, where mechanism needs agreeing → `*.spec.md`

If you want to edit this file to fit a project, what you actually want is a guardrail in that
project's design doc. Editing this file changes how *every* project works, so it is a deliberate
decision about the method — never a project decision, and never a way to accommodate one codebase.

Where this document illustrates a rule with HTTP, that is illustration and not requirement. BCDD
applies to any surface a caller can observe: an HTTP API, a command-line tool, a library, a screen,
a scheduled job, a message consumer. Read "route" as "operation" and "status code" as "result code"
wherever your project's surface differs. The project's design doc fixes which form applies.

## 1. The idea

A **Behaviour Contract (BC)** is a written, negotiated agreement about *observable behaviour* for
one slice of a system: the operations, the inputs, the success and failure results, what survives a
restart, who is allowed to see what. It says nothing about classes, layers, or libraries.

The behaviour is the behaviour. Its implementation is a detail.

BCDD is TDD/BDD's instinct — describe the behaviour before you build it — formalised into a durable
artifact that lives in the repo beside the code and changes in lock step with it. A BDD scenario
lives inside a test file and is discovered by reading the suite. A BC is agreed *before* either the
implementation or the tests exist, and it outlives any particular test.

That single property is what makes BCDD worth the ceremony: because the contract exists outside the
code, **any number of humans and agents can work from it at once**, and every one of them can be
checked against the same thing.

## 2. The artifact hierarchy

```
<project>.design.md          product, domain, flows, guardrails, behaviours, iteration plan
  └── bc/<slice>/<slice>.bc.md              observable behaviour for one slice
        ├── bc/<slice>/<slice>.<topic>.spec.md   0..N — how, when how needs agreeing
        └── bc/<slice>/<slice>.test.md           0..1 — only when tests are handed off
              └── code + tests                   executable truth
```

| Artifact | Answers | Never contains |
| --- | --- | --- |
| `*.design.md` | What are we building and why? What is true of the product? | Operation-level detail: exact inputs, results, error codes |
| `*.bc.md` | What does a caller observe? | Class names, libraries, schema |
| `*.spec.md` | How is it built? | Behaviour, result codes, product rules |
| `*.test.md` | Which cases prove the BC, and how is the suite arranged? | New behaviour not in the BC |
| Code + tests | What does it actually do right now? | — |

One project has one design doc and many BCs. A BC has zero or more specs. Most BCs need no spec at
all.

## 3. Behaviours and IDs

The design doc lists every product behaviour. Each behaviour has a **behaviour ID** and maps to
exactly one `*.bc.md` (or is marked *BC not written* until that file exists).

IDs are the thread from design through testing onto release. They do not require a ticket system.
In a shop that uses one, a story may point at a behaviour ID. Without a tracker, the ID still does
the same job: it is the name the design doc, the BC, the tests, and the release all share.

There are **two grains**. Mixing them is the most common way this idea goes wrong.

**Behaviour ID** — a product slice. Lives in the design doc. Maps to one BC file. This is the unit
of a change list, an iteration, and a release note.

Format: `BH-<SLUG>` (example: `BH-AUTH`). Stable once issued; do not reuse a retired ID for a
different behaviour.

**Case ID** — one observable outcome inside that BC. Lives only in the BC. Maps to the test that
proves it.

Format: `<SLUG>-<TOKEN>` where `<SLUG>` matches the behaviour (example: `AUTH-L2`). Stable once
issued; do not reuse a retired ID for a different case.

```
design.md  [BH-AUTH]  "owners have accounts and sessions"
    └── auth.bc.md
          ├── [AUTH-L1] login success  →  Login_AfterSignup_ReturnsToken
          ├── [AUTH-L2] wrong password →  Login_WrongPassword_ReturnsUnauthorized
          └── ...
```

A change list and a release name **BH-AUTH**. CI and review ask whether every **AUTH-*** case still
has a test. Case IDs do not appear in the design doc, a changelog, or a ticket unless the change
really is that one outcome.

A BC with no matching behaviour ID in the design doc means one of the two is wrong. A behaviour ID
with no BC is a planned slice, not an implemented one.

Tests declare the case ID they prove. The mechanical check is a test in the suite: it reads the
design-doc catalogue for BCs marked `done`, reads every `*.bc.md` for case IDs, and reflects over
the test assembly for the declarations. It fails when a done case has no test, or a test names an
ID that is not in any BC. That proves coverage of the contract, not that the assertions are
philosophically correct — but it turns "did we implement what we contracted?" into something you
can run. BCs that are not `done` are not required to have tests yet.

## 4. Precedence, and the amendment protocol

When two artifacts disagree, the higher one wins:

1. `*.design.md` — wins on any product question
2. `*.bc.md` — wins on any behaviour question, including over its own specs
3. `*.spec.md` — wins on mechanism
4. Code and tests

Code and tests are not the bottom of a hierarchy of importance; they are the only *executable* truth.
The ladder resolves *intent*, not fact. Code that contradicts a BC is a bug in one of the two, and
the answer is never to leave both standing.

**Implementation will prove assumptions wrong. That is expected, and it is the moment the method
earns its keep.** When it happens:

1. **Stop.** Do not write code that contradicts a document.
2. **Pick the layer.** Product question → design doc. Behaviour question → BC. Mechanism → spec.
3. **Amend that document.** If you add, remove, or change a behaviour, issue or retire its
   behaviour ID in the same step. If you add, remove, or change an outcome, issue or retire its
   case ID in the same step. Never silently reuse an ID.
4. **Then** change the code and tests to match.
5. If the amendment invalidates work someone else is doing against the same BC, say so before
   continuing. A silent contract change is worse than a wrong contract.

The documents are not a record of what you planned. They are a description of what is true.

## 5. Anatomy of a BC

A BC is short — one to three pages. It has:

- **Header** — which iteration, the **behaviour ID** it implements, and one line of that
  behaviour from the design doc. A BC with no matching behaviour ID means one of the two is wrong.
- **Scope** — and an explicit *out of scope* list. The out-of-scope list prevents the most common
  agent failure, which is helpfully building the next three iterations.
- **Precedence line** — "if design, tests, or code disagree with this file, this BC wins (unless the
  product changed — then update the design doc and this file first)."
- **Surface** — where the behaviour is reachable, including how tests reach it.
- **Authorisation and scoping** — who may call, how identity is established, and what a caller sees
  when they ask for something that is not theirs.
- **The surface, case by case** — for each operation: how it is invoked, what it accepts, the
  success result, and *every* failure with its real result code, stable error code, and **case ID**.
  For an HTTP service that is a route with a method, body, and status codes; for a command-line
  tool, an invocation with arguments and exit codes; for a library, a call with arguments and
  thrown errors.
- **Rules** — invariants a reader could not infer from the operation list. Distinct observable
  rules get their own case IDs.

Non-negotiables:

- **Real result codes and stable error strings.** A sketch is a sketch; the BC carries what the
  system actually returns. Error codes (`email_taken`, `storage_unavailable`) are part of the
  contract because callers branch on them.
- **No class names, no libraries, no schema.** If you feel the urge, you want a spec.
- **Failures are first-class.** Most contract disputes are about what happens when something goes
  wrong, so the failure list is usually longer than the success list.
- **State what must not happen.** "Never report a store failure as an empty list." "Another owner's
  id returns 404, not 403." These are the lines that survive contact with an implementer.

## 6. A BC ships with a scaffold

This is the part that makes BCDD more than documentation.

**Landing a BC means landing the BC file *and* a minimum implementation** — every operation, wired
and reachable, with real input and output shapes, real authentication and validation, and stubs that
do nothing but return an agreed *unimplemented signal*.

The scaffold must:

- **Compile, boot, and dispatch.** Every operation in the BC exists and is reachable.
- **Return one agreed unimplemented signal.** Each project names it once, in its design doc, and
  uses it everywhere. It must be impossible to mistake for a real result — an HTTP service typically
  uses `501` with a fixed error code; a library throws a distinct not-implemented error; a tool
  returns a reserved exit code.
- **Carry the real shapes.** Input and output types match the BC so tests compile against them.
- **Enforce what is free.** Authentication, authorisation, and input validation are cheap to wire
  and are part of the contract, so the scaffold implements them for real.
- **Fake nothing.** A stub that returns invented data is worse than no scaffold, because tests pass
  against a lie.

Why it matters: a test written against a scaffold fails for a *legible* reason. It fails on an
assertion, or on the unimplemented signal — never on a compile error, a missing operation, or a null
reference. That distinction is the entire difference between "not built yet" and "the contract is
wrong", and it is what lets a test author and an implementer work at the same time without blocking
each other.

**The unimplemented signal is a debt marker, and it is tracked.** A slice may not be called done
while any of them remain in it. Enforce it mechanically: a check that fails the build when a BC
marked done still returns the signal.

## 7. Concurrent development

A BC plus a scaffold is a seam. Once both exist, work can split:

| Role | Owns | May not |
| --- | --- | --- |
| Implementation | source files behind the stubs | change the BC alone; edit the test files |
| Test | test files and fixtures | assert on internals; add behaviour absent from the BC |
| Either | proposing an amendment (§4) | amend silently |

Rules that keep the seam clean:

- **Tests bind to the contract, not the implementation.** They call the surface the BC describes
  and declare the case ID they prove. A test that reaches inside a service will break on every
  refactor and is not testing the contract.
- **File ownership is by role**, so two agents running at once do not collide. If both need the same
  file, the slice is too big — split the BC.
- **The BC is the only shared mutable artifact**, so changing it is an announced act (§4).
- **"Done" is defined by the BC's case list**, not by either party's judgement. Both sides are
  checking against the same enumerated case IDs, which is why the case list must be exhaustive.

This works the same whether both roles are agents, both are people, or one of each. That is the
point: BCDD is a single source of truth *between* stages of the SDLC, whoever executes them.

## 8. Specs — only when *how* needs agreeing

Write a `*.spec.md` when the mechanism itself deserves a decision record. Signals: a new datastore,
a new dependency, a schema, a migration strategy, a test topology, anything touching auth, anything
hard to reverse.

A spec contains: the decisions and what they rule out, the file layout, the schema, error mapping,
configuration and where it comes from, local setup steps, the test topology, and pointers to the
BC's case IDs — not a restated case list.

A spec never restates behaviour. If a result code appears in both a BC and its spec, delete it from
the spec. **Duplicated facts drift**, and the copy that drifts is always the one you read.

Skip the spec when the slice only adds operations over machinery already decided. Most do.

## 9. Test docs — only for a handoff

Write a `*.test.md` when the test work goes to a *different* agent or person than the
implementation, and only then. It carries the case IDs derived from the BC, the fixture and
isolation strategy, and an explicit statement of what must not be asserted.

If you are writing both the code and the tests, skip it — the BC's case list already is the test
plan, and a second copy is the drift described above.

A document that exists only to instruct an agent is a prompt, not an artifact. Prompts are worth
committing when a long autonomous run depends on them. Otherwise they rot.

## 10. Guardrails belong in the design doc

Guardrails are the standing rules that bind humans and agents alike. They live in the design doc
because they cut across every BC. Keep them in these groups:

- **Product invariants** — the rules that make it this product and not a generic one.
- **Configuration security** — what may never be committed; what must come from the environment;
  what must fail to boot rather than start in a degraded state.
- **Operational** — identity and authorisation rules, what must be enforced by the datastore rather
  than application code, how failures must surface, and what must never be reported as a success.
- **Test** — where tests live, what a run may depend on, and that tests may not be skipped,
  filtered, or deleted to make a change look green.
- **Engineering path** — the locked stack, fixed names, and the instruction to implement *this*
  iteration only, with the smallest change that lands the behaviour.
- **Data and privacy** — what must be retained, what must be minimised, what may not appear in URLs
  or logs.

Write them as prohibitions with reasons. "Do not X" is followed nine times out of ten. "Do not X,
because Y" is followed the tenth time too, when the rule is inconvenient.

## 11. Iterations and definition of done

The design doc carries an iteration plan, and **an iteration is a set of behaviours**. Each
behaviour is listed with its ID, its BC, and its status. Slices small enough to finish beat slices
that look tidy.

A **BC is done** when:

- every case ID in it is declared by at least one test, and those tests pass in CI
- no unimplemented signal remains in its surface
- the BC, its specs, and the code agree — including any amendment made along the way
- the guardrails it touches are still true

A **behaviour is done** when its BC is done. An **iteration is done** when every behaviour in it is
done and the design doc says so.

CI is where the contract stops being a promise. It runs the full suite on every push, plus whatever
checks catch drift that green tests cannot see — a case ID with no test, a test pointing at a
retired ID, a stale migration, a stub left in a finished slice. Never make a change look green by
removing what proves it.

## 12. Anti-patterns

- **The same fact in six places.** The most likely failure of this method. Every fact has exactly
  one home: product in the design doc, behaviour in the BC, mechanism in the spec. Everything else
  points at it.
- **Case IDs in the design doc.** The design doc lists behaviours. Cases live in the BC.
- **A BC that names classes.** It is a spec wearing a BC's name, and it will forbid refactors that
  the contract does not care about.
- **A spec that restates behaviour.** See above about drift.
- **Sketch result codes in a contract.** Placeholders from a first draft become the thing an
  implementer builds.
- **Prompt files pretending to be artifacts.** If nothing reads it but an agent, once, it is a
  prompt.
- **Documents written after the code.** Then they are a changelog, and they cannot resolve
  disagreements.
- **A scaffold that returns plausible fake data.** Tests go green against nothing.
- **Amending the contract quietly to match what you built.** The contract exists precisely to make
  that a conversation.
- **Reusing a retired ID.** The trace from an old release to a test then points at the wrong thing.

## 13. Starting a new repo

1. Write `<project>.design.md`: the product, the domain objects, the flows, a **behaviour catalogue
   with IDs**, the guardrails, an iteration plan naming those IDs, and this project's unimplemented
   signal.
2. Copy this file to the repo root, unmodified.
3. For the first iteration, write each BC under `bc/<slice>/<slice>.bc.md`, each stating the
   behaviour ID it implements and assigning a case ID to every observable outcome.
4. Add a spec only for slices where the mechanism needs deciding.
5. Land the scaffold for each BC: operations, shapes, auth, validation, stubs returning the
   unimplemented signal.
6. Split the work — implementation and tests, in parallel if you have the hands for it. Tests
   declare the case IDs they prove.
7. Set up CI: full suite on every push, plus drift checks (including case-ID coverage once you
   have the check).
8. When an assumption breaks, amend the document first (§4).
9. Keep the README pointing at the design doc rather than restating it.

## 14. A BC in miniature

One example, for an HTTP service. The shape of the sections carries over to any surface; the routes
and status codes are what that surface happens to use.

```markdown
# Sessions Behaviour Contract

Implements **BH-SESS** — `<project>.design.md` "a person can sign in on several devices".

Iteration 1. Observable behaviour only. No class names. If design, tests, or code disagree with
this file, this BC wins (unless the product changed — then update the design doc and this file first).

Scope: sign in, sign out, and what a token authorises.
Out of scope: password reset, refresh tokens, sign-out-everywhere.

## API — `{hostAddress}/auth`

### POST /auth/login
- Body: `{ email, password }`
- **[SESS-L1]** Success: **200** + `{ id, email, token }`; a new session is created even if others exist
- **[SESS-L2]** Fail (unknown email or wrong password): **401** `{ error: "invalid_credentials" }`; no token
- **[SESS-L3]** Fail (store unreachable): **503** `{ error: "storage_unavailable" }` — never 401

### POST /auth/logout
- Header: `Authorization: Bearer {token}`
- **[SESS-O1]** Success: **204**; that session ends and that token never works again
- **[SESS-O2]** Fail (missing, expired, or already-revoked token): **401**

## Rules
- **[SESS-R1]** Sign-out ends only the presented session; other devices stay signed in
- **[SESS-R2]** A revoked session stays revoked across a restart
```

Note what is absent: no service names, no table names, no library. An implementer may choose any of
those. An implementer may not choose to return 403 instead of 404, or to sign every device out at
once — those are the contract.
