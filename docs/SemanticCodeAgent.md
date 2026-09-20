# SemanticCodeAgent

## Purpose

SemanticCodeAgent is the AllasCode code-intelligence agent responsible for turning source code into a canonical semantic representation that can be queried, compared, constrained, learned from, and verified.

Its goal is not only to understand code after it is written. It should progressively reduce the space of code the agent is allowed to generate by combining language-server semantics, syntax, type information, scope, behavior declarations, vectors, causal memory, diagnostics, tests, and runtime proof.

The central artifact is:

```text
lsp_semantic.2flow
```

This file represents the semantic structure that actually exists in source code, normalized independently of the concrete programming language.

It complements:

```text
behavior.2flow
= what should happen

lsp_semantic.2flow
= what structurally exists

runtime_trace.2flow
= what actually happened
```

Together they create a semantic evidence chain from declared intent to implementation and observed execution.

---

## 1. Semantic layers

SemanticCodeAgent operates across four distinct semantic authorities.

### 1.1 Language semantics

Provided by the compiler/toolchain, preferably through LSP.

Examples:

- Zig -> ZLS
- Rust -> rust-analyzer
- Go -> gopls
- TypeScript/JavaScript -> TypeScript Language Server
- Python -> Pyright-compatible server
- Haskell -> HLS

### 1.2 Architecture semantics

Provided by AllasCode declarations:

- Agent
- Intent
- Behavior
- Action
- Entity
- Capability
- Policy
- Constraint
- Skill
- Semantic Type

### 1.3 Behavioral semantics

Provided by executable tests and conformance declarations.

### 1.4 Runtime semantics

Provided by:

- Proof
- Governor
- runtime traces
- Event Sourcing
- Runtime acceptance

No layer replaces another. They corroborate one another.

---

## 2. LSP semantic vocabulary

The LSP semantic-token vocabulary is useful as the lowest common semantic layer across languages.

Canonical token kinds include:

```text
namespace
type
class
enum
interface
struct
typeParameter
parameter
variable
property
enumMember
event
function
method
macro
keyword
modifier
comment
string
number
regexp
operator
decorator
```

Relevant semantic modifiers include:

```text
declaration
definition
readonly
static
deprecated
abstract
async
modification
documentation
defaultLibrary
```

LSP symbol kinds additionally expose a structural symbol tree:

```text
File
Module
Namespace
Package
Class
Method
Property
Field
Constructor
Enum
Interface
Function
Variable
Constant
String
Number
Boolean
Array
Object
Key
Null
EnumMember
Struct
Event
Operator
TypeParameter
```

These two views must not be conflated.

`SymbolKind` models the structural program tree.

`SemanticTokenType` models the contextual semantic role of individual lexical elements.

For example:

```zig
const customer: Customer = getCustomer();
```

may be represented as:

```text
Symbol:
customer -> Variable

Semantic tokens:
const      -> keyword
customer   -> variable + declaration + readonly
Customer   -> type
getCustomer -> function
```

---

## 3. Canonical semantic normalization

SemanticCodeAgent must never expose provider-specific semantics as the canonical contract.

The pipeline is:

```text
source code
  ->
language-specific LSP
  ->
SemanticCodeProvider
  ->
AllasCode normalization
  ->
lsp_semantic.2flow
```

This is critical because ZLS, rust-analyzer, gopls, HLS, Pyright and TypeScript LS may expose different levels of detail.

The normalized AllasCode representation should contain at least:

```text
LspSemantic
  symbols
  tokens
  relations
  types
  scope
  diagnostics
  source locations
  provider capabilities
```

Canonical relations may include:

```text
contains
defines
references
implements
calls
returns
accepts
reads
writes
depends_on
inherits
instantiates
imports
exports
```

Not every relation is directly returned by every LSP. Some are derived from references, call hierarchy, hover/type information, AST context, completion information, implementation queries, or compiler metadata.

Derived relations must always remain distinguishable from directly observed relations.

