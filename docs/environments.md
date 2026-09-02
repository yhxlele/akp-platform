# Environments, stages, and where rendered manifests live

Two of the four example apps promote by pushing **rendered manifests** to a
branch. If you have read that branch-based environments are an anti-pattern,
that will look wrong. It isn't — but the reason is worth understanding before
you copy the pattern into a real repo.

## There are two trees, not one

| | Authored tree | Rendered output |
|---|---|---|
| Lives on | `main` | `rendered/<app>/<stage>` branches |
| Contains | Kustomize bases + overlays, Helm charts + values | plain YAML, no templating |
| Written by | people, via pull request | Kargo promotions, force-pushed |
| Owned by | app teams (see `.github/CODEOWNERS`) | nobody — it is generated |
| Reviewed as | the change you intended | the change that will actually apply |

They are separate trees with different requirements, so they do not have to
share a layout. In this repo the authored tree nests environments *under* each
app:

```
apps/orders/
├── base/                  # or chart/ for Helm
└── env/
    ├── dev/
    ├── staging/
    └── prod/
```

## `env/` vs `rendered/`

The two words are not interchangeable here:

- **`env/`** is a directory in the authored tree — the overlay a promotion
  renders **from**.
- **`rendered/`** is a branch prefix — the storage a promotion renders **to**.

So `apps/orders/env/prod/` is a Kustomize overlay you edit, and
`rendered/orders/prod` is a branch full of generated YAML you never edit.

## A hydrated branch is not an environment branch

The pattern to avoid is long-lived `dev`, `staging`, and `prod` branches that
you **merge or cherry-pick between**. They drift: a hotfix lands in `prod`
and never reaches `dev`, or a merge drags along changes nobody meant to ship.
The branch becomes a source of truth that no single commit describes.

A hydrated branch has none of those properties:

- Nothing is ever merged into it. Each promotion force-pushes a fresh render.
- Nobody edits it. Every byte is generated.
- It is fully reproducible from `main` plus a Freight reference.
- Deleting it loses nothing — the next promotion recreates it.

That is a build artifact, not an environment. The similarity is only in the
shape of the branch names.

## Think of these branches as storage

Kargo maintainer guidance frames it this way: rendered manifests may as well
be in an object-storage bucket. Git is chosen because Argo CD already reads
git, not because branches are meaningful here.

Once branches are storage, **branch count stops being a cost** — the same way
the number of keys in a bucket isn't one. That matters, because
`apps × stages` branches is the objection people raise first, and it is the
wrong thing to optimise.

## Why one branch per app per stage

Every branch naming convention the Kargo maintainers suggest carries **both**
the app and the stage, so one branch per app per stage is the endorsed
cardinality rather than a compromise. It also buys three concrete properties:

- **One writer per branch.** Promotions never race each other for a push.
- **`git-clear` is safe on the whole checkout.** The promotion can wipe its
  output branch and re-render from scratch, because the branch holds exactly
  one app's one stage. Share a branch between apps and this step becomes
  destructive.
- **Small clones.** A promotion checks out only its own output.

Consolidating branches gives all three up to reduce a number that the storage
framing says isn't a cost.

## Why the `rendered/` prefix

The real cost of many branches is human: a branch picker with thirty entries
interleaved among feature branches. A shared prefix fixes that without
touching topology — every rendered branch collapses into one filterable group.

The prefix also gives you globs for the two things you will want to automate:

| Target | Pattern | Matches | Does not match |
|---|---|---|---|
| Stage branches | `rendered/*/*` | `rendered/orders/prod` | `rendered/orders/review/1234` |
| Preview branches | `rendered/*/review/**` | `rendered/orders/review/1234` | `rendered/orders/prod` |

