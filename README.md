# Illumify CLI skills

The instructions a customer's coding agent reads when it runs `illumify skills get <name>`.

Three documents, and the CLI fetches them from here **live, on every call**. An edit to `main`
reaches every CLI that is not pinned to a tag, within about five minutes — GitHub's raw cache is the
only delay. There is no release step and nothing to publish.

| file | what it covers |
| --- | --- |
| `app.md` | what a valid storefront theme is: config, routing, `window.IllumifyStorefront`, the CSP |
| `data.md` | `@illumify/sdk`: the catalogue reads and the offer model |
| `deploy.md` | link, dev, build, upload, handoff, `.env`, the credential rules |

## How a CLI decides which version to read

For a binary at `X.Y.Z`, in order, first answer wins:

```
tag  X.Y.Z      exact pin for that release
tag  X.Y.0      the minor line
main            whatever is here now
```

So **untagged is the normal state**, and the normal state is that everyone reads `main`. Tags are the
exception: create one when a CLI release must keep reading the text that described it.

**Tag at the moment you make a breaking change, not afterwards.** Between the edit and the tag, every
older CLI is reading instructions about behaviour it does not have — and nothing surfaces that, because
the fetch succeeded. Late tagging is this design's failure mode and it is silent.

Tags here are expected to move. That is deliberate and differs from the usual convention, so the
branch protection on `main` does not extend to them.

## If this repository is unreachable

The CLI carries no copy of these files. A failed fetch — no ref answered, a rate limit, a DNS failure —
is an error that names the refs it tried and tells the agent to stop and report it rather than work
around it. That is deliberate: a stale compiled-in copy would read as instructions and be followed,
which is the failure this design exists to avoid.

## What must never appear here

This is public. The text has been scrubbed and the CLI has a test that fails if any of it comes back
— but that test only guards the copy inside the binary. **Edits made directly here do not pass
through it.** Review is the only gate on this side.

Never: real environment hostnames, our development stand's ports, internal service or class names,
API keys of any kind, customer names, or our own fixture ids. Examples use placeholders — a customer's
agent pastes what it reads, verbatim.

## Editing

These documents are executed, not skimmed. An agent will follow a sentence literally, so a claim that
is nearly true is worse here than in ordinary documentation: state the rule, the reason, and what
fails silently if it is broken.

The source of truth for the text is `skills/` in the CLI repository, where the content assertions
live. Changes made here reach customers faster; changes made there survive the next release. When
they diverge, the CLI's copy is the one that was tested.
