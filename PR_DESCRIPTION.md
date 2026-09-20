Closes #480

`crates/termlens/examples/inspect.rs` is kept in step with `termlens inspect`
by hand, and nothing checked it. This adds tests in
`crates/termlens/tests/inspect.rs` that run the example and the command on the
same program, side by side, and require the same screen on stdout, the same
trailer on stderr and the same exit code.

What is compared:

- a program that exits (status 0, a non-zero status, styled output, and the
  `--flag=value` spelling): same screen, same `--- exited: … ---` trailer;
- a program that outlives the wait, both ways the wait can end: the silence
  window (`--- still running (killed on exit) ---`) and the deadline
  (`--- still running at the deadline (killed on exit) ---`);
- a child that closes its terminal but keeps running, which the reap grace
  must not report as exited;
- exit code 2 and an empty stdout for a bad flag, a bad value, a flag missing
  its value, no program, and a program that cannot be spawned;
- `--help`/`-h` and `--version`.

What is deliberately not compared: the usage text and the diagnostics, which
name the tool and legitimately differ (#453).

Timing (CONTRIBUTING §3): the exit and closed-terminal cases wait on an EOF,
not a clock. The silence-window case uses a 1 s window against a 10 s deadline,
far above what starting `sh` and printing a word needs, so neither is what the
test measures. The deadline case is the one that must expire; its 3 s deadline
also covers the spawn of a shell that prints one word. Each pair runs both
binaries concurrently, so a pair costs the slower of the two.

Verified that it catches the regression: with the pre-#465 sequential wait
restored in the example, and separately in `termlens inspect`,
`…_when_the_silence_window_ends_the_wait` and
`…_on_a_child_that_closes_its_terminal` fail, e.g.

    example: "--- still running at the deadline (killed on exit) ---\n"
    command: "--- still running (killed on exit) ---\n"

Gates run locally: `cargo fmt --check`, `cargo clippy --workspace --all-targets`
(default and `--all-features`), `cargo clippy -p termlens --no-default-features`,
and the `inspect` test target with default and `--no-default-features`.

This adds PTY spawns, so per §3 it needs the stress workflow; I will trigger it
from the Actions tab on this branch.
