# IPI Vote

An experimental Cosmos governance interface used to evaluate wallet
connection, proposal display, and voting flows for IPI-compatible networks.

> **Status: inherited prototype.** This application is not the canonical IPI
> governance system, has not been audited, and must not be used to make or
> represent production governance decisions.

## Development

The codebase uses an older Create Cosmos App / Next.js stack. Review and update
its dependencies before exposing a deployment.

```sh
yarn
yarn dev
yarn build
```

The local application is served at `http://localhost:3000`. Network and wallet
settings are under `config/`; never assume template defaults identify the
intended IPI network.

## Contributing safely

Useful work includes explicit chain-identity display, transaction simulation,
human-readable vote messages, accessibility, deterministic tests, and clear
handling of wallet and RPC failures. Any proposed production use requires a
threat model and independent security review.

## Provenance and license

This repository derives from Hyperweb Create Cosmos App's `vote-proposal`
example. See [UPSTREAM.md](UPSTREAM.md) for provenance and migration context.
Upstream and IPI modifications are distributed under the [MIT License](LICENSE).
