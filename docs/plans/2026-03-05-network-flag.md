# `--network` flag for the coder container implementation plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement
> this plan task-by-task.
>
> **REQUIRED: Every commit must use the creative-commits skill. After each
> successful commit, run `git push`.**

**Goal:** Add a `--network <name>` flag (and `LEASH_NETWORK` env var) to leash so
the coder container can be attached to a named Docker network, enabling it to reach
other containers on that network (e.g. a backend API under development).

**Architecture:** The `--network` value is stored in `options.network`, read from
`LEASH_NETWORK` env var as fallback via `applyNetworkEnv`, and passed as
`--network <name>` to the `docker run` call in `launchTargetContainer`. The leash
container inherits the named network automatically via its existing
`--network container:<target>` setup. No changes to the security model or leash
container launch are required.

**Tech stack:** Go, Docker CLI, existing `internal/runner` package test patterns.

---

### Task 1: Add `network` field to `options` and parse the flag

**Files:**
- Modify: `internal/runner/runner.go`
- Modify: `internal/runner/runner_args_test.go`

**Step 1: Write the failing tests**

Add to `internal/runner/runner_args_test.go`:

```go
func TestParseArgsNetworkFlag(t *testing.T) {
	t.Parallel()

	opts, err := parseArgs([]string{"--network", "my-app-network", "--", "opencode"})
	if err != nil {
		t.Fatalf("parseArgs returned error: %v", err)
	}
	if opts.network != "my-app-network" {
		t.Fatalf("expected network %q, got %q", "my-app-network", opts.network)
	}
}

func TestParseArgsNetworkFlagEquals(t *testing.T) {
	t.Parallel()

	opts, err := parseArgs([]string{"--network=my-app-network"})
	if err != nil {
		t.Fatalf("parseArgs returned error: %v", err)
	}
	if opts.network != "my-app-network" {
		t.Fatalf("expected network %q, got %q", "my-app-network", opts.network)
	}
}

func TestParseArgsNetworkFlagMissingValue(t *testing.T) {
	t.Parallel()

	if _, err := parseArgs([]string{"--network"}); err == nil {
		t.Fatal("expected error for --network without value")
	}
}
```

**Step 2: Run tests to verify they fail**

```
go test ./internal/runner/... -run TestParseArgsNetwork -v
```

Expected: FAIL — `opts.network` field doesn't exist yet.

**Step 3: Add `network string` to the `options` struct**

In `internal/runner/runner.go`, add `network string` to the `options` struct
(after `publishAll bool`, around line 107):

```go
type options struct {
    // ... existing fields ...
    publishAll     bool
    network        string // --network / LEASH_NETWORK
}
```

**Step 4: Handle `--network` in `parseArgs`**

In the `switch arg` block inside `parseArgs` (after the `"-P", "--publish-all"`
case, around line 385), add:

```go
case "--network":
    if i+1 >= len(args) {
        return opts, fmt.Errorf("missing argument for %s", arg)
    }
    opts.network = strings.TrimSpace(args[i+1])
    i++
```

In the `default` / `strings.HasPrefix` block (around line 436), add:

```go
case strings.HasPrefix(arg, "--network="):
    opts.network = strings.TrimSpace(strings.TrimPrefix(arg, "--network="))
```

**Step 5: Run tests to verify they pass**

```
go test ./internal/runner/... -run TestParseArgsNetwork -v
```

Expected: PASS (3 tests).

**Step 6: Run full test suite**

```
go test ./internal/runner/...
```

Expected: all pass.

**Step 7: Commit and push**

Use the creative-commits skill to commit, then push:

```bash
git add internal/runner/runner.go internal/runner/runner_args_test.go && git commit && git push
```

---

### Task 2: Add `applyNetworkEnv` for `LEASH_NETWORK` env var fallback

**Files:**
- Modify: `internal/runner/runner.go`
- Modify: `internal/runner/runner_args_test.go`

**Step 1: Write the failing test**

Add to `internal/runner/runner_args_test.go` (this test modifies env; keep
serial — no `t.Parallel()`):

```go
func TestApplyNetworkEnv(t *testing.T) {
	clearEnv(t, "LEASH_NETWORK")

	opts := options{}
	applyNetworkEnv(&opts)
	if opts.network != "" {
		t.Fatalf("expected network to be empty when LEASH_NETWORK unset, got %q", opts.network)
	}

	setEnv(t, "LEASH_NETWORK", "my-app-network")
	opts = options{}
	applyNetworkEnv(&opts)
	if opts.network != "my-app-network" {
		t.Fatalf("expected network %q from env, got %q", "my-app-network", opts.network)
	}

	// CLI flag wins over env var
	setEnv(t, "LEASH_NETWORK", "env-network")
	opts = options{network: "cli-network"}
	applyNetworkEnv(&opts)
	if opts.network != "cli-network" {
		t.Fatalf("expected CLI value to win, got %q", opts.network)
	}
}
```

