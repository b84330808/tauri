# Cloud report: tauri-apps/tauri builtin dev server percent-decoding fix

## Summary

| | |
| --- | --- |
| Fork branch | `claude/modest-gauss-jkvkge` on `b84330808/tauri` |
| Commit | `443ca06e5391d8a3b3d8a3d6694aa4c256277a9f` (one commit) |
| Target | `tauri-apps/tauri` branch `dev` (PRs target `dev`), based on `548a3692f59405aeb239c5b63fa589824eee51e8` (upstream `dev` HEAD when the work was done) |
| Issue | https://github.com/tauri-apps/tauri/issues/12763 (open, `type: bug`, `status: needs triage`, no linked branch or PR) |
| Files | `crates/tauri-cli/src/dev/builtin_dev_server.rs`, `crates/tauri-cli/Cargo.toml`, `Cargo.lock` (one line), `.changes/cli-dev-server-percent-decode.md` |

Nothing was opened, edited or commented on upstream.

## Policy check (re-checked on upstream `dev` at `548a369`)

- **AI tool policy** (`.github/CONTRIBUTING.md#ai-tool-policy`): AI-assisted PRs are allowed. Two rules:
  1. "Review and test all LLM-generated content before submitting, you're the one responsible for it, not the AI."
  2. "Don't use AI to respond to review comments (except for translations)."

  PRs that break these rules are closed with an `ai-slop` label, and the label is in active use. The policy is not a ban, and neither the PR template nor CONTRIBUTING asks the author to attest to personal testing in the PR. So the work went ahead, but **before opening the PR, the local session must have the user review the diff and run the test** (commands below). **Review replies must be written by the user, not by an AI.**
- **Signed commits are required.** Both CONTRIBUTING ("You have to sign your commits") and the PR template (item 4) say so. The cloud commit carries an SSH signature from the cloud environment's key, not from the user's key, so GitHub will not show it as verified for `b84330808`. **Re-sign it locally before pushing for the PR**, for example:
  ```bash
  git commit --amend --no-edit -S
  ```
  Then force-push the branch to the fork (`git push --force-with-lease`).
- **CLA / DCO:** none. There is no CLA bot or DCO check in `.github/`, and no `Signed-off-by` requirement.
- **Change file:** required for released packages (`.changes/README.md`, covector). Added `.changes/cli-dev-server-percent-decode.md` with `tauri-cli` and `@tauri-apps/cli` as `patch:bug`, matching the format of the existing CLI change files. `node ./.scripts/ci/check-change-tags.js` and `prettier --check` pass on it.
- **PR title convention:** CONTRIBUTING asks for `(fix #xxxx)` in the title when an issue is resolved. The PR template asks for a descriptive conventional title and for `cargo test` and `cargo clippy` to pass.
- **New dependency:** `percent-encoding = "2"` is added to `tauri-cli`. It is already in `Cargo.lock` (2.3.2, used by `tauri`, `tauri-runtime-wry` and `url`), so no new crate enters the graph. cargo-vet in CI only runs `suggest`, and `cargo udeps` is fine because the crate is used.

## The bug

The builtin dev server (`crates/tauri-cli/src/dev/builtin_dev_server.rs`) serves `build.frontendDist` during `tauri dev` when no `devUrl` is set. It looked files up with `uri.path()`, which is still percent-encoded. A request for `/my%20font.woff2` searched for a file literally named `my%20font.woff2`, then `my%20font.woff2.html` and `my%20font.woff2/index.html`, and then fell back to `index.html`. The webview got the HTML page (with the reload script injected) instead of the file. Any file with a space or non-ASCII character in its name was affected.

Production builds are not affected: `AppManager::get_asset` in `crates/tauri/src/manager/mod.rs` has percent-decoded the path since https://github.com/tauri-apps/tauri/pull/1938, which fixed https://github.com/tauri-apps/tauri/issues/1879 ("Can't load files with spaces in build"). That matches the issue report exactly: the font loads in the built `.exe` but not in `tauri dev` (`frontendDist: ../src`, and the reporter confirmed in the thread that the app ran on port 1430, the builtin server's default).

The fix decodes the path the same way production does, with `percent_decode_str(..).decode_utf8_lossy()`, before the lookup. Decoding can produce `..` or an absolute path (`%2E%2E`, `%2F`). Every candidate path still goes through the existing `fs_read_scoped`, which canonicalizes the path and rejects anything outside of the served directory. The new test covers those traversal cases, and I confirmed it fails if the scope check is disabled (details below).

## PR title

```
fix(cli): percent-decode request paths in the builtin dev server (fix #12763)
```

## PR body

The template (`.github/PULL_REQUEST_TEMPLATE.md`) only contains a comment with guidelines: a descriptive title, a reference to the related issue, a change file, signed commits, and passing `cargo test` / `cargo clippy`. The body below follows them.

