---
name: Docs Sync
description: On a production release in a source repo, read the released code at a pinned SHA and open a documentation PR in this repository.
emoji: "📘"

on:
  repository_dispatch:
    types: [docs-sync]

  # repository_dispatch is NOT one of gh-aw's "safe events", so the pre-activation
  # membership check runs against github.actor. The dispatch arrives as the app bot,
  # which is not a team member, so it must be allow-listed here or every run is
  # silently rejected. This string must match the App slug from IP-2861 exactly.
  bots:
    - "molecule-docs-sync[bot]"

  # Pilot expiry (IP-2867). Baked to an absolute UTC timestamp at FIRST compile and
  # then preserved; editing this value alone does nothing. To extend deliberately:
  #   gh aw compile --refresh-stop-time
  stop-after: "+14d"

  permissions:
    contents: read

  # Deterministic relevance gate. Runs in pre_activation, before the agent job
  # exists, so an irrelevant release costs zero AI credits. Most releases are
  # internal and stop here.
  #
  # NB: skip-if-match cannot do this — it evaluates GitHub *search queries*, not
  # changed paths.
  steps:
    - name: Mint source-read token for the gate
      id: gate-token
      uses: actions/create-github-app-token@v3
      with:
        client-id: ${{ vars.DOCS_SYNC_APP_CLIENT_ID }}
        private-key: ${{ secrets.DOCS_SYNC_APP_KEY }}
        owner: moleculeprotocol
        repositories: ${{ github.event.client_payload.repo_name }}

    - name: Docs relevance gate
      id: relevance
      continue-on-error: true
      env:
        GH_TOKEN: ${{ steps.gate-token.outputs.token }}
        SRC_REPO: ${{ github.event.client_payload.repo }}
        SRC_SHA: ${{ github.event.client_payload.sha }}
        BASE_SHA: ${{ github.event.client_payload.base_sha }}
        PREVIOUS_VERSION: ${{ github.event.client_payload.previous_version }}
      run: |
        set -euo pipefail

        BASE="${BASE_SHA:-$PREVIOUS_VERSION}"
        if [ -z "$BASE" ]; then
          echo "::warning::No base reference in the payload; cannot diff. Stopping."
          exit 1
        fi

        # One compare call, no clone. Paths mirror the source-of-truth map in
        # .github/prompts/docs-sync.md — keep the two in step.
        CHANGED=$(gh api --paginate \
          "repos/${SRC_REPO}/compare/${BASE}...${SRC_SHA}" \
          --jq '.files[].filename' 2>/dev/null || true)

        if [ -z "$CHANGED" ]; then
          echo "::warning::Could not list changed files for ${BASE}...${SRC_SHA}. Stopping."
          exit 1
        fi

        # Hand-authored contract surfaces only. graphql/autogen and prisma/generated
        # are build artefacts and must never trigger a docs run.
        RELEVANT=$(printf '%s\n' "$CHANGED" | grep -E \
          -e '^graphql/schemas/' \
          -e '^prisma/schema\.prisma$' \
          -e '^lambda/(appsync-resolver-labs-lambda|appsync-resolver-evm-tokenization|appsync-resolver-lit-service|desci-hubs-auth-lambda|x402-gateway-lambda|kamu-client-lambda|did-linking-worker|labnft-metadata-lambda|ocl-processor)/' \
          -e '^lib/(shared-api-stack|evm-tokenization-service-stack|encryption-stack)\.ts$' \
          | grep -v -e '^graphql/autogen/' -e '^prisma/generated/' || true)

        COUNT=$(printf '%s' "$RELEVANT" | grep -c . || true)
        if [ "${COUNT:-0}" -eq 0 ]; then
          echo "::notice::Release touched no documented surface; stopping before any AI spend."
          exit 1
        fi

        echo "Doc-relevant paths changed ($COUNT):"
        printf '%s\n' "$RELEVANT"

# Skips activation and the agent when the gate exits non-zero. The run stays green.
if: ${{ needs.pre_activation.outputs.relevance_result == 'success' }}

engine: claude
timeout-minutes: 20

# Explicit, because max-ai-credits silently defaults to 1000 when omitted.
max-ai-credits: 600
max-daily-ai-credits: 3000

# The agent job is read-only. Strict mode rejects a write permission here; all
# writes happen in the separate, sanitised safe-outputs job.
permissions:
  contents: read

network:
  allowed: [defaults]

# The prompt body may NOT reference github.event.client_payload.* — the compiler
# rejects those expressions. Bridge them through env and refer to the names.
env:
  SRC_REPO: ${{ github.event.client_payload.repo }}
  SRC_SHA: ${{ github.event.client_payload.sha }}
  BASE_SHA: ${{ github.event.client_payload.base_sha }}
  VERSION: ${{ github.event.client_payload.version }}
  PREVIOUS_VERSION: ${{ github.event.client_payload.previous_version }}
  RELEASE_URL: ${{ github.event.client_payload.release_url }}
  SOURCE_PR: ${{ github.event.client_payload.pr_number }}
  RELEASE_NOTES: ${{ github.event.client_payload.release_notes }}

checkout:
  - path: .
    current: true
  - repository: moleculeprotocol/desci-infra
    ref: ${{ github.event.client_payload.sha }}
    path: ./source
    # 0, not 1: the agent diffs base_sha..sha, which a shallow clone cannot do.
    fetch-depth: 0
    github-app:
      client-id: ${{ vars.DOCS_SYNC_APP_CLIENT_ID }}
      private-key: ${{ secrets.DOCS_SYNC_APP_KEY }}
      owner: moleculeprotocol
      repositories: [desci-infra]

tools:
  edit:
  bash: ["git diff", "git log", "git show", "git status", "ls", "cat", "rg"]

safe-outputs:
  create-pull-request:
    title-prefix: "[docs-sync] "
    labels: [documentation, automation]
    draft: true
    if-no-changes: ignore
    max: 1
    # DOCS-2 decision: create-pull-request cannot update an existing PR (it calls
    # pulls.create unconditionally, every run). A rolling sync PR is therefore not
    # achievable. close-older-pull-requests gives the property we actually wanted —
    # at most one open docs PR at a time — without force-push hazards.
    close-older-pull-requests: true
---

# Docs Sync

A production release of the source repository has shipped. The release is described by these
environment variables:

- `SRC_REPO`, `SRC_SHA` — the repo and the released commit
- `BASE_SHA`, `PREVIOUS_VERSION` — the previous release, your diff base
- `VERSION`, `RELEASE_URL`, `SOURCE_PR` — identifiers for the PR body
- `RELEASE_NOTES` — the release body; **frequently empty, do not depend on it**

The source repository is checked out **read-only** at `./source`, pinned to `SRC_SHA`. This
documentation repository is the working tree at the workspace root.

**Before you do anything else, read `.github/prompts/docs-sync.md` in this repository.** It is your
knowledge base: the page↔source map, the house style, the guardrails about what you may assert, the
release-notes rules, and the required PR body structure. Follow it exactly.

Then:

1. Diff the release: `git -C source diff <BASE_SHA> <SRC_SHA>`.
2. Update only the pages the map connects to the paths in that diff.
3. Add a release-notes entry only if the release changed something a consumer can observe.
4. Open one pull request whose body follows the contract in the knowledge base.

Do not modify anything under `./source`. Never edit `SUMMARY.md`.
