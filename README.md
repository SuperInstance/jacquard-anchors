# jacquard-anchors

Chain-head anchoring checkpoints for Jacquard logs — the witness
history for spec section 15 of
[SuperInstance/jacquard](https://github.com/SuperInstance/jacquard)
(`spec/SPEC.md`). This repository is deliberately **separate**
from the kernel repository: the witness history must not be the
witnessed history's own repo.

## What is here

One signed checkpoint per file, at
`checkpoints/<log_id>/<checkpoint_seq>.json` (`checkpoint_seq`
zero-padded to six digits). A checkpoint is a small JSON document
whose signed body states, for one log: the log's identity
(`log_id` = the hash of its genesis entry), a prefix length
(`log_length`), the hash of the entry at that length
(`head_hash`), and the spec version (`jacquard/0.5`). The full
format — body keys, envelope, verification procedure, honest
limits — is spec section 15 in the kernel repository.

## Operator public key

Checkpoints in this repository are signed with Ed25519 under this
operator key (raw 32-byte public key, lowercase hex):

```
1aa240acef0058b732e6093e3728e65a6d95a4178f77664ea92772bfb3072be5
```

The same hex is in `pubkey.txt` at the repository root, one line,
for tools. A public key is trusted because it is pinned through a
channel you already trust — in a real dispute, confirm this hex
with the operator directly; do not trust a key merely because a
checkpoint envelope carries it. The corresponding private key is
never in any repository.

Key rotation, if it happens, will be recorded here as a note
naming the old key revoked, the new key, and the `checkpoint_seq`
(per `log_id`) from which the new key signs. No rotation has
occurred.

## Publisher discipline

This repository is append-only by discipline, and its history is
the evidence: the publisher only ever **adds** commits — one
commit per checkpoint, adding exactly that checkpoint's file (the
first commit adds this README and `pubkey.txt`) — and never
amends, rebases or force-pushes. If you mirror this repository,
a silent rewrite of published checkpoints becomes detectable in
exactly the way a silent rewrite of a log is not. Weigh any
deviation from the append-only shape accordingly.

## How to verify a log

You need: a clone of this repository, spec section 15, the
presented log file, and the kernel repository (for the reference
verifier, `tools/jqanchor.py` — the verifier needs no private
key and no kernel state). From the kernel repository root:

```
python3 tools/jqanchor.py verify <presented-log.jsonl> \
    --checkpoints <path-to-this-repo-clone> \
    --pubkey 1aa240acef0058b732e6093e3728e65a6d95a4178f77664ea92772bfb3072be5
```

The command prints a JSON result and exits 0, 1 or 3. Outcomes
(spec 15.5): `verified` (fully anchored, tail 0),
`verified_to_anchor` (anchored prefix verified; `anchored_length`
and `unanchored_tail` reported as distinct numbers — an extension
of an anchored log is this, not a forgery), `unanchored` (exit 3;
nothing here witnesses that log), `failed` (exit 1; the JSON
names the first divergent checkpoint, or the history / internal
stage that failed). Step-by-step drill, including the forged-log
exercise: `docs/anchor-drill.md` in the kernel repository.