**Step 2: Run test to verify it fails**

```
go test ./internal/runner/... -run TestApplyNetworkEnv -v
```

Expected: FAIL — `applyNetworkEnv` not defined yet.

**Step 3: Implement `applyNetworkEnv`**

Add near `applyOpenEnv` (around line 521 in `runner.go`):

```go
func applyNetworkEnv(opts *options) {
	if opts == nil || opts.network != "" {
		return
	}
	if net := strings.TrimSpace(os.Getenv("LEASH_NETWORK")); net != "" {
		opts.network = net
	}
}
```

**Step 4: Wire the call**

Find the call site for `applyOpenEnv` in `runner.go` and add `applyNetworkEnv`
immediately after it:

```go
applyOpenEnv(&opts)
applyNetworkEnv(&opts)
```

**Step 5: Run tests**

```
go test ./internal/runner/... -run TestApplyNetworkEnv -v
```

Expected: PASS.

**Step 6: Run full suite**

```
go test ./internal/runner/...
```

Expected: all pass.

**Step 7: Commit and push**

Use the creative-commits skill to commit, then push:

```bash
git add internal/runner/runner.go internal/runner/runner_args_test.go && git commit && git push
```

---

### Task 3: Pass `--network` to `launchTargetContainer` and test it

**Files:**
- Modify: `internal/runner/runner.go`
- Modify: `internal/runner/mounts_test.go`

**Step 1: Write the failing tests**

Add to `internal/runner/mounts_test.go`. Follow the same mock pattern as the
existing `TestLaunchCommandsUseSplitMounts` test (lock `mountStateTestMu`,
replace `runCommand` and `commandOutput` globals, build a `runner` struct,
call `launchTargetContainer`, inspect captured args).

Note: use `strings.Builder` for the logger (consistent with the existing test),
not `io.Discard`.