---

## 4. lsp_semantic.2flow

`lsp_semantic.2flow` is a generated artifact. It should not normally be handwritten.

Example source:

```zig
pub const Runtime = struct {
    governor: Governor,

    pub fn execute(
        self: *Runtime,
        action: Action,
    ) !Result {
        return self.governor.accept(action);
    }
};
```

Possible normalized representation:

```text
[File runtime.zig]
  -> [Struct Runtime]

[Struct Runtime]
  -> [Field governor: Governor]
  -> [Function execute]

[Function execute]
  -> [Parameter self: *Runtime]
  -> [Parameter action: Action]
  -> [Type Result]

[Function execute]
  -> calls [Method Governor.accept]

[Field governor]
  -> references [Type Governor]

[Parameter action]
  -> references [Type Action]
```

Nodes may carry semantic attributes:

```text
[Function Runtime/execute] {
  symbol_kind: Function,
  token_type: function,
  definition: true,
  exported: true,
  language: zig,
  provider: zls,
  name_path: Runtime/execute
}
```

The LSP-style `name_path` is especially useful as a stable local semantic identifier:

```text
Runtime/execute
MyClass/my_method
MyClass/my_method[0]
```

---

## 5. Corroborating behavior.2flow

The same semantic model can be observed at progressively smaller scales.

```text
behavior.2flow
  -> Agent / Intent level

action.2flow
  -> Action level

lsp_semantic.2flow
  -> symbol level

semantic_next.2flow
  -> token / AST-slot level
```

Example:

```text
behavior.2flow

[OrderAgent.ReceiveOrder]
  ->
[Order.Validate]
  ->
[Payment.Authorize]
  ->
[Order.Confirm]
```

Architectural binding:

```text
Agent.Order
  -> Intent.ReceiveOrder
  -> Action.Order.Validate
  -> Action.Payment.Authorize
  -> Action.Order.Confirm
```

Structural implementation:

```text
[Function OrderAgent.receiveOrder]
  -> calls [Function validateOrder]
  -> calls [Function authorizePayment]
  -> calls [Function confirmOrder]
```

Observed execution:

```text
Order.Validate.Ok
  ->
Payment.Authorize.Ok
  ->
Order.Confirm.Ok
```

The Proof layer can compare:

```text
Declared Semantics
  ∩
Architectural Semantics
  ∩
LSP Structural Semantics
  ∩
Observed Runtime Semantics
```

This transforms "the code appears to implement the behavior" into explicit semantic evidence.

---

## 6. Semantic Next Graph

Semantic tokens can be used to construct a graph of legal or likely next semantic states.

The naive form:

```text
function -> parameter
parameter -> type
type -> operator
```

is useful but insufficient because it loses context.

The correct unit is a conditioned semantic state.

```text
SemanticState {
  semantic_token
  symbol_kind
  type
  ast_role
  ast_parent
  scope
  visibility
  modifiers
  expected_type
  language
  intent_context
}
```

The graph then represents:

```text
SemanticState
  --constraints-->
PossibleNextSemanticState
```

rather than only:

```text
TokenType -> TokenType
```

---

## 7. Constrained code generation

The goal is to reduce free-form code generation.

Traditional generation:

```text
LLM
  -> writes arbitrary source
  -> compiler
  -> error
  -> retry
```

Semantic generation:

```text
LLM Intent
  ->
Grammar constraint
  ->
AST constraint
  ->
LSP semantic constraint
  ->
Type constraint
  ->
Scope constraint
  ->
Semantic Pattern Memory
  ->
small candidate set
  ->
LLM chooses
  ->
compiler/test/runtime proof
```

Conceptually:

```text
Candidates =
  LLM candidates
  ∩ GrammarAllowed
  ∩ SemanticAllowed
  ∩ TypeAllowed
  ∩ ScopeAllowed
  ∩ PolicyAllowed
  ∩ KnownSafePatterns
  - KnownFailurePatterns
```

