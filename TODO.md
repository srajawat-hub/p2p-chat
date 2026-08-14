# p2p-chat — TODO

Goal: make this repo defensible as proof-of-work for a Waku / Status / IFT
application. Order matters. Do not add features before Tier 1 is closed.

Weekend track only (~5-6 hrs/week). Must not eat the weekday SD/DSA slot.

---

## Tier 1 — correctness and defensibility (before any new feature)

### T1.1 Close the four crypto explain-gates  ⬜
No code. Spoken or written answers, then paste them into `LEARNING-LOG.md`.
Asked three times already (log entry 2026-08-13). This is the exact Tria
failure mode: working code, explanation collapses under follow-up.

1. `DH` returns an error — what must a hostile peer send to trigger it, and
   what breaks if it is ignored?
2. Why does `kdf` need an `info` string when the secret is already unique
   per conversation?
3. Eve steals Bob's chain key at message 500 — exactly which messages can she
   read, which are safe, and why?
4. Why is the all-zero nonce in `Encrypt` safe here when it is a textbook
   vulnerability anywhere else?

Bonus, also unanswered: why two ratchets (symmetric + DH) rather than one.

A reviewer reading `ratchet.go` asks #4 within a minute. Answer it first.

### T1.2 Fix the store cursor bug  ✅
Known broken, logged, currently masked by content-hash dedup.
Symptom: C restarts and refetches its own messages (`new 5` where 3 was right).

Two sub-fixes:
- Persist the store to disk (today it is in-memory, so a restart queries
  `since: 0`).
- Replace the timestamp cursor with an **opaque per-peer `nextCursor` token**
  that the client stores and never interprets.

The reason is the interview-grade part: `Timestamp` is the RECEIVER's clock
(deliberate — defends against lying senders), so C's clock compared against
B's clock drifts and drops or duplicates at the boundary. go-waku does exactly
this, see `company-prep/IFT/go-waku` `store/criteria.go`.

Shipping a known-wrong sync cursor to the team that wrote the correct one is
the biggest single risk in this repo.

### T1.3 Persist ratchet session state  ✅
`sessions.go:39` already admits it: state is in-memory, so a restart loses
every session and ciphertext stored before the restart becomes permanently
unreadable. That quietly defeats store-and-forward, the headline feature.

The interesting sub-problem to write up, not just implement: writing key
material to disk destroys forward secrecy unless it is encrypted at rest. Need
a decision, not a casual `os.WriteFile`.

---

## Tier 2 — the one new feature worth building

### T2.1 Content topics + light-client filter protocol  ✅  ★ highest value
Before this task, `AddressedEnvelope.To` carried a cleartext peer ID.
`sessions.go:26` named this as the social-graph leak and said "Waku's answer is
to address by TOPIC." This task builds that answer.

- Derive a content topic from the pairwise session instead of a plaintext
  recipient.
- Add `/chat/filter/1.0.0`: a light node subscribes to specific content topics
  from a full node instead of joining gossipsub.
- Show the trade-off empirically: a node subscribing to exactly one topic has
  told the full node what it cares about; a node taking everything has not.

This is Waku's real architecture (relay / store / filter / lightpush), it
fixes a leak already documented in the code, and it shows understanding of
*why* the split exists.

### T2.2 X3DH prekey bundles  ⬜  (runner-up)
`sessions.go:14` names the gap: both nodes must be online simultaneously at
least once, which is absurd in a store-and-forward system. Publishing a prekey
bundle to the store lets Alice open a session with an offline Bob. Same shape
as store-and-forward, so it composes with what exists.

### T2.3 RLN  ⛔ deliberately NOT doing
Flashiest Waku feature, but a zk rabbit hole that would eat every weekend and
end up carried undefended. The scoping decision is itself the interview story.

---

## Tier 3 — the 100-node demo and presentation

### T3.1 100-node in-process swarm demo  ⬜
Replaces the 3-node manual walkthrough as the headline demo. NOT 100
terminals — one Go process (a test or `cmd/swarm`) spawning 100 libp2p hosts
in-process, wired into a sparse random topology (each node dials ~3 others),
never a full mesh.

What 100 nodes proves that 3 cannot:
- Gossipsub actually works as a routing protocol, not a 3-node echo. Measure
  hop count and how many nodes a message reaches.
- **Message amplification** — the real finding. Pairwise sessions mean N
  ciphertexts per send (`EncryptToAll`). At N=100 that is 100 copies published,
  each gossiped to the whole network. This is the honest quantified argument
  for why real group messaging uses sender keys or MLS. Chart it: wire bytes vs
  N, at N = 3, 10, 50, 100.
- **Store-and-forward under churn** — randomly kill and restart 20% of nodes
  during the run, assert every surviving node converges to the same store
  count. This is the property test the 3-node demo can only gesture at.
- Partition and heal — split the topology in two, send on both sides, rejoin,
  assert convergence.

Assert, do not just print: final store counts equal across nodes, zero
decrypt failures at the intended recipients, relays never hold plaintext.

Note this depends on T1.3 (ratchet persistence) for the churn test to mean
anything, and will likely surface the T1.2 cursor bug loudly. Good — that is
the point of running it.

Practical: 100 hosts in one process is fine, but use in-memory or loopback
transport, raise the fd limit if needed, and keep the run under ~60s.

### T3.2 `./demo.sh` + asciinema recording  ⬜
One command a reviewer runs in 90 seconds. Reuse the scripted 3-node harness
from Stage 3 for the narrative version, and the swarm for the scale version.
Lead with the money shot already in `main.go:145`:
`[opaque] N bytes from <peer>, not for me` — the relay storing what it cannot
read, while `[stored: N]` climbs.

### T3.3 ARCHITECTURE.md  ⬜
Diagram plus a "what I deliberately did not build, and why" section (RLN,
discovery hardening, light-client protocols). The scoping decision is the
interview story, but only if a reviewer sees it before opening any Go file.

---

## Suggested schedule

| Weekend | Work |
|---|---|
| 1 | T1.1 explain-gates (morning, no code) + T1.2 cursor and store persistence |
| 2 | T1.3 ratchet persistence + T3.2 demo script + T3.3 ARCHITECTURE.md |
| 3 | T3.1 100-node swarm |
| 4+ | T2.1 content topics and filter protocol |
