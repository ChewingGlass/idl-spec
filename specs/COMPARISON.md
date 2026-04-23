# Unified IDL Spec: Cross-Framework Comparison

A working document to support debate on the unified Solana IDL spec. Compares four sources:

| # | Spec | Source of truth |
|---|------|-----------------|
| 1 | **This repo (v0.1.0)** | [`specs/v0.1.0.md`](./v0.1.0.md) — based on `anchor-lang-idl-spec` 0.1.0 |
| 2 | **Anchor** | [`anchor-lang-idl-spec`](https://docs.rs/anchor-lang-idl-spec/latest/anchor_lang_idl_spec/) — latest published is `0.1.0` (same as #1) |
| 3 | **Quasar** | [`blueshift-gg/quasar` → `schema/src/lib.rs`](https://github.com/blueshift-gg/quasar/blob/master/schema/src/lib.rs) |
| 4 | **Codama** | [`codama-idl/codama` → `packages/node-types/src/*`](https://github.com/codama-idl/codama/tree/main/packages/node-types/src) |

Two structural archetypes are in play:

- **JSON-snapshot-of-a-Rust-struct** (this repo, Anchor, Quasar) — flat tagged/untagged unions, serde-driven. Easy to `serde_json::from_str`. Limited expressivity — each framework has baked-in assumptions about discriminators, encoding, and defaults.
- **Typed node tree** (Codama) — every concept is a `*Node` with a `kind` string tag. Transformations happen via visitors. Higher expressivity at the cost of a larger surface and a learning curve.

The unified spec has to decide which archetype it's closer to, and which framework-specific concepts deserve promotion into the core.

---

## 1. Root object

| Field | This repo / Anchor | Quasar | Codama |
|---|---|---|---|
| `address` / program id | `address: string` on `Idl` | `address: String` on `Idl` | `publicKey: string` on `ProgramNode` |
| program metadata | `metadata: IdlMetadata` | `metadata: IdlMetadata` | Flattened into `ProgramNode` (name, version, docs, origin) |
| instructions | `instructions: IdlInstruction[]` | `instructions: Vec<IdlInstruction>` | `instructions: InstructionNode[]` |
| accounts | `accounts: IdlAccount[]` | `accounts: Vec<IdlAccountDef>` | `accounts: AccountNode[]` |
| events | `events: IdlEvent[]` | `events: Vec<IdlEventDef>` | `events: EventNode[]` |
| errors | `errors: IdlErrorCode[]` | `errors: Vec<IdlError>` | `errors: ErrorNode[]` |
| types | `types: IdlTypeDef[]` | `types: Vec<IdlTypeDef>` | `definedTypes: DefinedTypeNode[]` |
| constants | `constants: IdlConst[]` | — *(no equivalent)* | — *(no equivalent)* |
| PDAs | folded into instruction accounts | folded into instruction accounts | **First-class `pdas: PdaNode[]`** on `ProgramNode` |
| multi-program | — | — | **`RootNode` wraps `program` + `additionalPrograms[]`** |
| top-level `docs` | `docs: string[]` | — | `docs` on `ProgramNode` |

**Debate-worthy divergences:**

1. **Constants.** Only this repo/Anchor has `constants`. Quasar and Codama have no equivalent. Are they worth keeping? They're under-specified (value is a string — no rules for interpreting bigint, array, pubkey literals).
2. **First-class PDAs.** Codama extracts PDAs into a top-level `pdas` array with reusable `PdaNode`s; instruction accounts then *reference* a PDA by name. The other two specs re-inline the PDA definition every time an account uses it. Codama's model dedupes and lets shared PDAs (e.g. `global_config`) have one source of truth.
3. **Multi-program roots.** Codama's `RootNode` supports `additionalPrograms[]` — useful when a program's IDL needs to reference accounts/types from a *different* program (e.g. a wrapper around Token). The flat specs can't express this.
4. **Provenance.** Codama's `ProgramNode.origin: 'anchor' | 'shank'` records where the IDL came from. Useful for tools that need to apply framework-specific behavior.

---

## 2. Metadata

| Field | This repo / Anchor | Quasar | Codama |
|---|---|---|---|
| `name` | ✅ | ✅ | ✅ (on `ProgramNode`) |
| `version` | ✅ | ✅ | ✅ |
| `spec` | ✅ | ✅ | No equivalent (version is baked into each node type) |
| `description` | ✅ | ❌ | Use `docs` |
| `repository` | ✅ | ❌ | ❌ |
| `dependencies[]` | ✅ (name, version) | ❌ | Use `additionalPrograms` |
| `contact` | ✅ | ❌ | ❌ |
| `deployments` | ✅ (mainnet/testnet/devnet/localnet) | ❌ | ❌ |
| `origin` (source framework) | ❌ | ❌ | ✅ `'anchor' \| 'shank'` |

Quasar is intentionally minimal here. This repo inherits Anchor's full surface.

**Debate:**
- `deployments` is a half-measure: four fixed clusters, hardcoded keys. Real deployments are branchier (custom RPCs, staging clusters, migration windows). Does this belong in the IDL at all, or in a sidecar manifest?
- `contact` — a single string isn't great. Most specs that try this converge on a structured contact block (email, security.txt URL, PGP).
- `origin` is cheap to add and genuinely useful for tooling.

---

## 3. Instructions

| Field | This repo / Anchor | Quasar | Codama |
|---|---|---|---|
| `name` | camelCase | (same) | `CamelCaseString` |
| `docs` | `string[]` | ❌ no docs field | `Docs` |
| `discriminator` | `number[]` (8 bytes, SHA-256 derived) | `Vec<u8>` (1 byte, developer-assigned) | **`discriminators?: DiscriminatorNode[]`** — composable (constant, field, size-based) |
| `accounts` | `IdlInstructionAccountItem[]` (supports nested `Composite`) | `Vec<IdlAccountItem>` (flat only) | `accounts: InstructionAccountNode[]` |
| `args` | `IdlField[]` | `Vec<IdlField>` | `arguments: InstructionArgumentNode[]` |
| `returns` | `IdlType` (optional) | ❌ | — (modeled via `InstructionByteDeltaNode`?) |
| remaining accounts | ❌ (implicit) | **`hasRemaining: bool`** | **`remainingAccounts: InstructionRemainingAccountsNode[]`** |
| args encoding | Borsh (implied) | **`argsLayout: "fixed" \| "compact"`** | Fully explicit per-argument via type nodes |
| extra arguments | ❌ | ❌ | `extraArguments?` (added during resolve, not on wire) |
| byte deltas | ❌ | ❌ | `byteDeltas?: InstructionByteDeltaNode[]` (models account size growth/shrink) |
| sub-instructions | ❌ | ❌ | `subInstructions?` (inline CPIs for client helpers) |
| optional-account strategy | omission implicit per-account | — | **`optionalAccountStrategy: 'omitted' \| 'programId'`** |
| status | ❌ | ❌ | `status?: InstructionStatusNode` |

**Debate-worthy:**

1. **Discriminator model.** This is the single largest incompatibility.
   - This repo / Anchor: `number[]`, conventionally 8 bytes, derived from `sha256("global:<name>")` for instructions, `sha256("account:<Name>")` for accounts.
   - Quasar: `number[]`, typically 1 byte, developer-assigned.
   - Codama: a *list* of discriminator nodes that can include `ConstantDiscriminatorNode`, `FieldDiscriminatorNode` (dispatch on a struct field, like an enum tag), or `SizeDiscriminatorNode` (dispatch on account size). Supports multi-layer dispatch.

   A unified spec could:
   - (a) Keep the current flat `number[]` and document that length/derivation is framework-specific — the current choice. Works for simple cases; fails for anything doing field-discriminated dispatch (e.g. some native programs).
   - (b) Adopt a discriminator object (`{kind: "constant", bytes: [..]}` vs `{kind: "field", path: ".."}` vs `{kind: "size", value: n}`) — Codama-style but trimmed.
   - (c) Keep `number[]` as the common path and add an optional `discriminatorKind` to signal field/size dispatch when needed.

2. **Composite (nested) accounts.** This repo / Anchor support `IdlInstructionAccounts` — a named group of accounts. Quasar does not. Codama does not natively; it flattens. Is composite worth keeping given neither of the other two frameworks supports it?

3. **Remaining accounts.** Three different models:
   - This repo / Anchor: implicit — clients just append accounts past the declared set.
   - Quasar: a boolean flag `hasRemaining`. No type info.
   - Codama: an array describing shape, writability, signer-ness, and how the client should source them (`ArgumentValueNode` or `ResolverValueNode`).

   Codama's is clearly richer. A unified spec could at minimum promote Quasar's boolean (low cost, high value for decoders) and make structured remaining-accounts optional.

4. **Args layout.** Quasar's `argsLayout: "fixed" | "compact"` is unique. `"fixed"` is normal Borsh-like packing; `"compact"` refers to Solana's length-prefix compression used by native programs. Anchor's spec implicitly assumes Borsh always. If we want to describe native programs, this has to be expressible.

5. **Instruction-argument defaults.** Codama is alone in letting an instruction argument carry a `defaultValue` plus a `defaultValueStrategy: 'omitted' | 'optional'` — e.g. "if the client doesn't pass `bump`, derive it from the PDA." This is the difference between an IDL that describes the wire and an IDL that describes the *client API*.

---

## 4. Instruction accounts

| Field | This repo / Anchor | Quasar | Codama |
|---|---|---|---|
| `name` | camelCase | (same) | `CamelCaseString` |
| `docs` | ✅ | ❌ | ✅ |
| `writable` | `bool` (default false) | `bool` (default false) | `isWritable: boolean` (required) |
| `signer` | `bool` (default false) | `bool` (default false) | **`isSigner: boolean \| 'either'`** |
| `optional` | `bool` (default false) | ❌ | `isOptional?: boolean` |
| `address` | `string` (fixed/known) | `string` (fixed/known) | Modeled via `defaultValue: { kind: 'programIdValueNode' \| ... }` |
| `pda` | inline `IdlPda` | inline `IdlPda` | **`defaultValue: PdaValueNode` referencing a top-level `PdaNode`** |
| `relations` | `string[]` — names of related accounts | ❌ | Modeled via `defaultValue: AccountValueNode` / resolvers |
| `defaultValue` | ❌ | ❌ | **Rich union** — can reference payer, authority, PDA, another account's field, a resolver function, a conditional, etc. |

**Debate:**

- **`signer: 'either'`** (Codama only). Means the account *may* be a signer but the program accepts either. Useful for permissive multisig-friendly programs. Worth supporting?
- **`relations`** is Anchor-specific — it refers to the `#[account(has_one = xxx)]` Anchor constraint. Other frameworks don't have an equivalent concept. If kept, we'd be baking Anchor's macro model into the unified spec.
- **`relations` may be going away upstream.** Anchor issue [#4421](https://github.com/solana-foundation/anchor/issues/4421) proposes replacing `has_one` with a generalized `address =` constraint, flipping the relationship:
  ```rust
  // today
  struct Accounts {
    #[account(has_one = key)] data: Account<MyAccount>,
    key: Signer,
  }
  // proposed
  struct Accounts {
    data: Account<MyAccount>,
    #[account(address = data.key)] key: Signer,
  }
  ```
  In IDL terms this means: generalize the existing `address: string` field (currently only a literal pubkey, used for well-known programs) into an expression — literal pubkey **or** dot-delimited path to another account's field. `relations` then disappears entirely, folded into `address`. A unified spec designed today should anticipate this: either ship `address` as an expression from the start, or reserve syntactic space for it.
- **Default values** are the most important difference. The JSON-snapshot specs (Anchor, Quasar, this repo) describe a program's *on-chain interface*. Codama describes *both* the interface and the client-resolution behavior. A unified spec that stays in the first camp is simpler; one that crosses into the second replaces a lot of per-client resolver code with declarative IDL.

---

## 5. Type system

This is where the specs diverge most.

### Primitives

| Spec | Primitives |
|---|---|
| This repo / Anchor | `bool, u8/i8..u256/i256, f32, f64, bytes, string, pubkey` — as bare strings |
| Quasar | `IdlType::Primitive(String)` — any string; no fixed vocabulary |
| Codama | `BooleanTypeNode`, `NumberTypeNode` (with `format: u8..i128/f32/f64/shortU16` + **`endian: 'be' \| 'le'`**), `PublicKeyTypeNode`, `StringTypeNode` (with **`encoding`**), `BytesTypeNode`, `AmountTypeNode`, `SolAmountTypeNode`, `DateTimeTypeNode` |

**Debate:**

- **Endian.** Codama is the only spec that surfaces byte order. Solana uses LE everywhere on-chain today, so the JSON-snapshot specs bake that in. The moment you model a CPI to something non-Solana, or a program that pretends an integer is BE for compat, you need Codama's level of explicitness.
- **String encoding.** Codama separates the *logical* type (string) from its *bytes encoding* (utf8, base58, base64). Useful when an on-chain bytes field is conventionally base58-rendered for humans but stored raw.
- **Semantic numeric types.** Codama has `AmountTypeNode` (token amounts with decimals) and `SolAmountTypeNode` (lamports) and `DateTimeTypeNode` (timestamps). These don't change wire layout but improve client generation. Whether they belong in a "describes the wire" spec is a judgment call.
- **`shortU16`.** Solana's native compact-u16 encoding for account count prefixes. Only Codama has a dedicated primitive.

### Containers

| Container | This repo / Anchor | Quasar | Codama |
|---|---|---|---|
| Option | `{option: T}` | `{option: T}` | `OptionTypeNode`, **plus `ZeroableOptionTypeNode`** (sentinel-valued), **`RemainderOptionTypeNode`** (presence implied by trailing bytes) |
| Vec | `{vec: T}` | `{vec: {items: T, maxLength, prefixBytes}}` | `ArrayTypeNode` wrapped in a `CountNode` (`PrefixedCountNode`, `FixedCountNode`, `RemainderCountNode`) |
| Array (fixed) | `{array: [T, N]}` where `N` is a number or `{generic: "N"}` | — (not in schema!) | `ArrayTypeNode` + `FixedCountNode` |
| String (dynamic) | `"string"` (primitive) | **`{string: {maxLength, prefixBytes: 1\|2\|4\|8}}`** | `StringTypeNode` wrapped in size/count nodes |
| Map | ❌ | ❌ | `MapTypeNode` |
| Set | ❌ | ❌ | `SetTypeNode` |
| Tuple | via `IdlDefinedFields::Tuple` | ❌ | `TupleTypeNode` |
| Struct | `IdlTypeDefTy::Struct` | `IdlTypeDefType { kind: "struct", fields }` | `StructTypeNode` with `StructFieldTypeNode[]` |
| Enum | `IdlTypeDefTy::Enum` | ❌ | `EnumTypeNode` with explicit `size: NumberTypeNode` + three variant shapes |
| Type alias | `IdlTypeDefTy::Type { alias }` | ❌ | `DefinedTypeNode` wrapping any `TypeNode` |
| Generics | `{generic: "T"}`, `{defined: {name, generics}}` | ❌ | — (Codama inlines after Anchor import) |
| Defined (reference) | `{defined: {name, generics}}` | `{defined: string}` | `DefinedTypeLinkNode` (+ optional `ProgramLinkNode` for cross-program) |

**Debate:**

1. **Size prefixes.** Quasar's `prefixBytes: 1|2|4|8` on strings and vecs is the *wire* detail Anchor's spec omits. Anchor assumes u32-LE prefixes (Borsh). Quasar explicitly lets you use u8, u16, or u64. Codama goes further and factors the prefix out as a `CountNode`. For describing native/non-Anchor programs, explicit prefix size is probably a must.

2. **`maxLength`.** Quasar also records an explicit `maxLength` on dynamic strings/vecs — useful for zero-copy account sizing. Codama models fixed sizes via `FixedSizeTypeNode` / `FixedCountNode`. Anchor has no way to express "this string is at most 32 bytes."

3. **Missing types in Quasar.** No `enum`, no `array` (fixed-size), no `bytes`, no generics, no tuple, no aliases — just struct type defs. Quasar deliberately restricts what programs can do in the name of zero-copy safety. The unified spec presumably needs the full set; Quasar will either ignore fields it doesn't understand or error.

4. **Nothing expressive.** Codama has `FixedSizeTypeNode`, `HiddenPrefixTypeNode`, `HiddenSuffixTypeNode`, `PostOffsetTypeNode`, `PreOffsetTypeNode`, `SentinelTypeNode`, `NestedTypeNode`. These describe exotic on-chain layouts (metadata blobs, SPL programs with padding, version prefixes). None of these are in Anchor or Quasar. If the goal is to describe *any* Solana program rather than just Anchor-likes, these matter.

5. **Array vs Vec vs CountNode.** Codama's uniform `ArrayTypeNode + CountNode` is more consistent than the Anchor/this-repo split between `vec` and `array`. But it's a bigger cognitive load for people reading the JSON.

---

## 6. PDAs

| Field | This repo / Anchor | Quasar | Codama |
|---|---|---|---|
| `seeds` | `IdlSeed[]` | `Vec<IdlSeed>` | `seeds: PdaSeedNode[]` on a top-level `PdaNode` |
| `program` (cross-program PDA) | `IdlSeed` (optional) | ❌ | `programId?: string` on `PdaNode` |
| Seed: `const` | `{kind: "const", value: number[]}` | `{kind: "const", value: Vec<u8>}` | `ConstantPdaSeedNode` (stores a `type` + `value`, so constants are typed) |
| Seed: `arg` | `{kind: "arg", path: string}` | `{kind: "arg", path: string}` | Unified under `VariablePdaSeedNode { name, type }` — the actual binding happens in `InstructionAccount.defaultValue` |
| Seed: `account` | `{kind: "account", path: string, account?: string}` | `{kind: "account", path: string}` *(no `account` type name)* | Unified under `VariablePdaSeedNode` |
| Dedup / reuse | inline per-instruction | inline per-instruction | **PDAs are top-level, referenced by `PdaValueNode { pdaLink: ... }`** |

**Debate:**

- Codama's **variable seed** model (declare a named slot with a type, bind it at call-site) is the cleanest. The Anchor/Quasar/this-repo mix of `arg`-path vs `account`-path is ad-hoc.
- This repo's `IdlSeedAccount.account` (the account *type* name) is useful for clients needing to resolve a seed by deserializing another account first. Quasar drops this; Codama encodes it via linking.
- **Cross-program PDAs.** Only Anchor/this-repo (`program: IdlSeed`) and Codama (`programId: string` on `PdaNode`) support them. Quasar doesn't.

---

## 7. Accounts & events

| Field | This repo / Anchor | Quasar | Codama |
|---|---|---|---|
| Account: `name` | ✅ | ✅ | ✅ |
| Account: `discriminator` | 8 bytes | variable bytes | `discriminators?: DiscriminatorNode[]` (composable) |
| Account: `data` / layout | Resolved by name from `types[]` | Resolved by name from `types[]` | **Inline: `data: StructTypeNode`** — no separate `types` lookup needed |
| Account: `size` | ❌ | ❌ | **`size?: number \| null`** (explicit fixed size) |
| Account: `pda` | ❌ (inferred from instructions) | ❌ | **`pda?: PdaLinkNode`** (which PDA canonically produces this account) |
| Event: layout | Same as accounts — name maps to `types[]` | Same | Inline `data` on `EventNode` |

**Debate:**

- **Account → type separation.** This repo/Anchor/Quasar separate the account *identity* (name, discriminator) from the account *layout* (stored in `types[]` under the same name). Codama inlines the layout. Pros of inlining: easier for readers, no implicit name-matching contract. Pros of separation: accounts can share layouts, and layouts can be reused via `{defined: "X"}`.
- **Explicit account size** (Codama only) matters for zero-copy accounts, rent calculation, and client allocation. Quasar cares about this (its entire frame is zero-copy) but relies on `maxLength` on type fields instead.
- **Canonical PDA on an account** (Codama only) — bridges the "this account type is always at PDA X" relationship that Anchor encodes implicitly via the `#[account(seeds = [...])]` macro.

---

## 8. Errors

| Field | This repo / Anchor | Quasar | Codama |
|---|---|---|---|
| `code` | `u32` | `u32` | `number` |
| `name` | ✅ | ✅ | ✅ |
| `msg` / `message` | `msg: string?` | `msg: Option<String>` | `message: string` (**required**) |
| `docs` | ✅ | ❌ | ✅ |

Near-identical. One tiny wire divergence: `msg` vs `message`. Codama requires the message; the others treat it as optional.

---

## 9. Serialization, repr, generics (Anchor-specific surface)

| Concept | This repo / Anchor | Quasar | Codama |
|---|---|---|---|
| `IdlSerialization` | `"borsh" \| "bytemuck" \| "bytemuckunsafe" \| {custom}` | ❌ (borsh implied) | Modeled per-field via type nodes (zero-copy expressed via `FixedSizeTypeNode`) |
| `IdlRepr` (`rust`/`c`/`transparent`, packed, align) | ✅ | ❌ | ❌ (explicit layout via type node wrappers) |
| Generics on type defs | `IdlTypeDefGeneric` (`type` or `const`) | ❌ | ❌ (monomorphized at import) |

**Debate:**

- Anchor's `serialization` and `repr` fields leak Rust implementation details into the IDL. Codama's position: if the user cares about layout, they describe the layout explicitly with type nodes. Quasar: irrelevant (always borsh).
- **Generics** are almost certainly an Anchor-only concern; Codama and Quasar tools just don't emit them. Keep or drop?

---

## 10. Unique-to-Codama concepts (not matched elsewhere)

These are the big conceptual additions in Codama. A unified spec has to decide whether each is in-scope.

- **Link nodes** (`AccountLinkNode`, `DefinedTypeLinkNode`, `PdaLinkNode`, `ProgramLinkNode`, etc.) — typed references between nodes, including across programs.
- **Contextual values** — `AccountBumpValueNode`, `AccountValueNode`, `ArgumentValueNode`, `ConditionalValueNode`, `IdentityValueNode`, `PayerValueNode`, `PdaSeedValueNode`, `PdaValueNode`, `ProgramIdValueNode`, `ResolverValueNode`. These let an IDL say things like "this account defaults to the payer" or "this PDA's bump defaults to the canonical bump for the referenced PDA."
- **Resolvers** — `ResolverValueNode` points to a named function the client should invoke to populate a value. The IDL doesn't implement it; it just declares the contract.
- **Byte deltas** — `InstructionByteDeltaNode` tracks account size growth/shrink caused by the instruction. Useful for clients that need to fund realloc.
- **Sub-instructions** — declarative inner instructions (CPIs) a client can batch.
- **Optional-account strategy** — `'omitted' \| 'programId'` tells the client how to *encode* an absent account (skip it entirely vs pass the program id).
- **Hidden prefixes/suffixes, pre/post offsets, sentinels** — wire-layout directives for exotic serialization (Metaplex TokenMetadata, padding, header bytes).
- **Multi-program roots** — one IDL document describes multiple programs that work together.

---

## 11. Unique-to-Quasar concepts

- **`hasRemaining`** boolean on instructions.
- **`argsLayout: "fixed" | "compact"`** — distinguishes Borsh-style packing from Solana's compact encoding.
- **`maxLength` + `prefixBytes`** on strings and vecs — explicit wire sizing.
- **`crate_name`** in metadata (skipped during JSON emission but used internally for codegen).
- **`known_address_for_type`** map — built-in resolution of `SystemProgram`, `Program<Token>`, `Sysvar<Rent>`, etc. into canonical addresses at codegen time. Not an IDL feature per se; more a sign Quasar expects programs to lean on well-known accounts.

---

## 12. Unique-to-Anchor / this-repo concepts

- **`relations`** on instruction accounts — Anchor's `has_one` constraint leaked into the IDL.
- **`returns`** on instructions — declares return-data type.
- **`constants`** block.
- **`IdlRepr`** (`rust`/`c`/`transparent` + `packed` + `align`).
- **`IdlSerialization`** tag (`borsh`/`bytemuck`/`bytemuckunsafe`/`custom`).
- **Composite accounts** (`IdlInstructionAccounts` — nested account groups with a group name).
- **Generics on type defs** (`IdlTypeDefGeneric` with `type` and `const` kinds).
- **Full `deployments` block** (mainnet/testnet/devnet/localnet — all four always present when emitted).
- **`IdlMetadata.repository`, `.description`, `.dependencies[]`, `.contact`** — a grab-bag of optional provenance fields.

---

## 13. Speculative / ambitious additions

These are ideas floated that go beyond describing what programs *are* today, into reshaping how clients build transactions. Flagged separately because they carry real tradeoffs (spec complexity, compliance burden, expressivity escalation) that the items in section 14 do not.

### 13.1 Pre/post instruction markers

**Proposal (Dean):** Let an IDL declare that an instruction needs a setup or teardown step the client should splat into the transaction automatically.

> If we see an ATA with "init-if-needed" in the account struct, rather than requiring the user to put the ATA program in there, we just emit a marker to say "run `createATAIdempotent` before this instruction" and the client just picks it up. This saves a ~1000 CU CPI penalty.

**What gets better:**
- Smaller program binaries (no embedded ATA-creation CPI).
- Lower CU cost per transaction (no CPI overhead for common setup).
- Cleaner program code (no `init-if-needed` branch inside the handler).
- Client builders become smarter automatically — users don't need to remember to prepend setup ixs.

**What gets worse:**
- **Compliance burden skyrockets.** A conforming client now has to understand and execute a catalog of pre/post ix templates. Any framework or language that wants to consume the IDL has to ship that machinery or refuse to generate working clients. Minimal consumers (e.g. decoders, explorers, indexers) don't need this and pay no cost, but *transaction builders* can't opt out.
- **The catalog has to live somewhere.** Either the spec enumerates known setup ops (ATA create, account allocate, rent top-up, SPL approve) — which means the spec now owns a registry — or it lets IDLs declare arbitrary CPIs and the client decodes those too. The first option is narrow-but-finite; the second…
- **…accidentally makes IDLs Turing-complete.** The moment you allow "run this other instruction first, with parameters derived from this account's field, conditionally on this predicate" you've built a DSL for transaction construction. Codama's `ConditionalValueNode` + `ResolverValueNode` already lean in this direction and stop short of evaluation; this proposal would cross the line.
- **Versioning gets sharp.** If a setup marker's semantics change (e.g. "createATAIdempotent" grows a parameter), existing IDLs silently produce different transactions.

**Design sketches, in order of ambition:**

1. **Fixed registry, zero params.** Instructions carry an optional `preInstructions: string[]` / `postInstructions: string[]` listing well-known setup names (`createAtaIdempotent`, `createAccount`, etc.). The spec enumerates them; clients implement each explicitly. No expressions, no arithmetic. Closest to safe.
2. **Parameterized registry.** Same list but each entry has structured params referencing accounts/args in the parent instruction (`{kind: "createAtaIdempotent", mint: "mint", owner: "authority"}`). Still bounded; still spec-owned.
3. **Arbitrary CPI template.** A pre-instruction is itself a full instruction reference (program id + discriminator + account bindings + arg bindings). Maximum expressivity. This is the option that risks Turing-completeness in practice.

**Debate:** Is this in scope at all for v1? Separate RFC? Postponed indefinitely with reserved field names so future versions don't break?

### 13.2 Address-expression constraint (tracked: Anchor [#4421](https://github.com/solana-foundation/anchor/issues/4421))

Not as speculative as §13.1 — this one is an active Anchor proposal — but it does require a spec change. Detailed under §4 above. Summary: generalize `address: string` so it accepts either a literal pubkey or a path like `"data.key"`. Folds `relations` into `address`. See §4 and debate topic 8 below.

---

## 14. Proposed debate topics

Ordered roughly by impact:

1. **Discriminator shape.** Flat `number[]` (today) vs tagged union (Codama-style). If we stay flat, do we add `discriminatorKind` or `discriminatorLength` metadata for non-Anchor programs?
2. **Explicit wire layout** for dynamic containers — Quasar's `prefixBytes` / `maxLength` or Codama's `CountNode` wrappers. Without one of these, we can't describe non-Borsh or zero-copy programs.
3. **Endianness and number format.** Promote Codama's `endian` + `format` onto `NumberTypeNode`-style type expressions, or keep bare primitive strings?
4. **Default values on accounts/args.** Does the unified spec describe *only the wire* (keep it flat), or also *client-resolution behavior* (promote Codama's contextual values)?
5. **First-class PDAs.** Top-level `pdas: PdaNode[]` vs inline-per-instruction.
6. **Multi-program IDLs.** Do we support `additionalPrograms` for programs that share types with their peers (SPL ecosystem, multi-program DAOs)?
7. **Composite accounts.** Keep (Anchor has them) or drop (Quasar and Codama don't)?
8. **`address` as expression; drop `relations`.** Per Anchor [#4421](https://github.com/solana-foundation/anchor/issues/4421), generalize `address` to `pubkey | path` and remove `relations`. Alternative: keep both during a transition window.
9. **Generics.** Keep or drop? Codama monomorphizes on import.
10. **Serialization & repr hints.** Are they IDL concerns or codegen concerns?
11. **Semantic type nodes** (`AmountTypeNode`, `SolAmountTypeNode`, `DateTimeTypeNode`). Do we promote any?
12. **Provenance / origin field** on the IDL root — cheap Codama import, worth borrowing.
13. **`remaining accounts`.** Minimum: adopt Quasar's boolean. Maximum: adopt Codama's typed list.
14. **Constants block.** Keep (and tighten value format), or drop?
15. **Instruction `returns`.** Keep?
16. **`msg` vs `message` on errors.** Pick one; Codama requires the field, others don't.
17. **Key renaming.** Minor but worth settling: `isWritable` (Codama) vs `writable` (everyone else); `isSigner` (Codama) vs `signer`; `arguments` (Codama) vs `args`; `publicKey` (Codama) vs `address` (everyone else); `definedTypes` (Codama) vs `types` (everyone else).
18. **Pre/post instruction markers (§13.1).** In scope for v1? Separate future RFC? Reserved namespace only? Or explicitly out of scope forever because it makes IDLs Turing-complete?
19. **Per-signer signing templates.** Should each signing account carry its own human-readable "what you're authorizing" template, folded directly into the `messageSigner` field so presence-of-the-field declares the signer role? A swap's maker and taker legitimately see different things — instruction-level templates force an artificial single-perspective view.
20. **Message signers (signed-intent authorization).** Can an account authorize an instruction by ed25519-signing its own rendered template instead of signing the Solana transaction? If so: how to pin cross-implementation deterministic rendering; whether the signed content needs a domain preamble (EIP-712-style) for replay protection; how to express signature expiry.

---

## 15. Opinions (Noah)

Answers to the debate topics in §14. These are a working set of positions to drive the RFC discussion — not final decisions.

### Cross-cutting principles

Several themes kept recurring. Pulled out here so they don't get lost in the per-topic answers:

- **Optionality as a first-class idea.** Expressive features should be available but rarely *required*. Producers may emit; consumers may ignore. A minimal conforming IDL stays small; a rich producer can add detail without breaking anything downstream. Applies to contextual values (#4), semantic types (#11), remaining-accounts shape (#13), generics (#9), and Codama-style sugar in general.
- **Tagged unions, never `string | object` mixes.** Every union should be a tagged object (`{ type: "constant", value: ... }` vs `{ type: "field", path: ... }`), not a fused primitive-or-object form. Covers #1, #8, and by extension the legacy `IdlArrayLen` (`number | { generic: string }`) which should be rewritten as `{ type: "fixed", value: N }` / `{ type: "generic", name: "N" }` in v1.
- **Push layout into the type system.** If a field has a wire layout quirk (size prefix, endianness, fixed size, sentinel, padding), describe it via a type-node wrapper — not a sidecar `serialization` / `repr` hint. Covers #2, #3, #10, #15. The IDL should fully describe the serialization without needing to reference Rust-flavored type names or framework-specific assumptions.
- **Inline at the point of use.** If a value matters (a seed, a discriminator, a default), describe it where it's used — not in a separate `constants` block. Covers #14, and reinforces #5 (first-class PDAs are the one exception where extraction > inlining because reuse is common).
- **Anchor-compat where cheap; diverge where the ecosystem has already moved on.** Keep majority naming conventions (#17) and generics (#9) to minimize migration friction. But: adopt tagged discriminators (#1), drop serialization/repr hints (#10), and accept that zero-copy programs need first-class wire-layout expressivity (#2).

### Answers

| # | Topic | Position |
|---|---|---|
| 1 | Discriminator shape | **Codama-style tagged union.** `{ type: "constant", bytes: [...] }` \| `{ type: "field", path: "..." }` \| `{ type: "size", value: n }`. If Codama saw the need, there are real programs with exotic dispatch that need the expressivity. |
| 2 | Explicit wire layout for dynamic containers | **Codama's `CountNode` wrappers.** The ecosystem is moving toward zero-copy; Quasar already requires explicit prefix widths. We need something expressive enough to describe any serialization — not just fixed Borsh. Whether Codama's exact shape is the right one is a v1 design question, but the axis of expressivity is non-negotiable. |
| 3 | Endianness and number format | **Hybrid.** Keep bare strings (`"u64"`, `"i32"`) as shorthand for little-endian. Allow an explicit object form (e.g. `{ number: "u64", endian: "be" }`) when non-LE is needed. Add `shortU16` as a bare primitive. |
| 4 | Default values on accounts/args | **Optional descriptive layer.** Contextual-value nodes (payer, identity, PDA, resolver, conditional) are permitted but never required. An IDL that describes only the wire is still conforming. Producers that want richer client generation may opt in. |
| 5 | First-class PDAs | **Top-level `pdas: PdaNode[]`**, instructions reference by name. Additionally: optional canonical PDA ↔ account-type link (mirrors Codama's `AccountNode.pda?: PdaLinkNode`) — 99.9% of PDAs map 1:1 to an account type, so surfacing that relationship is nearly free. |
| 6 | Multi-program IDLs | **Adopt `additionalPrograms[]`.** A program document can describe itself plus peer programs whose types/accounts it references. |
| 7 | Composite accounts | **Flatten in the IDL, preserve grouping metadata.** Accounts appear as a flat list, but each carries its composite-group membership. Dot-path accessors (`token_accounts.mint`) remain valid for UX/docs. This matches the Anchor developer experience without forcing consumers to parse a tree. |
| 8 | `address` as expression; drop `relations` | **Adopt.** `address` becomes a tagged union: `{ type: "constant", value: "<pubkey>" }` or `{ type: "field", path: "data.key" }`. `relations` disappears. (See also: cross-cutting principle on tagged unions.) |
| 9 | Generics | **Keep as optional / Anchor-compat.** Producers that use generics can emit them; consumers may treat generic type defs as opaque. No-cost inclusion for Anchor; no burden for Quasar/Codama/native. |
| 10 | Serialization & repr hints | **Drop both.** The IDL should fully describe serialization via the type system. No `borsh` / `bytemuck` / `bytemuckunsafe` / `custom` tags; no `rust` / `c` / `transparent` / `packed` / `align` hints. If a layout needs describing, describe it. |
| 11 | Semantic type nodes (`AmountTypeNode`, etc.) | **Supported but optional.** Producers may emit; consumers may ignore. Some client generators will use them for nicer UX. Keep the registry open — future additions don't require a spec bump. |
| 12 | Provenance / origin field | **Structured `metadata.generator: { name, version }`.** Pins the exact generator version — useful for bug reproduction and for tools that want to apply framework-specific defaults. |
| 13 | Remaining accounts | **Hybrid: structured when present, not required.** Presence alone signals the instruction accepts a remaining-accounts tail. When structured (name, isSigner, isWritable, docs, value source), it becomes client-generation sugar. Codama's typed list is nice but can't always be produced. |
| 14 | Constants block | **Drop.** Any constant that matters — a PDA seed, a discriminator, a default value, a magic number — can and should be described inline where it's used. A free-floating `constants[]` array with stringly-typed values adds surface without solving a real problem. |
| 15 | Instruction `returns` | **Keep as `returns: IdlType`.** Because the type system is expressive enough after #2/#3/#10 to describe any return shape, the field is both cheap and useful. Programs that don't return data omit it. |
| 16 | `msg` vs `message` on errors | **`message`, optional.** Aligns with Codama and the broader JS/TS ecosystem. Not required — programs may emit error codes without human-readable messages. |
| 17 | Key renaming | **Keep majority convention.** `writable`, `signer`, `args`, `address`, `types`. Don't adopt Codama's `is`-prefix / `definedTypes` / `publicKey` renames. Migration cost of renaming outweighs the stylistic upside. |
| 18 | Pre/post instruction markers | **Parameterized registry in v1.** Spec enumerates a fixed set of setup ops (e.g. `createAtaIdempotent`, `createAccount`). Each entry carries structured params that reference accounts/args in the parent instruction. Bounded enough to avoid Turing-completeness; expressive enough to be actually useful. Arbitrary-CPI templates (§13.1 option 3) are explicitly out of scope. |
| 19 | Per-signer signing templates | **The template lives on the signing account, folded into `messageSigner` itself.** An account that authorizes via message signature carries `messageSigner: { template, placeholders }` — presence of the field declares the role; there is no separate boolean + summary. A swap's maker has `messageSigner.template: "Sell X for Y to {taker}"`; the taker has `messageSigner.template: "Buy X for Y from {maker}"`. Placeholders reuse the §11 path-expression and semantic-format machinery — one mechanism, shared across `address` (#8), defaults (#4), semantic types (#11), and message-signer rendering (#20). English-only; localization is a client concern. |
| 20 | Message signers (signed intent) | **Supported; the `messageSigner.template` IS the signing contract.** `IdlInstructionAccount.messageSigner` is an object (or absent) — not a boolean. Presence declares the account as a message signer; absence means no. Mutually exclusive with `signer: true`. Each message signer ed25519-signs the exact rendered bytes of **its own** template; the program reconstructs + verifies via the ed25519 sigverify precompile (one precompile call per message signer). Taking inspiration from EIP-712, signed bytes include a fixed domain preamble (`"Solana Signed Intent v1\nProgram: <addr>\nVersion: <ver>\nExpires: <iso8601>\n\n..."`) to prevent cross-program, cross-version, and post-expiry replay — the preamble is implicit, derived from IDL metadata and a per-signer `expiresAt` path, not hand-written. Spec pins deterministic canonical-rendering rules per semantic format so signing and verification agree byte-for-byte. Changing a signer's `template` (or bumping `metadata.version`) invalidates all prior signatures, same as changing a function signature. |

### What this shape looks like

Rolling the answers up: a v1 unified IDL is a **typed node tree** closer to Codama's archetype than Anchor's flat Rust-struct snapshot, but deliberately restrained. It:

- Uses tagged unions everywhere (#1, #8, and elsewhere).
- Pushes all wire-layout detail into the type system; no `serialization` / `repr` hints (#2, #3, #10).
- Extracts PDAs to the program root (#5), optionally linked to account types.
- Supports multi-program documents (#6).
- Adopts `address` as an expression, drops `relations` (#8).
- Flattens composite accounts with a `group` tag (#7).
- Carries optional descriptive layers — contextual values (#4), semantic types (#11), structured remaining-accounts (#13), instruction summaries (#19) — that producers can opt into without affecting minimal conformance.
- Drops the `constants` block (#14) and the `serialization` / `repr` metadata (#10).
- Preserves Anchor-compat naming (#17), generics (#9), return types (#15).
- Adds `metadata.generator` (#12), a parameterized pre/post-instruction registry (#18), structured error messages (#16), per-signer templates folded into `messageSigner` presence (#19), and message-signer authorization with signed-intent + EIP-712-style domain preamble + per-signer expiry (#20).
- Reuses one path-expression and one semantic-format machinery across `address` (#8), defaults (#4), semantic type wrappers (#11), and message-signer rendering (#19/#20) — avoiding four almost-identical union definitions.

The spec stays small in the required core; the optional layers carry most of the expressivity.

---

## References

- This repo spec: [`specs/v0.1.0.md`](./v0.1.0.md)
- [`anchor-lang-idl-spec` on docs.rs](https://docs.rs/anchor-lang-idl-spec/latest/anchor_lang_idl_spec/) (latest: `0.1.0`)
- Quasar schema: [`blueshift-gg/quasar/schema/src/lib.rs`](https://github.com/blueshift-gg/quasar/blob/master/schema/src/lib.rs)
- Codama node types: [`codama-idl/codama/packages/node-types/src`](https://github.com/codama-idl/codama/tree/main/packages/node-types/src)
- Codama nodes package (helpers): [`codama-idl/codama/packages/nodes/src`](https://github.com/codama-idl/codama/tree/main/packages/nodes/src)