The LLM should increasingly choose among valid semantic alternatives instead of inventing unrestricted source text.

---

## 8. Example: return expression

Given:

```zig
fn calculate(a: u32, b: u32) u32 {
    return
}
```

The semantic state is:

```text
context: return-expression
expected_type: u32
scope:
  a: u32
  b: u32
```

The Semantic Next Graph may reduce candidates to:

```text
return
  ->
Expression<u32>
  |- variable:a
  |- variable:b
  |- function where returns == u32
  |- number compatible_with u32
  |- operator expression resolving_to u32
```

and remove semantically impossible classes such as:

```text
namespace
class declaration
string
comment
unrelated struct declaration
```

---

## 9. Example: member access

Given:

```text
user.<cursor>
```

and:

```text
user: User
```

the next semantic candidates should be constrained to members available on `User`:

```text
[variable user: User]
  ->
[
  property<User.*>,
  field<User.*>,
  method<User.*>
]
```

The LSP completion endpoint is especially valuable here because it already encodes scope-aware and type-aware concrete candidates.

SemanticTokenTypes provide the abstract category.

Completion provides the concrete symbols.

Type information filters compatibility.

The LLM selects the candidate that best matches the intended behavior.

---

## 10. Vector representation of semantic code

The semantic flow should also be embedded into vector space.

The preferred representation is not raw source alone.

Instead:

```text
source
  ->
lsp_semantic.2flow
  ->
canonical semantic serialization
  ->
embedding
  ->
vector index
```

This allows structurally similar code to be found even when syntax differs.

Example:

Zig:

```zig
const user = try repo.find(id);
return user.name;
```

Rust:

```rust
let user = repo.find(id)?;
user.name
```

Both may normalize toward:

```text
[variable id]
  ->
[function find]
  ->
[result-like propagate]
  ->
[variable user]
  ->
[property name]
  ->
[return]
```

Therefore a shared semantic vector space can support polyglot similarity search.

---

## 11. Detecting duplicated semantic patterns

If many pieces of code map to similar semantic flows, SemanticCodeAgent can identify implementation variants that should be standardized.

Example:

```text
Validate
  -> Normalize
  -> Persist
  -> Emit
```

If hundreds of implementations produce this same normalized flow, the system can cluster them.

Possible output:

```text
Pattern Cluster
  semantic_similarity: > 0.92
  instances: 417
  variants: 17
  canonical_candidate: variant-03
```

This can reveal candidates for promotion into:

- Primitive Action
- reusable Action
- Skill
- Standard Library operation
- canonical Behavior
- conformance rule

Semantic duplication is therefore detectable independently of textual duplication.

---

## 12. Semantic optimization

Semantically equivalent flows can be compared using execution evidence.

For every observed implementation, store optional metrics such as:

```text
latency
allocations
memory
CPU
cyclomatic complexity
binary size
diagnostics
test result
runtime result
trace count
failure rate
```

Two semantically equivalent flows can then be compared.

```text
Flow A
  semantics = X
  latency = 4.8 ms
  allocations = 12

Flow B
  semantics = X
  latency = 1.7 ms
  allocations = 2
```

The system may infer that A is a candidate for normalization toward B, while preserving correctness through Proof.

Optimization must never be inferred from vector similarity alone.

Semantic equivalence, tests, constraints, architecture invariants and runtime evidence must corroborate the optimization.

---

## 13. Semantic Pattern Memory

SemanticCodeAgent should maintain a reusable memory of code patterns.

```text
SemanticPatternMemory
  |
  |- ValidPatterns
  |   |- canonical
  |   |- optimized
  |   |- proven
  |
  |- FailurePatterns
  |   |- compiler
  |   |- runtime
  |   |- test
  |   |- security
  |   |- conformance
  |
  |- Corrections
      |- before_flow
      |- after_flow
      |- cause
      |- reason
      |- proof
      |- confidence
```

A correction should never be stored only as "bad source -> fixed source".

