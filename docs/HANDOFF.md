# Kairo 4.0 — handoff (2026-10-12)

Where the Rust rewrite stands, for whoever picks it up next. The full
decision record is [RUST_REWRITE_LOG.md](RUST_REWRITE_LOG.md); this page is
the short version and the to-do list.

## State
- Branch **`rust-rewrite`**, version **4.0.0** (unreleased — keep 4.0.0 until
  it ships). **Not merged to main**: the owner merges after their full test.
- The Node server is replaced by `src-tauri/crates/kairo-server` (axum),
  running inside the app in release builds. Node is no longer bundled.
  `server/` (Node) stays only for comparison tools until the full test passes.
- Installed on the owner's Mac: `/Applications/Kairo.app` built from this
  branch (with their data and the offline model kept).

## Run / test / build
- Dev: `npm run dev` (port 7777 must be free). Server alone: `npm run server`.
- Rust tests: `cargo test --release --workspace` in `src-tauri`.
- UI tests: `node --test src/<file>.test.js` (each file; the directory form errors).
- Detection replays (no app needed): `cargo run --release -p kairo-eval --bin replay`
  (`--meaning gemma-legacy.json`, `--offline <name>`, `--verbose`, `--trace a..b`).
  Plain transcript through the engine: `--bin say -- file.txt --explain`.
  Offline engine on a recording: `--bin offline-transcribe -- <audio> <name>`.
- Bundle data (models + verse index): `--bin bundle-data` — also the
  `beforeBuildCommand`. Mac build: `npx @tauri-apps/cli@latest build --bundles app,dmg`.

## Numbers (recorded sermons, answer keys in `eval/fixtures`)
- Deepgram, with meaning models: 226/251 found (Node 216), Possible Matches
  ~14/hour.
- Scores as of 2026-10-14 (plain replay, no meaning models; the scorer now
  reads range entries — `verseStart`/`verseEnd` — and four duplicate key
  entries were removed, so totals moved from 255 to 258):
  - Deepgram, dev sermons: 223/258 found, 6 wrong (218/255, 8 wrong before
    the spoken-correction, range, split-citation, sibling-book, finished-
    range and twin-verse fixes).
  - Held-out (checked, never tuned): 63/76, 4 wrong — unchanged by those
    fixes. Node scored 25/30 on shiloh2025-love-mystery vs Rust 23/30; that
    gap predates them and hasn't been examined (it's held out).
  - Offline (blank penalty 1.0) vs Deepgram on the two sermons with audio:
    64/81, 1 wrong vs 66/81, 0 wrong. Drops of 4+ words 39 -> 23.
  - Possible Matches can't be promoted by score: even at 0.90+ there are 9
    wrong offers per 29 right ones.

## Constraints from the owner
- Nothing pushed or merged to **main** without them; commit freely on the
  branch; scan for secrets first. Never print the Deepgram key
  (`databases/settings.json`, untracked).
- No added costs for users and no app bloat (no cloud LLM, no bundled LLM;
  optional downloads only when a feature is used).
- Don't tune on the held-out sermons (oyedepo-meditation,
  shiloh2025-love-mystery).
- The UI must look the same except where the owner asked for changes.
- Brand: solid `#F5301A` for primary actions (no gradient buttons); default
  Bible theme Kairo Red (`#F5301A`→`#9E1608`); neutral white keyboard focus.

## To do
1. **Owner's full test** of the installed 4.0.0.
2. ~~First CI run~~ — done 2026-10-13 (run 38345883921): macOS universal,
   Windows (.msi + setup .exe) and Linux (.deb, .rpm) all build, with update
   signatures. Linking notes, in `/.cargo/config.toml`: sherpa-onnx ships
   Windows libraries for the static C runtime only, so Windows builds use
   `+crt-static` (no VC++ redistributable needed); on Windows and Intel Macs
   ort links no ONNX Runtime of its own and uses sherpa-onnx's (1.28.2) —
   ort's Windows build is for the DLL runtime and it has none for Intel.
   Bump ort and sherpa-onnx together and keep their ONNX Runtime versions
   compatible. The Pages job fails on branches other than main (environment
   rule) — expected.
3. **Release**: tag `v4.0.0` (draft release), publish it **as Latest** —
   "Offline STT model v1" is currently marked Latest, so the updater's
   `releases/latest/download/latest.json` finds nothing.
4. After the full test: remove `server/` (the Node app) and
   `npm run server:node`; move the eval fixtures out of `eval`.
5. Beam search: **leave it off**, even once sherpa-onnx releases it for NeMo
   streaming models (k2-fsa/sherpa-onnx#3895; 1.13.8 exits if asked for it).
   Measured 2026-10-14 on master built from source, with the blank penalty:
   62/81 found, 3 wrong — worse than the blank penalty alone (65/81, 1 wrong);
   alone (greedy-era) it was 65/81, 2 wrong. Book-name hotwords also stay off:
   `hotwords()` hands sherpa word pieces, but it needs `modeling_unit = "bpe"`
   + a `bpe_vocab` and plain names; done that way at 1.5 they found one more
   verse but inserted ~3 false book names per fix ("Matthew three times").
6. Possible later: a second, non-streaming pass for offline (Parakeet) —
   an optional download; ask the owner first. Tried and not kept (2026-10-13):
   restarting the stream at every pause (same-word 78-80% vs 86-88%), and a
   second stream re-listening after each pause (recovered single phrases, but
   net 64/81 at ~2-3x CPU). Run one full-sermon transcription at a time on the
   8 GB M1 — several at once swap and crawl at 0.2x real time.

## Untracked on purpose
`parity/lexical_node.jsonl`, `parity/parser_node.jsonl` (generated by the Node
parity scripts; large, regenerable).
