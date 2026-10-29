# The tpm2-trust workspace

Each directory here is a project with a repository of its own: aSSH
(`assh/`), tpm2-kira (`tpm2-kira/`), the vault (`tpm2-kira-vault/`), Marify
(`marify/`), safearch (`safearch/`), tpm2-verifier (`tpm2-verifier/`,
independent of the others: not in `go.work`, pins nothing), go-tpm
(`go-tpm/`, the fork), mkinitcpio-systemd-extras. This repository holds only what ties them
together: `go.work`, `pin`, and these rules. Each project's own AGENTS.md
applies inside it.

- **Edit a project in its checkout here, never in a copy of it inside
  another.** `tpm2-kira/assh` and `safearch/assh` are submodules: pins of a
  commit of `assh/`, from which releases are built. A change to aSSH is
  made and committed in `assh/`; `./pin assh` then moves both submodules to
  it and stages the new pointers, which are committed in tpm2-kira and
  safearch. Go refuses to build inside a submodule here (it is not in
  `go.work`), which is the reminder.
- **Development builds use the checkouts, releases the pins.** `go.work`
  makes `go build` and `go test` in assh, tpm2-kira and the vault use each
  other's checkouts, so a change reaches the others before it is committed.
  Every release build sets `GOWORK=off` and builds the pinned commits:
  tpm2-kira's PKGBUILD and debian/rules, safearch's `lib/assh.sh`, Marify's
  `build-core.sh` (from a fresh archive). A new release build does the same.
- **`./pin status` before a release, and after a change to a project
  others use.** It shows every pin against the checkout it pins: the aSSH
  submodules, Marify's `core.properties` (the tpm2-kira commit the app
  links; `./pin tpm2-kira`), the vault's `replace ../tpm2-kira` (always the
  checkout), and go-tpm, which tpm2-kira and aSSH pin by intent and `pin`
  shows without moving.
- **Pinning is staging.** `pin` never commits; the commit of a new pin is
  the consuming project's, with its tests run against it.