It should preserve semantic context.

Example:

```text
FailurePattern {
  before_flow,
  diagnostic,
  compiler_error,
  failing_tests,
  type_context,
  scope_context,
  intent_context,
  hypothesis,
  correction_flow,
  proof,
  confidence
}
```

---

## 14. Negative semantic memory

The system should learn from failures so that known-invalid semantic sequences are pruned before being emitted again.

Example failed pattern:

```text
[optional variable]
  ->
[property access]
```

Diagnostic:

```text
optional value must be unwrapped
```

Known correction:

```text
[optional variable]
  ->
[unwrap]
  ->
[property access]
```

Stored knowledge:

```text
KnownFailurePattern #4821

unsafe_when:
  source_type == Optional<T>

before:
  Optional<T> -> property

known_correction:
  Optional<T> -> unwrap<T> -> property

evidence:
  successful_corrections: 184
  regressions: 0

confidence: 0.997
```

When the agent reaches a matching semantic state, the bad branch can be removed before source generation.

---

## 15. Context-sensitive failure rules

Failure memory must not create absolute bans from shallow token sequences.

For example:

```text
variable -> property
```

is valid for many types and invalid for others.

Therefore the identity of a reusable failure pattern should be based on a semantic state fingerprint.

```text
SemanticStateFingerprint =
  normalized_flow
  + type_constraints
  + scope
  + AST parent
  + semantic modifiers
  + expected_type
  + diagnostic context
  + intent context
  + language constraints
```

The stored rule should mean:

```text
known_bad_in_context
```

not:

```text
globally_forbidden_sequence
```

---

## 16. Positive semantic memory

The same mechanism stores proven good patterns.

```text
KnownValidPattern {
  semantic_state,
  next_flow,
  proof_count,
  test_pass_count,
  runtime_success_count,
  performance_profile,
  confidence
}
```

The next-token or next-AST-slot search can therefore prefer patterns with stronger historical evidence.

This turns generation into evidence-guided traversal.

---

## 17. Causal-Topological memory

Vector similarity alone is not enough.

SemanticCodeAgent should combine:

```text
vector similarity
+
topological similarity
+
causal relations
```

The causal structure should preserve:

```text
FailureFlow
  --caused_by-->
SemanticViolation
  --corrected_by-->
CorrectionFlow
  --validated_by-->
Proof
```

This allows the system to distinguish:

- code that merely looks similar;
- code with the same topology;
- code that failed for the same cause;
- code that was corrected by the same semantic transformation.

This is the preferred integration point with Causal-Topological RAG.

---

## 18. Polyglot semantic memory

Language-specific failures should be promotable into language-independent semantic rules.

Example Zig rule:

```text
Optional<T>
  -> unwrap
  -> value access
```

Example Rust rule:

```text
Result<T,E>
  -> propagate | match | unwrap
  -> value access
```

These can be generalized into:

```text
ResultLike<T,E>
  MUST transition through
  Unwrap | Propagate | Match
  before
  ValueAccess
```

The platform can therefore learn concepts at the semantic level instead of memorizing syntax.

---

## 19. Learning from valid code corpora

Semantic Next Graphs can be bootstrapped from existing valid code.

```text
valid repositories
  ->
parser + LSP
  ->
semantic traces
  ->
SemanticState -> NextSemanticState
  ->
Semantic Next Graph
```

Per-language graphs may exist:

```text
zig.semantic.graph
rust.semantic.graph
go.semantic.graph
typescript.semantic.graph
python.semantic.graph
```

but each should map into the canonical AllasCode metamodel.

For Zig, the graph can combine:

```text
Zig grammar
+
ZLS semantics
+
Zig Semantic Dictionary
+
valid source corpus
+
compiler evidence
```

---

## 20. Learning loop

The complete loop is:

```text
write
  ->
validate semantically
  ->
compile
  ->
test
  ->
execute
  ->
observe
  ->
fail or succeed
  ->
understand semantically
  ->
heal if necessary
  ->
prove
  ->
store pattern
  ->
reuse / avoid
```