This works because [GitHub ruleset patterns don't match `/` with a single
`*`](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository#using-fnmatch-syntax)
— a preview branch has one segment more than a stage branch, so depth alone
separates the permanent from the disposable.

> **The two matchers disagree.** `git ls-remote` uses `fnmatch` *without* the
> pathname flag, so at the CLI `*` **does** cross `/`, and `rendered/*/*`
> would also match previews. Use segment depth for GitHub rules; anchor CLI
> sweeps on the literal `review/`:
>
> ```sh
> git ls-remote --heads origin 'rendered/*/review/*' | sed 's#.*refs/heads/##'
> ```

`rendered/<project>/review/<pr-id>` is a reserved namespace — this quickstart
does not ship a preview pattern, but Kargo PR workflows create transient
branches, and giving them a home now avoids renaming later.

## Branch protection: push restriction, not approval

Promotions **force-push**. So on a rendered branch you cannot require pull
requests or block force-pushes — those rules break promotions outright.

What a rule on `rendered/*/*` can do is restrict *who* pushes: only your Kargo
instance's identity, so nobody hand-edits production manifests. The approval
gate for a release belongs in the Kargo Stage — manual promotion, or a
verification step — not in git.

## Rendered output belongs in this repo, not your app repo

Rendered manifests are deployment state, not application source. Argo CD and
Kargo read them; application developers do not.

The practical argument is whose branch list pays the cost. Developers open the
app repo every day, so generated-YAML branches there are in the way of the
people least equipped to explain them. This repo is browsed by people who
already know what those branches are, and it already holds the Kargo
credentials and the Argo CD Applications that read them.

## Alternatives

All four options put plain manifests at a known location, and an Argo CD
Application can target any of them. They differ in what they cost.

| | Extra branches | Contention | Main trade-off |
|---|---|---|---|
| **`rendered/<app>/<stage>` branches** (this repo) | `apps × stages` | none | branch list grows |
| `rendered/` directory on `main` | none | every promotion **and** every human commit | Freight-loop risk; `main` history fills with promotions |
| One shared `rendered` branch mirroring `main` | 1 | all promotions | `git-clear` must be path-scoped or it wipes other apps |
| A dedicated rendered repo | in another repo | none | one more repo to fork, credential, and personalize |

### Rendering into `main` instead of a branch

This is the option people ask about most, and Kargo maintainer guidance
supports it while noting it is the *second* preference. Output goes to a
top-level directory:

```
├── apps/
│   └── orders/                # Warehouse subscribes here
│       ├── base/
│       └── env/dev|staging|prod/
└── rendered/                  # promotions write here — never subscribed
    └── orders/
        ├── dev/manifests.yaml
        ├── staging/manifests.yaml
        └── prod/manifests.yaml
```

**The rule that makes it work:** `rendered/` must sit outside every
Warehouse's `includePaths`. If a Warehouse subscribes to a path that
promotions write, each promotion produces new Freight, which triggers another
promotion — forever. This repo's commit-to-main apps avoid the same trap by
subscribing to the image only; see the comments in
[`apps/guestbook-kustomize/kargo/warehouse.yaml`](../apps/guestbook-kustomize/kargo/warehouse.yaml).

The costs: promotions now contend with human commits on `main`, `main`'s
history fills with promotion commits, and `git-clear` has to be scoped to one
path instead of the whole checkout.

If this layout appeals to you, read the next option first — it is the same
directory tree without the two drawbacks that come from putting it on `main`.

### One shared `rendered` branch that mirrors `main`

The same idea as the previous option — every app and stage as *directories*
rather than branches — but the tree lives on a single dedicated branch instead
of on `main`. In the simplest form, `rendered` is a mirror of `main` in
rendered form, following the same directory structure:

```
main                          rendered
├── apps/orders/              └── dev/
│   ├── base/                     ├── orders/manifests.yaml
│   └── env/dev|staging|prod/     └── billing/manifests.yaml
└── apps/billing/             staging/ …
                              prod/ …
```

This **strictly dominates rendering into `main`** on the two things that make
that option unattractive: promotions never collide with human commits, and no
Warehouse can accidentally subscribe to a path that promotions write, so the
Freight-loop trap disappears. If your instinct is "directories, not branches,"
this is the version to reach for.

Two further properties worth knowing:

- **No SHA pinning.** Every Application can track the tip of `rendered` and
  select its environment by path, rather than being pinned to the exact commit
  a promotion produced. (The per-stage-branch apps in this repo do pin, via
  `desiredRevision` in their `argocd-update` step.)
- **Directory order is free.** Because the tree is generated, `<env>/<app>`
  and `<app>/<env>` cost the same to produce. Prefer `<env>/<app>` if your
  environment names are consistent across apps and you want to diff a whole
  environment at once; `<app>/<env>` if apps have divergent stage sets.

The costs are real, though:

- **All promotions contend on one branch.** Every push is a fast-forward on a
  tip that another promotion may have just moved, so promotions retry.
- **`git-clear` must be path-scoped.** Clearing the whole checkout erases
  every other app's output, so the promotion must clear only its own
  directory — and that means it can only re-render its own paths. Re-rendering
  the entire tree on every promotion is the alternative, and it gets expensive
  as directories multiply.

### A dedicated rendered repo

Takes the storage framing literally: a separate repository whose only content
is rendered output. Branch count there is irrelevant because nobody browses it
interactively, and this repo's branch list stays clean. The cost is another
repo to fork, credential, and personalize — which is why the quickstart does
not ship it, and why it is often the right answer once you have real apps.

## What not to do

**Don't hoist environments to a top-level directory in the authored tree.**
A layout like `environments/prod/orders/` scatters one app's files across N
paths, breaks the one-directory-per-app onboarding this repo is built on, and
makes `CODEOWNERS` awkward. Keep environments nested under the app; if you
want a top-level environment view, that is what the rendered tree gives you.

**Don't consolidate branches to make the list shorter.** It trades away
single-writer promotions, safe `git-clear`, and small clones for a number that
doesn't matter. Use the prefix instead.

**Don't invent a new branch convention per app.** Any consistent scheme works,
but mixing several across projects makes protection rules and cleanup globs
impossible to write.

## See also

- [`apps/guestbook-rendered/`](../apps/guestbook-rendered/) — Kustomize →
  rendered branch
- [`apps/guestbook-helm-rendered/`](../apps/guestbook-helm-rendered/) — Helm →
  rendered branch, and the base to grow PR previews from
- [`docs/onboarding.md`](onboarding.md) — the naming conventions for adding an
  app
- [Kargo docs: git subscription path filtering](https://docs.kargo.io/concepts#git-subscription-path-filtering)
