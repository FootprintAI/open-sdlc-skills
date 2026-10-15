# Contributing

Thanks for considering a contribution. These skills are prompts, not code —
which makes them easy to change and easy to change *badly*. The bar below
exists because a vague skill produces vague work.

## What's most useful

1. **A role that's missing** — a seat on a software team that none of the
   existing skills covers.
2. **A guardrail that misfired on a real repo** — the most valuable bug
   report here is "I ran `/qa-e2e-test` on my project and it did X, which
   was wrong because Y."
3. **Stack coverage** — a language, framework, or deploy target the
   architect/engineer/QA skills should know how to handle.
4. **Sharpening an existing skill** — replacing a vague instruction with a
   specific, checkable one.

## The bar for a skill

Every skill in this repo follows the same shape. A new or edited skill
should keep it:

- **Frontmatter**: `name`, `description`, `category`, `tags`. The
  `description` is what the model reads to decide whether the skill
  applies — it must say what the skill does *and when it applies*, in one
  sentence that can stand alone.
- **A role statement** — "Act as the X role…" — and an explicit statement
  of what the role does *not* own. Skills that don't say what they refuse
  to do end up doing everything.
- **Input** — what the user types, including modes if the skill has them.
- **Principles before steps.** The principles are the part that generalizes
  to situations the steps didn't anticipate.
- **Numbered steps** with concrete commands where commands exist.
- **A named artifact.** Every skill must produce something durable — a
  committed markdown doc, a GitHub issue, a PR, a published release. A
  skill whose only output is chat is not a skill.
- **Guardrails** — a closing list of what the skill must never do. Include
  what it must never *claim*: no skill should report success it cannot
  evidence.

## Style

- Evidence over assertion. Prefer "verified by running the tests" to
  "ensure tests pass."
- Say why, briefly, when a rule is non-obvious. The reasoning is what lets
  the model apply the rule to a case the rule didn't name.
- Confirm before anything hard to undo: bulk issue creation, scope cuts,
  force-pushes, production deploys, published releases.
- Keep prose wrapped at ~76 columns to match the existing files.
- No company-internal hostnames, URLs, tool names, or personal identifiers.
  This is a public repo; skills must run for someone who has never heard of
  us.

## Testing a change

Skills can't be unit-tested, so test them the only way that counts: install
the changed skill and run it against a real repository, then say so in the
PR — which repo, which command, what it produced, and where it still fell
short. A PR that says "ran `/architect-design` on a Rust project, the
sanctioned-languages section handled it like this" is worth ten that only
argue about wording.

## Submitting

1. Fork and branch.
2. One skill (or one coherent change) per PR.
3. In the PR description: what changed, why, and the evidence from your
   test run above.

By contributing, you agree your contributions are licensed under the
[Apache License 2.0](LICENSE).
