# Contributing to the Solana IDL Specification

Thank you for your interest in contributing. This document explains how different types of contributions are handled and the RFC process for spec changes.

## Types of Contributions

| Contribution | Process |
|---|---|
| **Spec changes** (new fields, type modifications, new spec versions) | RFC required |
| **Tooling / ecosystem additions** (new framework, client generator) | Standard PR - minimun 2 maintainers approval |
| **Examples** | Standard PR - minimun 2 maintainers approval |
| **Typo / editorial fixes** | Standard PR - minimun 2 maintainers approval |

## RFC Process

Any change to the specification itself -- adding fields, modifying types, introducing new top-level sections, or proposing a new spec version -- must go through the RFC (Request for Comments) process.

### 1. Open an RFC Issue

Create a new issue using the **RFC** issue template. Fill in all sections: summary, motivation, detailed proposal, alternatives considered, and any breaking changes. A well-written RFC makes the discussion productive and the review faster.

### 2. Discussion Period

Once opened, the RFC enters a discussion period of **at least 1 week**. During this time, maintainers and community members will provide feedback. Be prepared to revise the proposal based on discussion. The issue will be labeled `rfc` by a maintainer.

An RFC may be:
- **Accepted** -- the proposal is approved and ready for implementation.
- **Revised** -- the author is asked to update the proposal before it can be accepted.
- **Declined** -- the proposal is closed with an explanation.

### 3. Submit a PR

Once the RFC is accepted, submit a pull request implementing the changes. Reference the RFC issue in the PR description (e.g. `Implements #12`). The PR should include:

- Updated or new spec document under `specs/`
- At least one example under `examples/` demonstrating the change
- Updates to `README.md` if the repo structure or tooling sections are affected

### 4. Review and Merge

Maintainers will review the PR against the accepted RFC. Once approved, the PR is merged and the RFC issue is closed.

## Standard Pull Requests

For contributions that do not modify the spec (tooling additions, examples, editorial fixes), open a PR directly -- no RFC needed.

### PR Conventions

- **Branch naming**: `feat/<short-description>`, `fix/<short-description>`, or `docs/<short-description>`
- **Commit messages**: use a short imperative summary (e.g. "Add Codama to client generation tools", "Fix typo in IdlType table")
- **Scope**: keep PRs focused on a single change; avoid bundling unrelated modifications

### What Makes a Good PR

- Clear title and description explaining **what** and **why**
- Links to any related issues
- Passes any CI checks (if configured)

## Code of Conduct

We are committed to providing a welcoming and respectful environment for everyone. Be kind, constructive, and professional in all interactions. Harassment, discrimination, and disruptive behavior will not be tolerated. Maintainers reserve the right to remove content or block participants who violate these expectations.