For failures:

```text
write
  ->
fail
  ->
identify semantic cause
  ->
heal
  ->
prove
  ->
store before/after causal pattern
  ->
prevent equivalent failure
```

The objective is not literal "never fail again".

The objective is:

```text
never knowingly repeat the same proven semantic failure under equivalent context
```

---

## 21. Pattern promotion

A local incident can become platform knowledge.

```text
incident
  ->
correction
  ->
learned pattern
  ->
semantic rule
  ->
Skill
  ->
conformance rule
  ->
Primitive Action / Standard Library candidate
```

Promotion requires evidence thresholds.

Possible signals:

- repeated successful corrections;
- no observed regressions;
- high semantic similarity across occurrences;
- cross-project reuse;
- cross-language generalizability;
- stable Proof results;
- deterministic implementation;
- clear invariant.

This avoids prematurely promoting accidental patterns into architecture.

---

## 22. Code generation as graph traversal

At maturity, SemanticCodeAgent should behave less like a text generator and more like a semantic graph navigator.

```text
Intent
  ->
Behavior target
  ->
current semantic state
  ->
Semantic Next Graph
  ->
allowed transitions
  ->
Known Valid Patterns
  ->
Known Failure vetoes
  ->
LSP completion
  ->
type/scope filtering
  ->
LLM semantic choice
  ->
source materialization
  ->
LSP diagnostics
  ->
compiler
  ->
tests
  ->
Proof
```

The LLM's primary job becomes:

```text
choose which semantically valid transition best satisfies the Intent
```

instead of:

```text
guess arbitrary source tokens
```

---

## 23. Semantic Code Intelligence pipeline

The complete conceptual pipeline is:

```text
Agent Intent
  ->
behavior.2flow
  ->
SemanticCodeAgent
  ->
source/project context
  ->
LSP / compiler semantic extraction
  ->
lsp_semantic.2flow
  ->
SemanticStateFingerprint
  ->
Semantic Next Graph
  ->
Vector Search
  +
Causal-Topological Search
  ->
KnownValidPatterns
  +
KnownFailurePatterns
  +
KnownOptimizedPatterns
  ->
candidate pruning
  ->
LLM selection
  ->
semantic mutation
  ->
LSP validation
  ->
compiler / formatter
  ->
tests / conformance
  ->
runtime execution
  ->
Proof
  ->
Runtime Acceptance
  ->
Semantic Pattern Memory update
```

---

## 24. Semantic blast radius

Before destructive mutation, SemanticCodeAgent should calculate bounded semantic impact.

A blast-radius record should include:

```text
target_symbol
symbol_kind
semantic_token_kind
definition_location
language
provider
references
implementations
callers
callees
type dependencies
public/exported status
affected files
diagnostics before mutation
known patterns
known failures
provider capabilities
confidence
```

This evidence should be given to CodeManagerAgent before CodeHealer executes the change.

---

## 25. Refactoring invariants

Operations such as:

```text
rename
move
delete
signature change
API migration
body replacement
```

must use semantic preflight where supported.

SemanticCodeAgent must not silently downgrade destructive refactoring to blind text replacement when semantic tooling is available but failed.

A provider failure must become explicit evidence.

The Runtime remains the authority for final `Ok` / `Error`.

A successful LSP edit is not a proof of behavioral correctness.

---

## 26. Provider-neutral architecture

Serena is useful as the initial adapter because it already normalizes several language servers and exposes symbol-aware operations.

But the architecture must remain provider-neutral.

```text
SemanticCodeAgent
  ->
SemanticCodeProvider
  |- SerenaProvider
  |- DirectLspProvider
  |- CompilerProvider
  |- TreeSitterProvider
  |- future providers
```

The AllasCode contract must reference semantic capabilities, not Serena-specific APIs.

Canonical Actions may include:

