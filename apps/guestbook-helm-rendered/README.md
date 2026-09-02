# guestbook-helm-rendered — Helm chart rendered into hydrated branches

Completes the pattern matrix: Helm as the templating tool, rendered manifests
as the delivery mechanism. `main` holds the chart and per-env values; on every
promotion Kargo:

1. clones the sources at the Freight's commit,
2. runs `helm-template` with `env/<stage>/values.yaml`, pinning the Freight's
   image tag via `setValues`,
3. force-pushes the rendered YAML to the branch
   `rendered/guestbook-helm-rendered/<stage>`,
4. syncs the Argo CD Application tracking that branch.

**Pipeline:** `Warehouse → dev → staging → prod`

**Why choose this pattern:** you keep Helm's authoring ergonomics but Argo CD
only ever sees plain YAML — no chart rendering on the control plane, fully
diff-able environment state, and identical mechanics to what powers ephemeral
PR-preview environments and templated team golden paths. If you plan to grow
into those, start here.

**Things to know**

- Until a stage's first promotion, its Application reports a comparison error —
  the rendered branch doesn't exist yet. Promote in the Kargo UI to fix.
- The Warehouse watches the image *and* this app's chart/values on main, so
  editing `env/prod/values.yaml` also produces promotable Freight.
- Kargo needs git *write* credentials for this project to push the rendered
  branches — see the repo root README, step 5.
