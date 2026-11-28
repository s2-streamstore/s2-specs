## Local Conventions

- PR title + description become the squashed commit message at merge time; use conventional commit format.

## Source-of-Truth Boundaries

Keep server/API contracts and SDK requirements cleanly separated so they cannot drift:

- **API and wire contracts (`s2/v1/`):** `s2/v1/openapi.json`, `s2/v1/s2.proto`, `s2/v1/s2s.md`, and `s2/v1/error-codes.json` (together with `s2.dev/docs`) are the single source of truth for server and API behavior.
  - Regenerate `s2/v1/openapi.json` from `s2-streamstore/s2` `main` (`cargo run --locked -p s2-lite --features utoipa --bin openapi`); never hand-edit generated OpenAPI descriptions or schemas.
  - Keep `s2/v1/s2s.md` focused on the `s2s` wire protocol in plain language (it is embedded in public Mintlify docs; avoid internal spec filenames or SDK jargon).
- **SDK specification (`sdk/spec.md`):** `Terms` and all lettered chapters state **SDK requirements only**.
  - For every sentence, ask: *is this something an SDK must implement, or something the server promises?* Do not restate API option defaults or server behavior in chapter rules unless an SDK must enforce or rely on it directly (such as `A6c` on non-session transports, or `R1`'s `tail_offset=0` equivalence).
  - **`## Server guarantees`** is a concise dependency index (1–2 sentences per row): state only the minimal server invariant an SDK rule breaks without (`Relied on by`) and link to its authoritative source (`Source`). Omit server behavior no SDK rule relies on, full wire/framing details, and server-version rollout notes.
- **Rationale (`sdk/rationale.md`):** Informative only. Put design reasoning, trade-offs, cross-SDK notes, and server-version rollout context here—never normative requirements.
- **Shared vectors (`sdk/vectors/`):** Keep `sdk/vectors/*.json` and `sdk/vectors/README.md` aligned with `sdk/spec.md` and usable across all four SDKs (Python, TypeScript, Go, Rust), including `2⁵³ − 1` exact-integer boundaries (`D4`/`D5`) and sleep-free black-box synchronization.

## Cross-Cutting Changes

- When a review surfaces a server edge case, update the authoritative server/API spec first (`s2.proto`, `s2s.md`, `s2` OpenAPI docstrings, `docs`), and touch `sdk/spec.md` `Server guarantees` only if the minimal invariant an SDK rule relies on actually changed.
- While Revision 1 is in draft, keep `sdk/spec.md`, `sdk/rationale.md`, and `sdk/vectors/` rule IDs in sync; once finalized, rule IDs are permanently stable (never renumbered or reused).