```markdown
Closes #12763

The builtin dev server, which `tauri dev` uses when `build.devUrl` is not set, looked files up with the raw request path from `Uri::path()`, which is still percent-encoded. A stylesheet with `src: url("my font.woff2")` makes the webview request `/my%20font.woff2`. The server looked for a file literally named `my%20font.woff2`, did not find it and fell back to `index.html`. So the font, or any other file with spaces or non-ASCII characters in its name, failed to load in development. Production builds are not affected because `AppManager::get_asset` has percent-decoded the asset path since #1938.

This PR decodes the path the same way, with `percent_decode_str(..).decode_utf8_lossy()`, before the lookup. Decoding can produce `..` or an absolute path (`%2E%2E`, `%2F`), but every candidate still goes through `fs_read_scoped`. That function canonicalizes the path and rejects anything outside of the served directory.

`percent-encoding` is now a direct dependency of `tauri-cli`. It was already in the lockfile, used by `tauri` and `url`.

### Testing

- The new `dev::builtin_dev_server::tests::serves_percent_encoded_paths` test calls the request handler for `/my%20font.woff2` and `/%E9%A1%B5%E9%9D%A2.css` (`页面.css`) and checks that the file contents are returned. It also checks that `/%2E%2E/secret.txt`, `/..%2Fsecret.txt` and a percent-encoded absolute path don't return a file outside of the served directory. Without the fix, the first request returns `index.html` and the test fails.
- `cargo test -p tauri-cli`, `cargo clippy -p tauri-cli --all-targets -- -D warnings` and `cargo fmt --check` pass on Linux.

I used Claude Code to write this change.
```

## Fail-before (fix reverted, test and dependency kept)

`RUST_BACKTRACE=0 cargo test -p tauri-cli --lib builtin_dev_server`, with only the handler change reverted (`let uri = uri.path();`). In that state the assertion is on line 401; it is line 404 in the committed file.

```text
running 5 tests
test dev::builtin_dev_server::tests::parses_host_header ... ok
test dev::builtin_dev_server::tests::host_check ... ok
test dev::builtin_dev_server::tests::origin_check ... ok
test dev::builtin_dev_server::tests::fs_read_scoped_rejects_outside_scope ... ok
test dev::builtin_dev_server::tests::serves_percent_encoded_paths ... FAILED

failures:

---- dev::builtin_dev_server::tests::serves_percent_encoded_paths stdout ----

thread 'dev::builtin_dev_server::tests::serves_percent_encoded_paths' (11400) panicked at crates/tauri-cli/src/dev/builtin_dev_server.rs:401:5:
assertion `left == right` failed
  left: b"<html><head><script>// Copyright 2019-2024 Tauri Programme within The Commons Conservancy\n// SPDX-License-Identifier: Apache-2.0\n// SPDX-License-Identifier: MIT\n\n// taken from https://github.com/thedodd/trunk/blob/5c799dc35f1f1d8f8d3d30c8723cbb761a9b6a08/src/autoreload.js\n\n;(function () {\n  const reload_url = 'ws://127.0.0.1:1430/__tauri_cli' [... rest of the injected reload script ...] </script></head><body></body></html>"
 right: "font"

failures:
    dev::builtin_dev_server::tests::serves_percent_encoded_paths

test result: FAILED. 4 passed; 1 failed; 0 ignored; 0 measured; 109 filtered out; finished in 0.00s

error: test failed, to rerun pass `-p tauri-cli --lib`
```

## Pass-after (commit `443ca06`)

```text
running 5 tests
test dev::builtin_dev_server::tests::parses_host_header ... ok
test dev::builtin_dev_server::tests::host_check ... ok
test dev::builtin_dev_server::tests::origin_check ... ok
test dev::builtin_dev_server::tests::fs_read_scoped_rejects_outside_scope ... ok
test dev::builtin_dev_server::tests::serves_percent_encoded_paths ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 109 filtered out; finished in 0.00s
```

Full crate, `RUST_BACKTRACE=0 cargo test -p tauri-cli`:

```text
     Running unittests src/lib.rs (target/debug/deps/tauri_cli-d8c92ca74b866261)
test result: ok. 114 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 11.51s
     Running unittests src/main.rs (target/debug/deps/cargo_tauri-6ee0b1888a118378)
test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
     Running tests/init.rs (target/debug/deps/init-9a34e31e92e6c813)
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.03s
   Doc-tests tauri_cli
test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Other checks, run on Linux x86_64 with rustc 1.97.0:

- `cargo fmt -p tauri-cli --check`: clean.
- `cargo clippy -p tauri-cli --all-targets -- -D warnings`: clean. CI also passes `--all-features`; that wasn't run here because of the `native-tls` system dependencies.
- `npx prettier@3 --check .changes/cli-dev-server-percent-decode.md`: clean.
- `node ./.scripts/ci/check-change-tags.js .changes/cli-dev-server-percent-decode.md`: exit 0.

### Extra checks (not committed)

1. **Real server over TCP.** A temporary test started the real server with `start()` on `127.0.0.1:14301` and sent a raw `GET /my%20font.woff2` for a file `my font.woff2` containing `FONT-BYTES`:

   Without the fix:
   ```text
   HTTP/1.1 200 OK
   content-type: text/html
   content-length: 1043
   body: "<html><head><script>// Copyright 2019-2024 Tauri Programme w..."   (index.html)
   ```
   With the fix:
   ```text
   HTTP/1.1 200 OK
   content-type: application/octet-stream
   content-length: 10
   body: "FONT-BYTES"
   ```
   The content type is `application/octet-stream` only because the fake bytes aren't a real WOFF2 file. With a real font, `infer` sniffs `font/woff2`.

2. **The traversal assertions really test the scope check.** With `fs_read_scoped`'s `path.starts_with(scope)` check temporarily disabled, the new test fails at the `%2E%2E` assertion (`left: b"secret"`, `right: "secret"`). It passes again once the check is restored.

## Suggested local verification (for the AI policy)

```bash
git fetch origin claude/modest-gauss-jkvkge
```

```bash
git checkout claude/modest-gauss-jkvkge
```

```bash
cargo test -p tauri-cli --lib builtin_dev_server
```

Optional manual repro: create an app without `devUrl` and with `frontendDist` pointing at a folder that has a font file whose name contains a space, used from CSS. Run `tauri dev` with the CLI built from `dev` (the font fails to load), then with the CLI built from this branch (`cargo install --path crates/tauri-cli --debug`), and the font loads.

## Duplicate search (all states, `tauri-apps/tauri`, 2026-10-30)

Searched with the GitHub search API (PR search covers open, closed and merged PRs), and read issue #12763's page for comments and linked PRs.

| Query | Result |
| --- | --- |
| PRs `12763` | No PR references the issue. 4 hits, all unrelated dependency bumps (https://github.com/tauri-apps/tauri/pull/4102, https://github.com/tauri-apps/tauri/pull/4296, https://github.com/tauri-apps/tauri/pull/1248, https://github.com/tauri-apps/tauri/pull/1244) |
| PRs `builtin dev server` | https://github.com/tauri-apps/tauri/pull/16095 (closed, Host/Origin validation), https://github.com/tauri-apps/tauri/pull/10600 (closed, Android forwarding), https://github.com/tauri-apps/tauri/pull/9344 (closed, directory traversal), https://github.com/tauri-apps/tauri/pull/10456 (closed, iOS tun address). None touch percent-decoding |
| PRs `builtin_dev_server` | none |
| PRs `"dev server" decode path` | none |
| PRs `dev server percent decode` | only bot version-update PRs |
| PRs `dev server space file name` | only bot and dependency PRs |
| PRs `percent-decode` | https://github.com/tauri-apps/tauri/pull/1938 (merged 2021, the production `get_asset` fix for #1879), https://github.com/tauri-apps/tauri/pull/2427 (asset protocol), https://github.com/tauri-apps/tauri/pull/4035 (file drop), https://github.com/tauri-apps/tauri/pull/16197 (empty URI path), https://github.com/tauri-apps/tauri/pull/15969 (CSP data URLs). None touch the CLI dev server |
| PRs `spaces file name` | https://github.com/tauri-apps/tauri/pull/11218 (bundler `Exec` quoting), https://github.com/tauri-apps/tauri/pull/12290 (iOS team name). Unrelated |
| PRs `url encoded path` | `convert_file_src` features, dependency bumps. Unrelated |
| PRs `mime type case insensitive extension` | none |
| PRs by `author:b84330808` | none (no earlier PRs by the user in this repo) |
| Issues: dev server, space, percent-encoded | https://github.com/tauri-apps/tauri/issues/12763 is the only match describing this bug |
| Issues: dev mode, non-ASCII file names | #12763 again. https://github.com/tauri-apps/tauri/issues/1879 (closed 2021) is the production-build version, fixed by #1938 |
| Issue #12763 page | 2 comments: a contributor asked whether the app ran on port 1430, and the author said yes. Development: "No branches or pull requests" |

## Candidates rejected before this one

| Issue | Reason |
| --- | --- |
| https://github.com/tauri-apps/tauri/issues/15635 (`tauri init` panics on quotes in the title) | Already fixed on `dev`: `render_tauri_config` JSON-escapes values |
| https://github.com/tauri-apps/tauri/issues/16033 (`tauri info` panics on pnpm peer-suffixed paths) | Open PR https://github.com/tauri-apps/tauri/pull/16058 |
| https://github.com/tauri-apps/tauri/issues/16250 (target-table `tauri` features merged) | Open PR https://github.com/tauri-apps/tauri/pull/16393 (and closed https://github.com/tauri-apps/tauri/pull/16364) |
| Case-sensitive extension matching in `tauri_utils::mime_type` (`STYLE.CSS` gets the fallback type) | Self-found and kept as a backup idea. Lower impact, no issue, not pursued |

## Notes

- The fix branch has exactly one commit on top of upstream `dev` `548a369`. Author and committer: `b84330808 <b84330808@gmail.com>`. Trailer: `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- The commit message says `Fixes tauri-apps/tauri#12763`. The PR title and body use `#12763` / `Closes #12763`, as upstream conventions ask.
- If upstream `dev` moves before the PR is opened, rebase onto it. The diff touches only the files listed above. A `Cargo.lock` conflict is unlikely but would just mean re-adding `"percent-encoding"` to the `tauri-cli` dependency list.
