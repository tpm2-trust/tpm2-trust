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

- [x] 1. **The shared core as a module of its own** (decided: inside tpm2-kira), in tpm2-kira's
      repository: `core/` with its own go.mod
      (`github.com/mrwiora/tpm2-kira/core`), holding `attest/`,
      `transport/`, `vaultproto/`, `mobile/`. tpm2-kira requires it
      (`replace ./core`), `go.work` uses it; the vault and Marify require
      the core alone. Marify's build-core drops its go.mod edit. The
      module boundary keeps the core free of tpm2-kira's `internal/` and
      of aSSH (it cannot import what its go.mod does not require).
      Done: tpm2-kira 86340f0, the vault 6c8c112, Marify 95ea722, the
      workspace e9d75fd; tpm2-kira's and the core's suites pass, the vault's
      too; Marify's Android run against it - see below.
- [x] 2. **The vault pins the core**: as aSSH is pinned - a git submodule
      `tpm2-kira/` in the vault, its go.mod replacing the core with
      `./tpm2-kira/core`, moved by `./pin tpm2-kira` (with Marify's pin);
      `go.work` gives the checkout for development. (A pseudo-version
      would have Go fetch tpm2-kira from its remote, and nothing is pushed:
      open question 5.)
      Done: the vault 215d1b8, go.work replaces the core at
      v0.0.0 with the checkout; the vault's tests pass from the pin.
- [x] 3. **One EK root store and verifier**: tpm2-verifier's `verify`
      package and vendor data become what the phone core and tpm2-kira
      check EK chains with; the phone app keeps embedding the data at build
      time (nothing added at runtime). Decided: tpm2-verifier is the EK
      root store - the vendor roots are curated there from now on - as a
      package every project uses (the phone core, tpm2-kira, the vault,
      tpm2-verifier's command), its data signed. A TPM of a vendor not in
      the store is not untrusted: the result says "vendor not known" - a
      risk to weigh, not a refusal - until its vendor is added.
      Design (to confirm):
      - **Where:** a package in tpm2-verifier, `ekroots`, that embeds its
        vendor directory (`go:embed`) with the `MANIFEST` and
        `MANIFEST.sig` it already has, and checks them at load against the
        signing key compiled in (`manifest`, SSH signature, namespace
        `tpm2-verifier-vendors`): a file that is not in the signed
        manifest, or differs, is never used.
      - **API:** `ekroots.Check(chain) Result` - `Verified` (vendor and
        chain), `VendorUnknown` (no root of that vendor: information, a
        risk, the boot is not refused for it), `Failed` (a known vendor's
        chain that does not verify: refused). The same result in the
        phone, tpm2-kira's `phone ekcert`, the vault and tpm2-verifier's
        command.
      - **Users:** tpm2-kira's core (`attest/ekcert.go`, the phone through
        it) instead of `core/attest/ekroots/`, which goes; tpm2-verifier's
        command keeps reading a newer signed directory next to the binary,
        through the same package.
      - **Pinned like the others:** the core requires tpm2-verifier
        (open question 6).
      Done: tpm2-verifier 68fe206..804ab7e (`ekroots`, the data in
      `ekroots/vendors/`), tpm2-kira 69bcd7a, 31b83d0 (the core on
      `ekroots`, `ek_vendor_unknown`), the vault 3d0795c, Marify 3f7a63b
      (the bind screen), the workspace a935f5e. The signature: the
      mechanism checks `MANIFEST.sig` against `manifest/allowed_signers`,
      which names no key yet - open question 7.
- [x] 4. **One go-tpm**: tpm2-verifier on the same go-tpm as the others
      (`36f2059`), and `./pin status` warns when two projects' go-tpm differ.
      Done: tpm2-verifier 6537aad (its tests, swtpm included, pass),
      `pin status` compares the go-tpm of every Go module.
- [ ] 5. **`pin` reads its pins from a manifest** (`pins.conf`: consumer,
      kind - submodule, properties line, go.mod - target), so a new
      dependency is a line, not code; `pin status` lists them all.
- [ ] 6. **mkinitcpio-systemd-extras out of the workspace** (archived, or
      only kept as a reference): open question 4.
- [ ] 7. AGENTS.md (workspace and the projects') follow each step.

## Open questions

1. ~~Step 1: inside tpm2-kira or its own repository?~~ Decided: inside.
2. Step 3: tpm2-verifier is the EK root store (decided). Still open: its
   store is a signed directory read at runtime, and the phone must not take
   roots at runtime - so the phone core embeds tpm2-verifier's vendor data
   at build time (`go:embed` through a package of tpm2-verifier's), and
   tpm2-kira's `core/attest/ekroots/` goes. tpm2-verifier then becomes a
   dependency of the core (go-tpm and yaml): decided, acceptable.
3. ~~Step 4: go-tpm `36f2059` for tpm2-verifier too?~~ Decided: yes.
4. Step 6: remove `mkinitcpio-systemd-extras/` from the workspace, or keep
   it as an unrelated checkout?
6. Step 3: the core module would require tpm2-verifier. Today every pin
   between projects is a git submodule, because nothing is pushed; a
   submodule `tpm2-verifier/` inside `tpm2-kira/core/` works offline but
   makes Marify's core build fetch it too (the archive of a commit has no
   submodules). Pushing tpm2-verifier and tpm2-kira and pinning by Go
   pseudo-versions (`go.mod`, `go.sum`) is the plain Go way and needs no
   submodule anywhere - for the core, aSSH and the vault alike. Which?
7. Step 3: `manifest/allowed_signers` names no key and `MANIFEST.sig` does
   not exist, so the EK root data is checked against its manifest's hashes,
   not yet a signature. Whose ed25519 key signs it (`ssh-keygen -Y sign -n
   tpm2-verifier-vendors`)? Once its public key is in `allowed_signers`,
   `ekroots` uses signed data alone.
5. Step 7 (findings 7): which remotes, and when to push - the submodule
   pins only resolve for others once `assh/` is pushed.
