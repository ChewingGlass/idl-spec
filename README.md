# Solana IDL Specification

A unified reference for Solana program Interface Description Language (IDL) formats. IDLs describe the public interface of on-chain programs -- their instructions, accounts, types, events, errors, and constants -- enabling tooling to generate clients, documentation, and transaction builders automatically.

This first version of the specification is based on the [`anchor-lang-idl-spec`](https://docs.rs/anchor-lang-idl-spec/0.1.0/anchor_lang_idl_spec/) and will serve as a framework-agnostic standard for all Solana IDLs.

## Specification

| Version | Status     | Document | JSON Schema |
|---------|------------|----------|-------------|
| `0.1.0` | Current    | [`specs/v0.1.0.md`](specs/v0.1.0.md) | [`schema/v0.1.0.json`](schema/v0.1.0.json) |

## Ecosystem Tools

### Frameworks

Frameworks that generate IDLs conforming to this spec:

- **[Anchor](https://anchor-lang.com/)** -- Solana program framework maintained by Ottersec. Generates IDLs at build time via `anchor build`.
- **[Quasar](https://quasar-lang.com/)** -- Solana program framework maintained by Blueshift. Generates IDLs at build time via `quasar build`.

- Native or [Pinocchio](https://github.com/anza-xyz/pinocchio) programs can also generate [Codama IDLs](https://github.com/codama-idl/codama#getting-a-codama-idl) via Codama macros. This Solana IDL spec can be converted to Codama IDLs if needed.

### Client Generation

#### Anchor

Anchor automatically generates Web3js clients for your program via `anchor build`.

#### Quasar 

Quasar automatically generates Js clients and Rust clients for your program via `quasar build`.

#### Codama
The Solana IDL can be converted easily into a Codama IDL and then be used for client generation.

**[Codama](https://github.com/codama-idl/codama)** -- Generate typed clients from Solana IDL. Supports multiple target languages (TypeScript, Rust, Go, Python, etc.).

```bash
pnpm install codama
codama init
codama run --all
```

#### C# 

You can generate C# clients for your program via the [Magic block sdk]`https://github.com/magicblock-labs/Solana.Unity.Anchor`.

```bash
dotnet anchorgen -i idl/file.json -o src/ProgramCode.cs
```

### Upload & Manage On-Chain Metadata

**[Program Metadata](https://github.com/solana-program/program-metadata)** -- Attach IDLs, security.txt, and other metadata to any Solana program via on-chain PDA accounts.

```bash
# Upload an IDL to your program
npx @solana-program/program-metadata@latest write idl <program-id> ./idl.json

# Fetch an IDL from a deployed program
npx @solana-program/program-metadata@latest fetch idl <program-id> --output ./idl.json
```

Using the legacy IDL upload via the injected anchor code into anchor programs is deprecated for security reasons and you should move to use the new Program Metadata program. 

### Historical IDL Indexing

**[Historical IDL](https://github.com/Woody4618/historical-idl)** -- Reconstruct the full version history of a Solana program's IDL from on-chain transactions. Supports both legacy Anchor IDL and Program Metadata formats.

```bash
npx tsx src/cli.ts <program-address> --rpc <rpc-url> --type both --dump-idls ./idls
```

## Repository Structure

```
idl-spec/
├── README.md                   # This file
├── CONTRIBUTING.md             # Contribution guidelines and RFC process
├── LICENSE                     # MIT license
├── schema/
│   └── v0.1.0.json             # JSON Schema for IDL validation
├── specs/
│   └── v0.1.0.md               # IDL spec v0.1.0
└── examples/
    └── counter.json            # Example IDL
```

## Contributing

Contributions are welcome. See [`CONTRIBUTING.md`](CONTRIBUTING.md) for full details.

- **Spec changes** go through the [RFC process](CONTRIBUTING.md#rfc-process) -- open an issue using the RFC template, allow at least 2 weeks for discussion, then submit a PR once accepted.
- **Tooling additions, examples, and editorial fixes** can be submitted as a standard pull request.

## License

This project is licensed under the [MIT License](LICENSE).