```go
func TestLaunchTargetContainerWithNetwork(t *testing.T) {
	mountStateTestMu.Lock()
	t.Cleanup(mountStateTestMu.Unlock)

	shareDir := t.TempDir()
	privateDir := filepath.Join(shareDir, "private")
	logDir := filepath.Join(shareDir, "log")
	cfgDir := filepath.Join(shareDir, "cfg")
	for _, dir := range []string{privateDir, logDir, cfgDir} {
		if err := os.MkdirAll(dir, 0o755); err != nil {
			t.Fatalf("mkdir %s: %v", dir, err)
		}
	}

	var (
		mu       sync.Mutex
		captured []string
		origRun  = runCommand
		origOut  = commandOutput
		logBuf   strings.Builder
	)
	runCommand = func(_ context.Context, _ string, args ...string) error {
		mu.Lock()
		defer mu.Unlock()
		copied := make([]string, len(args))
		copy(copied, args)
		captured = copied
		return nil
	}
	commandOutput = func(_ context.Context, name string, args ...string) (string, error) {
		if name == "docker" && len(args) >= 3 && args[0] == "inspect" &&
			strings.Contains(args[2], "{{.Architecture}}") {
			return "amd64\n", nil
		}
		return "", fmt.Errorf("unexpected commandOutput: %s %v", name, args)
	}
	t.Cleanup(func() {
		runCommand = origRun
		commandOutput = origOut
	})

	r := &runner{
		logger: log.New(&logBuf, "", 0),
		cfg: config{
			shareDir:         shareDir,
			privateDir:       privateDir,
			logDir:           logDir,
			cfgDir:           cfgDir,
			targetImage:      "example/target:latest",
			targetContainer:  "leash-target-net",
			callerDir:        shareDir,
			listenCfg:        listen.Default(),
			bootstrapTimeout: 30 * time.Second,
		},
		opts: options{
			network: "my-app-network",
			command: []string{"opencode"},
		},
	}

	if err := r.launchTargetContainer(context.Background(), "SIGTERM"); err != nil {
		t.Fatalf("launch failed: %v", err)
	}

	mu.Lock()
	args := captured
	mu.Unlock()

	found := false
	for i, arg := range args {
		if arg == "--network" && i+1 < len(args) && args[i+1] == "my-app-network" {
			found = true
			break
		}
	}
	if !found {
		t.Fatalf("expected --network my-app-network in docker args, got: %v", args)
	}
}

func TestLaunchTargetContainerWithoutNetwork(t *testing.T) {
	mountStateTestMu.Lock()
	t.Cleanup(mountStateTestMu.Unlock)

	shareDir := t.TempDir()
	privateDir := filepath.Join(shareDir, "private")
	logDir := filepath.Join(shareDir, "log")
	cfgDir := filepath.Join(shareDir, "cfg")
	for _, dir := range []string{privateDir, logDir, cfgDir} {
		if err := os.MkdirAll(dir, 0o755); err != nil {
			t.Fatalf("mkdir %s: %v", dir, err)
		}
	}

	var (
		mu       sync.Mutex
		captured []string
		origRun  = runCommand
		origOut  = commandOutput
		logBuf   strings.Builder
	)
	runCommand = func(_ context.Context, _ string, args ...string) error {
		mu.Lock()
		defer mu.Unlock()
		copied := make([]string, len(args))
		copy(copied, args)
		captured = copied
		return nil
	}
	commandOutput = func(_ context.Context, name string, args ...string) (string, error) {
		if name == "docker" && len(args) >= 3 && args[0] == "inspect" &&
			strings.Contains(args[2], "{{.Architecture}}") {
			return "amd64\n", nil
		}
		return "", fmt.Errorf("unexpected commandOutput: %s %v", name, args)
	}
	t.Cleanup(func() {
		runCommand = origRun
		commandOutput = origOut
	})

	r := &runner{
		logger: log.New(&logBuf, "", 0),
		cfg: config{
			shareDir:         shareDir,
			privateDir:       privateDir,
			logDir:           logDir,
			cfgDir:           cfgDir,
			targetImage:      "example/target:latest",
			targetContainer:  "leash-target-nonet",
			callerDir:        shareDir,
			listenCfg:        listen.Default(),
			bootstrapTimeout: 30 * time.Second,
		},
		opts: options{
			command: []string{"opencode"},
		},
	}

	if err := r.launchTargetContainer(context.Background(), "SIGTERM"); err != nil {
		t.Fatalf("launch failed: %v", err)
	}

	mu.Lock()
	args := captured
	mu.Unlock()

	for _, arg := range args {
		if arg == "--network" {
			t.Fatalf("expected no --network flag when opts.network is empty, got args: %v", args)
		}
	}
}
```

**Step 2: Run tests to verify they fail**

```
go test ./internal/runner/... -run TestLaunchTargetContainerWith -v
```

Expected: FAIL — `--network` arg not emitted yet.

**Step 3: Add `--network` to `launchTargetContainer`**

In `runner.go`, after the port publish block (after the
`for _, ps := range r.opts.publishes` loop, around line 1838), add:

```go
if r.opts.network != "" {
    args = append(args, "--network", r.opts.network)
}
```

**Step 4: Run tests**

```
go test ./internal/runner/... -run TestLaunchTargetContainerWith -v
```

Expected: PASS (2 tests).

**Step 5: Run full suite**

```
go test ./internal/runner/...
```

Expected: all pass.

**Step 6: Commit and push**

Use the creative-commits skill to commit, then push:

```bash
git add internal/runner/runner.go internal/runner/mounts_test.go && git commit && git push
```

---

### Task 4: Update `usage()` to document the flag and env var

**Files:**
- Modify: `internal/runner/runner.go` (the `usage()` function, around lines 300–338)

**Step 1: Add to the Flags section**

After the `-P, --publish-all` line, add:

```
  --network <name>                Attach the coder container to a named Docker network.
```

**Step 2: Add to the Environment variables section**

After the `LEASH_LISTEN` line, add:

```
  LEASH_NETWORK                Attach the coder container to a named Docker network (overridden by --network).
```

**Step 3: Run full suite to confirm nothing broken**

```
go test ./internal/runner/...
```

Expected: all pass.

**Step 4: Manually verify the flag appears in help output**

```
go run ./cmd/leash --help 2>&1 | grep -i network
```

Expected: both `--network` and `LEASH_NETWORK` appear.

**Step 5: Commit and push**

Use the creative-commits skill to commit, then push:

```bash
git add internal/runner/runner.go && git commit && git push
```

---

### Task 5: Final verification

**Step 1: Run the full test suite**

```
go test ./...
```

Expected: all pass.

**Step 2: Verify flag appears in help**

```
go run ./cmd/leash --help 2>&1 | grep -i network
```

Expected output includes both:
- `--network <name>`
- `LEASH_NETWORK`