```text
Code.Symbol.Find
Code.Symbol.Definition.Get
Code.Symbol.References.Find
Code.Symbol.CallHierarchy.Get
Code.Symbol.TypeDefinition.Get
Code.Diagnostics.Get
Code.Workspace.Symbols.Find
Code.Symbol.Rename
Code.Symbol.Body.Replace
Code.Symbol.Delete
Code.Refactor.SymbolMove
Code.Refactor.SignatureChange
Code.Api.Migrate
```

---

## 27. Suggested persisted model

A semantic pattern record may look like:

```yaml
pattern_id: zig.optional-access.001
kind: failure_correction

state:
  semantic_token: variable
  semantic_type: Optional<T>
  ast_role: expression
  ast_parent: member_access

before:
  flow:
    - variable<Optional<T>>
    - property<T>

failure:
  class: type_violation
  diagnostic: optional_not_unwrapped

correction:
  flow:
    - variable<Optional<T>>
    - unwrap<T>
    - property<T>

evidence:
  compiler_pass: true
  tests_passed: true
  occurrences: 184
  regressions: 0

confidence: 0.997
```

The vector index should reference this record rather than replace it.

The graph store should hold its causal and topological relationships.

The event store should preserve how the pattern was learned and promoted.

---

## 28. Storage roles

A practical split is:

```text
Event Store
  -> immutable learning/evidence history

Vector Store
  -> semantic similarity retrieval

Causal-Topological Graph
  -> topology, causes, corrections, transitions

Document/Projection Store
  -> current consolidated pattern state

Local Semantic Cache
  -> fast next-state generation constraints
```

No single storage engine should be treated as the sole semantic authority.

---

## 29. Proof-oriented learning

A pattern must accumulate evidence.

Possible lifecycle:

```text
Observed
  ->
Candidate
  ->
Repeated
  ->
Validated
  ->
Proven
  ->
Canonical
```

Failure patterns can follow:

```text
Observed Failure
  ->
Diagnosed
  ->
Corrected
  ->
Correction Proven
  ->
Repeated Correction
  ->
Preventive Rule
```

Confidence should derive from evidence, not from LLM self-assessment.

---

## 30. Core invariants

1. `lsp_semantic.2flow` is generated from semantic evidence, not handwritten as truth.
2. Language-specific provider data must be normalized before becoming an AllasCode semantic artifact.
3. Vector similarity alone must never prove equivalence.
4. A semantic failure rule is context-sensitive by default.
5. Destructive edits require semantic preflight when available.
6. Provider failure must never silently become blind text replacement.
7. Compiler, tests, conformance, architecture invariants and runtime Proof remain final acceptance evidence.
8. The Runtime owns terminal `Ok` / `Error`.
9. Every learned correction must preserve before-state, cause, correction, and proof.
10. Repeated proven patterns may be promoted into Skills, conformance rules, Actions, or Standard Library primitives.
11. Negative memory must prevent known equivalent failures, not ban superficially similar syntax.
12. Cross-language patterns should be generalized only when their semantic invariants truly match.
13. Semantic Next Graph candidates must be conditioned by grammar, AST, type, scope and intent context.
14. SemanticCodeAgent should progressively reduce unconstrained source generation.
15. Every semantic mutation and learning decision should be auditable through Event Sourcing.

---

## 31. Target outcome

The target model is:

```text
CodeHealer does not merely write code.

It navigates a constrained semantic space,
chooses transitions compatible with the declared Behavior,
avoids previously proven failure states,
reuses proven successful patterns,
materializes source in the target language,
and proves the result through the compiler,
tests, runtime evidence and AllasCode acceptance.
```

At that point Semantic-as-Code exists at every scale:

```text
Intent
  ->
Behavior
  ->
Action
  ->
Symbol
  ->
AST slot
  ->
Semantic token
  ->
Runtime event
```

The same declarative model is therefore carried from agent behavior down to the smallest semantically meaningful units of source code.
