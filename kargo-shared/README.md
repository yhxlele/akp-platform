# kargo-shared — custom promotion steps

Kargo ships a large set of built-in promotion steps (`git-clone`, `git-commit`,
`kustomize-build`, `helm-template`, `argocd-update`, …). A
**`CustomPromotionStep`** is the escape hatch for everything else: it runs a
container you choose, so a promotion can call a migration tool, a policy engine,
a scanner, or your own script.

These live at the repo root rather than under `apps/<name>/kargo/` because they
are **cluster-scoped** — define one here and every Kargo Project can `uses:` it.
`bootstrap/kargo-shared.yaml` syncs this directory to the Kargo control plane.

> **Requires Akuity-hosted Kargo.** These use the `ee.kargo.akuity.io` API
> group. On a Kargo without it, the `kargo-shared` Application reports
> `no matches for kind CustomPromotionStep`. Nothing else in the repo is
> affected — the four demo apps promote normally either way.

> **Not to be confused with `kargo-shared-resources`**, which the main README
> mentions under git credentials. That is a Kargo *namespace* designated as a
> global credential source. This is a *directory* of promotion steps. Similar
> names, unrelated mechanisms.

## Not the same thing as a PromotionTask

| | `PromotionTask` (`apps/*/kargo/tasks.yaml`) | `CustomPromotionStep` (here) |
|---|---|---|
| What it is | a named, reusable **sequence of existing steps** | **one new step**, backed by a container |
| Scope | namespaced to one Kargo Project | cluster-scoped, shared by all Projects |
| Reach for it when | several Stages repeat the same promotion recipe | no built-in step can do the thing |

## What's here

| Step | Does | Output |
|---|---|---|
| `hello-world` | echoes a message you pass in | writes `message=…`, but its `output:` block is left commented |
| `random-number` | picks a number in `1..range` | `result` — declared via `output.source.type: Pipe` |

Both run `busybox:1.37.0`. They exist to prove the mechanism end to end, so you
can copy one and swap the image and script for something real.

## Using one

Add it to any `promotionTemplate` or `PromotionTask` `steps:` list. `as:` names
the step so you can reference its outputs; `config:` becomes `${{ config.* }}`
inside the container.

```yaml
- uses: random-number
  as: rng
  config:
    range: 100
- uses: hello-world
  as: report
  config:
    message: "rng said ${{ outputs.rng.result }}"
```

`apps/guestbook-kustomize/kargo/tasks.yaml` carries exactly this block,
commented out — uncomment it and promote to see the steps run.

## Returning outputs

Two ways, both visible in the files here:

- **`Pipe`** (`random-number`) — write `key=value` to `$KARGO_OUTPUT`. Read it
  back as `${{ outputs.<as>.<key> }}`.
- **`File`** (shown commented in `hello-world`) — write JSON to a path and
  declare it:

  ```yaml
  output:
    source:
      path: /tmp/output.json
      type: File
  ```

A step needs no `output:` block at all if nothing consumes its result.

## Adding your own

1. Drop a new `<name>.yaml` in this directory. `metadata.name` is what `uses:`
   matches, and it must be unique cluster-wide. No `namespace`.
2. Commit and push. The `kargo-shared` Application is auto-synced, so it lands
   without a manual apply.
3. Keep the image pinned to a digest or exact tag — this runs inside every
   promotion that references it, so a floating tag makes promotions
   irreproducible.

Ported from the internal `sedemo-platform` repo, which has further examples
(Flyway migrations, Kyverno/OPA policy checks, Trivy and Grype scans, ORAS
push/resolve, Teams notifications).
