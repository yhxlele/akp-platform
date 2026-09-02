# guestbook-rendered — rendered manifests (hydrated branch) + Kustomize

The flagship pattern. `main` holds Kustomize sources (`base/` + `env/<stage>/`
overlays); **nothing on main ever contains a concrete image tag per
environment**. On every promotion, Kargo:

1. clones the sources at the Freight's commit,
2. pins the Freight's image tag into the stage overlay,
3. `kustomize build`s the overlay to plain YAML,
4. force-pushes the result to the branch `rendered/guestbook-rendered/<stage>`,
5. syncs the Argo CD Application tracking that branch.

**Pipeline:** `Warehouse → dev → staging (verified) → prod`

**Why choose this pattern:** what runs in each environment is always visible
as plain, diff-able YAML on its own branch — auditors and humans never need to
run kustomize/helm to know cluster state. Rollbacks are `git revert` on the
rendered branch or re-promoting old Freight.

**Things to know**

- Until a stage's first promotion, its Application reports a comparison error —
  the rendered branch doesn't exist yet. Promote in the Kargo UI to fix.
- The Warehouse watches the public `ghcr.io/akuity/guestbook` image (SemVer
  tags) *and* this app's directory on main, so both new images and manifest
  changes create Freight.
- `staging` runs a sample verification (`kargo/analysis.yaml`) gating
  promotion to `prod`.
- Kargo needs git *write* credentials for this project to push the rendered
  branches — see the repo root README, step 5.
