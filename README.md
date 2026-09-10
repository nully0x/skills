# rust-style

Style guide for writing Rust code. Tells your agent how to write boring, idiomatic, strongly-typed Rust — not AI-slop.

## Installation

```bash
npx skills add your-username/rust-style
```

Pick `rust-style` when prompted.

## What it does

This skill teaches your agent to:

- Use concrete types instead of `serde_json::Value` / `dyn Any`
- Return `Result` instead of swallowing errors
- Keep traits small and justified
- Avoid pointless abstraction layers, factories, builders
- Write straightforward control flow with early returns
- Validate once at the boundary, not everywhere
- Keep code boring and explicit

## License

MIT
