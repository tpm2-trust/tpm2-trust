# PLAN: the workspace's structure, as it grows

## Goal

The projects depend on each other more with every feature (aSSH under
tpm2-kira and safearch, tpm2-kira's core under the vault and Marify, three
go-tpm versions). Make every dependency explicit, pinned and cheap to
follow, and remove the duplicates, before more is built on top.

## Findings

1. **tpm2-kira's shared core is not a module of its own.** `attest/`,
   `transport/`, `vaultproto/` and `mobile/` depend on go-tpm and
   x/crypto alone, but the vault and Marify take them through the whole
   `github.com/mrwiora/tpm2-kira` module: its go.mod requires aSSH
   (replaced by a submodule), huh, go-attestation. Marify's core build has
   to edit that go.mod (drop aSSH) to build at all; the vault's module
   graph carries tpm2-kira's TUI libraries.
2. **The vault is not pinned.** `replace ../tpm2-kira` builds whatever the
   checkout is; every other consumer builds a pinned commit for a release.
3. **Two curated EK root stores.** tpm2-kira's `attest/ekroots/` (embedded
   into the phone app, Intel PTT) and tpm2-verifier's `vendors/` (a signed
   directory next to the binary, Intel) each pin vendor roots, with their
   own verification code.
4. **Three go-tpm versions.** tpm2-kira: upstream `36f2059` (the HMAC
   fix); aSSH: the same, plus a `replace` to the fork for `tpmadmin`;
   tpm2-verifier: v0.9.8.
5. **`pin` knows its pins by code.** Each new dependency means editing the
   script.
6. **`mkinitcpio-systemd-extras/` is used by nothing here** (tpm2-kira's
   own SSH server replaced its hooks).
7. **Nothing is pushed, and the workspace repository has no remote.** The
   submodules point at commits only these checkouts have.

## Steps (proposed - nothing is done before confirmation)

- [ ] 1. **The shared core as a module of its own** (decided: inside tpm2-kira), in tpm2-kira's
      repository: `core/` with its own go.mod
      (`github.com/mrwiora/tpm2-kira/core`), holding `attest/`,
      `transport/`, `vaultproto/`, `mobile/`. tpm2-kira requires it
      (`replace ./core`), `go.work` uses it; the vault and Marify require
      the core alone. Marify's build-core drops its go.mod edit. An import
      test keeps the core free of tpm2-kira's `internal/` and of aSSH.
- [ ] 2. **The vault pins the core**: a commit in the vault's go.mod (a
      pseudo-version of the core module, as tpm2-kira pins go-tpm), moved by
      `./pin core`; `go.work` gives the checkout for development.
- [ ] 3. **One EK root store and verifier**: tpm2-verifier's `verify`
      package and vendor data become what the phone core and tpm2-kira
      check EK chains with; the phone app keeps embedding the data at build
      time (nothing added at runtime). Needs (open question 2).
- [ ] 4. **One go-tpm**: tpm2-verifier on the same go-tpm as the others
      (`36f2059`), and `./pin status` warns when two projects' go-tpm differ.
- [ ] 5. **`pin` reads its pins from a manifest** (`pins.conf`: consumer,
      kind - submodule, properties line, go.mod - target), so a new
      dependency is a line, not code; `pin status` lists them all.
- [ ] 6. **mkinitcpio-systemd-extras out of the workspace** (archived, or
      only kept as a reference): open question 4.
- [ ] 7. AGENTS.md (workspace and the projects') follow each step.

## Open questions

1. Step 1: the core inside tpm2-kira's repository (a second module there,
   tagged `core/vX`), or a repository of its own (`tpm2-trust/kira-core`)?
   Inside keeps one history and one review; its own makes the boundary
   impossible to cross by accident. Recommended: inside.
2. Step 3: tpm2-verifier's root store is a signed directory read at
   runtime; the phone must not take roots at runtime. Is tpm2-verifier's
   `verify` package to be used as a library with the data embedded
   (`go:embed` of the same vendor files), and is tpm2-verifier the place
   where vendor roots are curated from now on?
3. Step 4: may tpm2-verifier move to go-tpm `36f2059` (an unreleased
   upstream commit), or should everything wait for the next go-tpm
   release?
4. Step 6: remove `mkinitcpio-systemd-extras/` from the workspace, or keep
   it as an unrelated checkout?
5. Step 7 (findings 7): which remotes, and when to push - the submodule
   pins only resolve for others once `assh/` is pushed.
