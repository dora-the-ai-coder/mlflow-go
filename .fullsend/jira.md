# Jira epic-to-code automation

Fullsend polls Red Hat Jira every 30 minutes for this candidate set:

```jql
project = RHAI AND issuetype = Epic AND labels = "fs-integration"
  AND statusCategory != Done ORDER BY updated DESC
```

The target is `opendatahub-io/mlflow-go`. There is no fixed issue key. Epics are
the coding work items; their children supply context, not separate dispatches.

## Triggering implementation

An RHAI user with a project role named `Developers` or `Administrators` can:

- Add `fs-integration` to an existing epic to start coding.
- Create an epic with that label to start coding.
- Comment `/fs-code` on a labeled epic to retry or request a new run.

The label is this repository's readiness signal; `ready-to-code` is not also
required. Ordinary comments and other label changes do not trigger coding.
The poller excludes its own service account, and the local trigger requires a
human with `write` or `admin` authorization. Done epics and unlabeled epics are
excluded. The agent rechecks the live epic before editing code.

The code agent reads the epic and its direct children through the inherited
read-only Jira provider, implements the SDK scope, runs checks, and commits
locally. Fullsend's post-script pushes a branch and creates a GitHub PR against
`main` with the Jira link. Human review and CI still govern merging. An epic
that needs clarification or spans other repositories can produce an explanation
instead of a code change; the agent must not invent a complete implementation.

## Activation

1. Complete the GitHub App, token mint, and OpenAI setup in [README.md](README.md).
   This workflow forwards the same inference secrets and uses the existing
   repository-scoped OpenAI identity variables. The code agent remains Pi with
   `openai/gpt-6.1-sol`.
2. Set `JIRA_TOKEN` and `JIRA_USER_EMAIL` as Actions secrets on
   `opendatahub-io/mlflow-go`. Use a dedicated Jira service account. The workflow
   fixes the tenant to `https://redhat.atlassian.net`; no Jira URL variable is
   required.
3. Confirm the account can browse the RHAI epics, comments, and child issues,
   including issue-security restrictions, and that the organization permits
   API-token access. Unlike a read-only connection check, polling writes Jira
   issue properties for locks and checkpoints. The pinned poller documents
   Developers and Administrators project roles for its service account so it
   can coordinate polls and inspect command authors' role membership.
4. Confirm RHAI's role names and membership with its administrators. The poller
   maps `Developers` to `write` and `Administrators` to `admin`; custom role
   names do not imply write access. This authorization uses Jira membership,
   not membership in the GitHub repository.
5. Merge to `main`. Scheduled runs use the default branch; manual runs also
   check out `main`. Fork workflows are blocked by the repository guard.
6. Add the label to a small, well-scoped RHAI epic, then run **Actions → Fullsend
   RHAI epic coding → Run workflow**, or wait for the scheduled poll. Verify the
   poll dispatches `code`, the sandbox reads Jira, and a linked GitHub PR is
   created with the expected scope and passing CI. This live smoke test has not
   been run as part of this change.

Credentials stay in Actions secrets and Fullsend's credential providers. Do not
put token values in YAML, Jira comments, workflow inputs, or PR descriptions.
The PR may contain implementation details derived from the epic, so only opt in
work appropriate for this public repository.

## Polling behavior and recovery

Fullsend detects events; it does not start a fresh run for every labeled epic
on every poll. Its per-repository Jira checkpoints suppress repeated processing.
The pinned poller only backfills 24 hours. An old labeled epic with no recent
qualifying activity will not automatically dispatch when this workflow is first
enabled: an authorized user should post `/fs-code` or remove and re-add the
label. The same applies to missed events after an outage longer than 24 hours.

Workflow-level concurrency serializes manual and scheduled discovery/dispatch.
An empty dispatch list skips the coding job; malformed poll output fails the
workflow. Polling checkpoints are not an exactly-once delivery guarantee. If
dispatch or coding fails after checkpointing, inspect the run and retry with a
fresh `/fs-code` comment. Check for an existing PR first. Removing the label
prevents future discovery but does not cancel an already running job; cancel
that Actions run explicitly if needed.

The default poller considers up to 50 recently updated candidates and processes
up to five per cycle. Monitor backlog and rate limits if the opted-in set grows.
GitHub can disable scheduled workflows after repository inactivity; check that
the workflow remains enabled. Jira authentication, project permissions, and
API-policy failures require operator attention.

## Pins and validation

The CLI installer and dispatcher retain Fullsend commit
`af8cd4756172ec105c3aa523515c9c7d51684e90`. The code harness explicitly pins
`fullsend-ai/agents` commit `f4acd0350dc9445112f4a0a6f781c02a31a64636` with a
SHA-256 digest. This harness includes the Jira pre-script, read-only provider,
and external-work-item-aware GitHub post-script. Older integration documentation
describing the missing Jira pre/post pipeline does not describe this pinned
agent implementation. The existing GitHub code stage also uses this code harness;
the Jira overlay and local epic skill apply only to Jira source events.

Validate configuration with `fullsend agent list --fullsend-dir .fullsend`,
workflow syntax with `actionlint .github/workflows/fullsend-poll-jira.yaml`, and
CEL routing with `fullsend dispatch --input-driver json --input-file <fixture>
--config-dir .fullsend`. Synthetic dispatch tests establish routing, not live
Jira connectivity, sandbox credential injection, or end-to-end PR creation.

Run the routing regression cases with the pinned CLI on your path:

```bash
python3 .fullsend/tests/test_jira_dispatch.py
```

Set `FULLSEND_BIN` to use an explicit binary path. The cases cover activation,
retry, ordinary comments, label removal, authorization, and repo/project/tenant
boundaries. The CLI may fetch the pinned public harness into `.fullsend-cache/`;
do not commit that generated cache.

References:

- [Reference poller](https://github.com/ralphbean/jira-code-test/blob/main/.github/workflows/fullsend-poll-jira.yaml)
- [Pinned Jira-capable code harness](https://github.com/fullsend-ai/agents/blob/f4acd0350dc9445112f4a0a6f781c02a31a64636/harness/code.yaml)
- [Pinned Jira poller guide and operational limits](https://github.com/fullsend-ai/fullsend/blob/af8cd4756172ec105c3aa523515c9c7d51684e90/docs/guides/user/jira-integration.md)
