# Containarium bot — what goes where

Status: **draft**. `Containarium-cloud` is private, so the split below holds.

The bot is `@containarium <verb> <target>` — for example `start #123` or
`review #45`. It is wired in `Containarium-cloud`. The skills it runs live in
this repo. This doc sets the line between the two.

## 1. This repo (public, Apache 2.0)

Allowed:

- Skill contracts: what a role does, its inputs, its outputs, its evidence
  gates. Generic process.
- The user-facing command grammar (verb names, what each verb takes).
- Adapter docs that call **Containarium MCP tool names** as a backend, such
  as `_shared/tracker-containarium.md`. Tool names are already public via the
  MCP server.
- Synthetic examples: `acme-demo`, `example.com`, placeholder IDs.

Not allowed:

- Any real or working credential, token, key path, or secret name.
- Hostnames, IPs, daemon addresses, tenant or user names, or customer names.
- How the hosted control plane authorizes, caps, times out, or rejects a
  request. That is the map section 2 keeps private.
- Roadmap, pricing, competitor notes, or product strategy from real planning
  (PRDs, sprint plans, `product-define` output for real work).
- Output of a real run that contains any of the above.

## 2. Containarium-cloud (private)

Owns:

- Dispatch: which verb starts which skill, and under which model tier.
- Who may trigger a verb (collaborator check, write access, bot identity).
- Server-side enforcement: label allow-lists, depth and fan-out caps, claim
  takeover rules, sanitization of comment bodies, identity stamping.
- Rate limits, timeouts, and budget caps.
- Secret storage and the rule that the hosted plane never holds tenant
  plaintext.

Enforcement must hold even if someone has read these rules. Keeping them
private is defense in depth, not the defense itself.

## 3. Cleanup before the bot ships

Review these against the rules above. Do not move them without a decision:

- `_shared/tracker-containarium.md` — the "Resolve", "Verbs", and "Labels"
  sections describe enforcement details (allow-list gate, depth and fan-out
  caps, the 2-hour stale-claim window, the 501 on hosted secrets). Candidate
  to trim to tool names and the caller's contract; the enforcement detail
  moves to the cloud repo.
- `devops/containarium-env.md` — the secrets section names where tenant
  tokens live (`containarium secrets list <tenant> --server <daemon>`). Keep
  the check procedure; drop anything that reads like a real layout.

## 4. Rules for anyone writing to either repo

- Write the contract, not the mechanism. "Only collaborators may start a
  run" is fine here. The check itself belongs in the cloud repo.
- Before committing, scan the diff for hostnames, IPs, tokens, tenant names
  and customer names. Use synthetic values.
- If a doc describes a cloud behavior, mark which repo owns it.
- When unsure whether a detail is safe to publish, leave it out and ask.
