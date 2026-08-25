# TOS Documentation

This repository contains technical documentation for projects in the
[TOS Network](https://github.com/tosnetwork) ecosystem.

## Agentic Internet Constitution

- [The Agentic Internet Constitution — TOS Founding
  Edition](AGENTIC_INTERNET_CONSTITUTION.md) — open Agentic Internet design
  principles, independent-implementation guarantees, rights, institutional
  boundaries, amendment rules, and decision tests for the proposed TOS
  adoption. Its current status is stated in the document; repository
  publication alone does not constitute ratification.
- [Constitution Completeness
  Review](AGENTIC_INTERNET_CONSTITUTION_COMPLETENESS_REVIEW.md) — internal
  findings, remediations, completeness matrix, and remaining ratification
  gates. It is not an implementation, deployment, legal, or external-security
  audit.

## Documentation Sets

- [TOS Blockchain](tos-blockchain/README.md) — protocol specifications,
  architecture and standards, node and validator operations, smart-contract
  tooling, and blockchain development guides.

The corresponding Layer-1 implementation is maintained in the
[`tosnetwork/tos`](https://github.com/tosnetwork/tos) repository.

## Repository Layout

Each top-level directory is a self-contained documentation set for one TOS
project or technical domain. Start with that directory's `README.md` for its
index and scope.

## Contributing

Keep implementation-specific documentation aligned with the source repository
and use relative links between documents in the same documentation set.
Documents presented as open Agentic Internet protocols must define
implementation-neutral behavior and conformance; they must not require reuse
of a TOS reference codebase.

## License

This documentation repository is licensed under the [GNU General Public
License v3.0](LICENSE). The included license text matches the license used by
the [`tosnetwork/tos`](https://github.com/tosnetwork/tos/blob/main/LICENSE)
repository.
