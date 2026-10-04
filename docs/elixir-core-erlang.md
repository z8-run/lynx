## Implementation prompt: Elixir → Core Erlang → Boogie → Z3

You are implementing a **reusable, local-first formal verification system for Elixir projects**.

Build a working system-not merely an architecture document or a translator that handles one demonstration. The complete pipeline must extract real compiled code, translate a precisely defined subset, verify user-supplied contracts, explain failures, and retain reproducible evidence.

"Complete" means an end-to-end product for its documented supported subset. It does **not** mean pretending to support every Elixir or OTP feature.

### 1. Architecture and technology

Implement this pipeline:

```text
Elixir project snapshot
        ↓
Compile with debug information
        ↓
Recover Erlang abstract forms
        ↓
Extract a pinned Core Erlang representation
        ↓
Check supported constructs and resolve dependencies
        ↓
Normalize into a small verification IR
        ↓
Generate Boogie procedures, contracts, and obligations
        ↓
Run Boogie with Z3
        ↓
Produce source-linked results and an evidence bundle
```

Use Elixir for orchestration, contract processing, translation, and reporting. Isolate compiler-specific Erlang interactions in a small adapter.

Use Boogie as the verification-condition generator and Z3 as its solver. Do not implement another SMT solver or verification-condition engine. This is the frontend/backend architecture Boogie is intended to support. [GitHub](https://github.com/boogie-org/boogie?utm_source=chatgpt.com)

The verification IR should contain only what is necessary: explicit bindings, control flow, calls, normal and exceptional outcomes, source references, and established type information. Do not recreate the entire Erlang compiler.

Do not introduce Lean, F*, TLA+, a web application, or a hosted service as dependencies.

### 2. Keep the system outside the target repository

Maintain the verifier, contracts, semantic models, caches, build snapshots, and reports in a separate workspace:

```text
elixir_verify/
  lib/
  adapters/
  semantics/
  schemas/
  contracts/
  test/
  fixtures/
  docs/

verification-workspace/
  snapshots/
  artifacts/
  cache/
  reports/
```

The target project must require no verification dependency, source annotations, or changes to `mix.exs`.

Compile a snapshot that includes the intended working-tree changes, not merely the last Git commit. Record exactly what was included.

Keep builds and dependency preparation outside the original repository. Verify that running the tool leaves the original repository unchanged.

Provide local commands suitable for later CI invocation, but do not create remote workflows, webhooks, or orchestration automatically.

### 3. Implement extraction with explicit provenance

Start by probing the available Elixir, OTP, .NET, Boogie, and Z3 versions. Select and pin a compatible, tested toolchain without silently upgrading the target project.

For each compiled module:

1. Read its debug-information chunk.
2. Invoke the declared debug backend to request `erlang_v1`.
3. Convert the returned abstract forms into the selected Core stage.
4. Serialize the Core structure without losing semantic information.

Use the debug backend interface rather than depending on the opaque internal layout of its stored data. Erlang documents this interface, and Elixir's backend provides the relevant representations. [Erlang.org](https://www.erlang.org/doc/apps/stdlib/beam_lib.html)

For the initial adapter, investigate and test this extraction profile:

```elixir
:compile.noenv_forms(forms, [
  :to_core,
  :binary,
  :no_copt,
  :deterministic,
  :return_errors,
  :return_warnings
])
```

Treat this as a **version-specific profile**, not universally correct boilerplate. Handle the actual return shapes of the pinned compiler.

Inspect compilation attributes and transformations. Do not accidentally repeat parse transforms, omit Core transforms, or override semantic compiler options. In particular, OTP's `no_copt` path can skip Core transformations; reject configurations the adapter cannot preserve. [Erlang.org](https://www.erlang.org/doc/apps/compiler/compile.html?utm_source=chatgpt.com)

Record at least:

```text
Source snapshot and file hashes
Dependency lockfile and resolved dependency identities
Relevant build configuration and environment
Elixir and OTP versions
Compilation and extraction options
Debug backend and extraction stage
BEAM and Core hashes
Contract and semantic-model hashes
Translator, Boogie, and Z3 versions
```

Preserve source annotations where available. Mark compiler-generated or unmapped locations explicitly rather than inventing source positions.

Serialize arbitrary-size integers losslessly, for example as tagged decimal strings.

**State clearly that this route reconstructs Core through debug information; it does not verify the BEAM instruction stream or prove compiler correctness.**

### 4. Define and implement the supported subset

The mandatory first release should verify useful sequential, side-effect-free functions containing:

- Integers, atoms, booleans, `nil`, tuples, proper lists, and maps with a declared finite atom-key schema.
- Arithmetic, comparisons, pattern matching, ordered clauses, guards, and branching.
- Statically resolved local and cross-module calls, including supported recursion.
- Normal returns and the exception behavior required by those constructs.

Implement the Core nodes and compiler-generated constructs needed for this subset, including failure paths. Maintain a machine-readable capability inventory for every node, primitive operation, and built-in function.

For each supported operation, retain its semantic rule, authoritative reference, implementation, and regression tests.

Initially reject reachable behavior involving unsupported floating-point operations, bit syntax, unresolved higher-order or dynamic calls, NIFs, ports, ETS, process dictionaries, external I/O, spawning, messaging, selective receive, or runtime code replacement.

Determine coverage from the actual extracted call graph. Standard-library calls are not automatically trusted merely because they are standard.

A function may use a narrower domain established by its contract. That restriction must appear in its result.

Unknown operations must produce `UNSUPPORTED`, not an optimistic implementation or an unannounced assumption. Code may be excluded as unreachable only when that exclusion is justified under the stated contract.

### 5. Preserve Erlang/Elixir semantics

Do not translate operations merely by matching names or syntax.

**Values and types.** Use explicit tagged values or justified typed specialization. Mapping an argument to Boogie `int` requires a declared integer input domain or a proved type fact. Do not treat `@spec` declarations or inferred types as established proof assumptions.

Preserve the relationship between booleans and atoms, list structure, tuple arity, map-key presence, and exact equality. Do not replace a finite Elixir map with an unconstrained total SMT map.

**Arithmetic.** Model integers mathematically under an explicit sufficient-resources assumption. Do not silently introduce machine-integer overflow.

Implement truncating division and remainder deliberately. For example:

```text
div(-3, 2) = -1
rem(-3, 2) = -1
```

Division by zero and wrong operand types must follow the appropriate exceptional or guard-failure paths. Elixir's `div/2` truncates toward zero, and `rem/2` follows the dividend's sign. [Hexdocs](https://hexdocs.pm/elixir/Kernel.html)

**Control flow.** Preserve clause ordering, pattern failures, short-circuit behavior, and evaluation behavior represented by Core.

**Guards.** A failing arithmetic operation inside a guard must not be treated like an uncaught exception from a function body. Erlang specifies that invalid operations in guards cause the guard to fail. [Erlang.org](https://www.erlang.org/doc/system/expressions.html)

**Exceptions.** Represent outcomes explicitly, conceptually:

```text
Normal(value)
Raised(class, reason)
```

Distinguish `error`, `throw`, and synchronous `exit`. Preserve propagation and supported handling behavior. Reject code that inspects stack traces until those semantics are implemented.

Do not confuse divergence with an exceptional return.

### 6. Provide external, declarative contracts

Implement a small, typed contract language stored outside the target repository. Use JSON for the container format and a restricted expression grammar for predicates.

Do not evaluate arbitrary Elixir or accept unrestricted Boogie injection when reading contracts.

Support:

```text
Module/function/arity identity
Parameter names and input domains
Preconditions
Normal-return postconditions
Allowed exceptional outcomes
Termination requirements and ranking expressions
State invariants for pure transition functions
```

For this source:

```elixir
defmodule Demo do
  @offset 2

  def calculate(n) when is_integer(n) do
    doubled = n * 2
    doubled + @offset
  end
end
```

Implement a contract equivalent to:

```json
{
  "module": "Elixir.Demo",
  "function": "calculate",
  "arity": 1,
  "parameters": [
    {"name": "n", "type": "integer"}
  ],
  "requires": "true",
  "returns": {
    "name": "result",
    "type": "integer"
  },
  "ensures": [
    "result == 2 * n + 2",
    "n >= 0 implies result >= 2"
  ],
  "raises": [],
  "termination": "required"
}
```

The parameter domain means this contract concerns integer calls. It does not establish behavior for every Erlang term.

Implement separate contracts and tests for invalid-input behavior over supported input types.

Generate contract templates when useful, but do not fabricate business requirements. Never derive the intended postcondition solely from the implementation and present the resulting agreement as independent correctness evidence.

Changing a failing contract, narrowing its domain, or adding assumptions must be a visible specification change-not an automatic repair.

### 7. Generate compositional Boogie verification

Translate implementations into procedures and contracts into verification conditions. Generate deterministic identifiers and maintain mappings from every obligation back to Core and source locations.

For the integer-specialized example, the arithmetic portion should be equivalent to:

```boogie
procedure Demo_calculate(n: int) returns (result: int)
  ensures result == 2 * n + 2;
  ensures n >= 0 ==> result >= 2;
{
  var doubled: int;

  doubled := n * 2;
  result := doubled + 2;
}
```

This specialization is acceptable only after establishing its input-domain and outcome correspondence.

For calls, prove the callee's precondition at the call site. Consume a callee summary only when its implementation is verified, its recursive proof obligations are being discharged as a valid group, or it is explicitly registered as a trusted external assumption.

Build the call graph and identify strongly connected components. Verify recursive groups coherently; do not pass callers using failed or missing callee proofs.

Separate:

```text
Partial correctness:
  Permitted outcomes satisfy the contract when execution terminates.

Exception safety:
  Disallowed exceptional outcomes cannot occur.

Termination:
  Execution cannot continue indefinitely within the modeled semantics.
```

For total correctness, establish well-founded ranking measures and the termination of relevant callees. Prove measure nonnegativity from the original input contract; do not silently strengthen that contract.

Probe whether the pinned Boogie version provides suitable measure support and reuse it where appropriate. Current upstream contains measure checking, but support must be established for the installed version. [GitHub](https://raw.githubusercontent.com/boogie-org/boogie/master/Source/Core/MeasureChecker.cs)

Do not replace an unbounded proof with fixed-depth unrolling. Any bounded analysis must have a separate result category and report its bounds.

For pure state-transition functions, prove invariant establishment and preservation:

```text
Init(state) ⇒ Invariant(state)

Invariant(state) ∧ AllowedInput(input) ∧
Step(state, input, next_state)
⇒ Invariant(next_state)
```

Do not describe these proofs as verification of an entire GenServer, scheduler, mailbox, or distributed protocol.

### 8. Prevent vacuous and accidental success

Implement explicit checks against contradictory assumptions and skipped verification.

Check consistency of the semantic prelude and satisfiability of each contract's entry conditions. Include an isolated `assert false` entry probe: successful verification of that probe indicates inconsistent assumptions. Boogie's documentation explicitly describes this vacuity hazard. [Boogie Documentation](https://boogie-docs.readthedocs.io/en/latest/LangRef.html)

A timeout or `unknown` result does not establish satisfiability. Retain the solver outcome and available witness evidence.

Keep all axioms and trusted summaries in a small, reviewed semantic library. Record their use transitively in each result.

Never use `assume false`, fabricated summaries, unchecked `free` contracts, or verification-skipping attributes to make unsupported code pass. Audit emitted assumptions and verification settings.

Track the expected procedures and obligations independently of the solver output. A successful process exit with zero relevant obligations checked is not successful verification.

Include negative canaries that intentionally fail. They must detect disabled verification, omitted implementations, swallowed solver errors, and accidental suppression of assertions.

These checks improve assurance; do not present testing or vacuity probes as a formal proof of translator soundness.

### 9. Implement honest results and diagnostics

Use explicit result categories:

| Result | Meaning |
|---|---|
| `PROVED` | All required obligations for the stated scope were discharged. |
| `CONDITIONAL` | The result depends on explicitly declared, unproved external summaries. |
| `NOT_PROVED` | One or more obligations failed; this is not automatically a concrete program bug. |
| `UNSUPPORTED` | Required semantics or extraction behavior is missing. |
| `UNKNOWN` | Solver reasoning was inconclusive or exceeded its budget. |
| `INVALID_SPEC` | The contract is malformed, ill-typed, or has inconsistent entry conditions. |
| `TOOL_ERROR` | Compilation, extraction, translation, execution, or result parsing failed. |
| `BOUNDED_ONLY` | The result covers only explicitly reported bounds. |

Report termination separately as `proved`, `not_requested`, or `not_proved`.

For each result, include the contract, input domain, source/Core identity, checked properties, dependency closure, assumptions, semantic exclusions, and toolchain.

Distinguish solver counterexample candidates from concrete bugs. Replay candidates against the exact compiled snapshot in isolation when possible. Only call a case a reproduced counterexample after replay confirms it.

Produce readable console output and machine-readable JSON. Include source-linked diagnostics, raw backend logs, and replay commands.

Never report "the repository is verified" when only selected functions were checked. Show selected roots, their dependency closure, and excluded or unexamined code.

### 10. Build a practical CLI and incremental runner

Implement commands equivalent to:

```bash
elixir_verify doctor

elixir_verify inventory \
  --project /path/to/project \
  --workspace /path/to/verification-workspace

elixir_verify verify \
  --project /path/to/project \
  --contracts /path/to/contracts \
  --workspace /path/to/verification-workspace

elixir_verify explain \
  --report /path/to/report.json \
  --obligation OBLIGATION_ID

elixir_verify replay \
  --report /path/to/report.json \
  --case CASE_ID
```

Provide a strict exit policy: success requires every mandatory root and property to satisfy policy. Missing roots, unexpected omissions, unsupported behavior, and inconclusive results must not silently pass.

Use content-addressed caching based on extracted code, contracts, semantic models, dependency summaries, compiler configuration, translator version, backend versions, and verification options.

Compile once per valid build snapshot. Reverify changed functions and affected dependencies rather than the whole project unnecessarily.

Changes to an implementation must invalidate its own evidence. Callers may retain valid evidence only when the summary they depend on remains unchanged **and its proof for the new implementation is successfully re-established**.

Apply explicit CPU, memory, concurrency, and wall-time limits. Terminate the entire subprocess tree on cancellation. Do not retry indefinitely.

Treat compilation as execution of potentially untrusted project code. Use an isolated build environment without host credentials or an exposed Docker socket. Separate dependency fetching from the restricted verification run.

Dialyzer and tests may be optional supporting checks, but they must not substitute for Boogie obligations or supply unchecked assumptions. Dialyzer is a success-typing analysis tool, not this system's contract prover. [Erlang.org](https://www.erlang.org/doc/apps/dialyzer/dialyzer.html?utm_source=chatgpt.com)

### 11. Required acceptance tests

Build fixtures demonstrating both correct verification and correct refusal.

| Fixture | Required outcome |
|---|---|
| `Demo.calculate/1` with the contract above | Proved, including required termination. |
| Change `+ 2` to `+ 3` | Not proved; reproduce a concrete violating input. |
| Negative division, remainder, and zero divisors | Correct signed results and failure behavior. |
| The same invalid operation in a guard and a body | Correctly distinguish guard failure from exception propagation. |
| Overlapping clauses, tuple patterns, list recursion, and map updates | Preserve ordering and data semantics. |
| Missing map keys and failed matches | Preserve the corresponding failures. |
| Cross-module call with an unmet precondition | Reject the caller obligation. |
| Recursive function with a valid measure | Prove the requested properties and termination. |
| Infinite recursion or an invalid measure | Never receive a total-correctness result. |
| Contradictory precondition or semantic axiom | Detect invalid specification or semantic configuration. |
| Unsupported reachable operation or unhandled compiler transform | Report unsupported; do not pass the root. |
| Solver timeout, missing executable, malformed output, or skipped obligations | Report the correct non-success state. |
| Source, contract, dependency, or semantic-model changes | Invalidate affected evidence correctly. |
| Repeated unchanged run | Reuse valid cached evidence without modifying the target repository. |

Add differential tests comparing supported Core evaluation or verification-IR evaluation with actual BEAM execution on generated inputs.

Add mutation tests for semantic rules and the translation itself. Testing is evidence against implementation errors, not a substitute for the verification claim.

### 12. Implementation sequence and deliverables

Implement in this order:

```text
1. Toolchain probe, isolated snapshot, extraction, provenance.
2. Integer-only vertical slice with real Boogie verification.
3. Contract parser, source mapping, failure classification, vacuity checks.
4. Required structured values, exceptions, calls, and recursion.
5. Incremental caching, counterexample replay, isolation hardening.
6. Full acceptance suite, documentation, and reproducible packaging.
```

At each milestone, run the relevant acceptance tests before broadening support. Do not stop after the arithmetic demonstration or declare unimplemented adapters complete.

Deliver executable code, installation instructions, a pinned toolchain manifest, contract schemas, example projects, semantic models, tests, CLI documentation, and a sample evidence bundle.

Include `SOUNDNESS.md` describing:

```text
The precise property each result establishes
The supported language and input domains
Core-to-verification-IR and IR-to-Boogie correspondence
The trusted computing base
External summaries and environmental assumptions
Compiler and runtime correspondence limitations
Known unsupported behavior
```

Document the intended assurance condition: **the verification model must not omit any supported execution that could violate the checked property**.

End the implementation report with exact commands executed, acceptance results, supported capabilities, and remaining limitations. Never present an unrun command, generated Boogie file, or successful translation as a completed proof.
