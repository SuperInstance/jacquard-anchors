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

## Operator public keys and the key registry

Checkpoints in this repository are signed with Ed25519. Which
key must have signed which checkpoint is stated by the **key
registry**, `keys.json` at the repository root (format
`jacquard/anchor-keys/2`, spec section 15.3 of the kernel
repository): per `log_id`, each key's `signs_from` /
`signs_until` windows partition the checkpoint sequence, and a
verifier refuses a checkpoint whose envelope key is not the
scheduled key. A key that names a `log_id` in neither map
holds a null window for that log — it signs nothing there. Pin the registry (and the keys in it) through a
channel you already trust — in a real dispute, confirm the hexes
with the operator directly; do not trust a key merely because a
checkpoint envelope carries it. The corresponding private keys
are never in any repository; from the 2026-11-02 rotation they
are passphrase-encrypted at rest on the operator's machine.

`pubkey.txt` at the repository root names the *current* active
key, one line, for tools:

```
ce805efd20de4b8a6f7bee1544c8db1173678cfbe19a3f9814315b48845f5072
```

### Rotation note — 2026-11-02 (spec 0.6, slice 21)

The original operator key

```
1aa240acef0058b732e6093e3728e65a6d95a4178f77664ea92772bfb3072be5
```

is **retired**. It signed checkpoints `checkpoint_seq` 0 and 1
for `log_id`
`5f4bf2b400dc3de3c6b4798028592650ca606facabf5c7a43e68ecd90db1814e`
(the proof-run-2/kernel log), and no checkpoint from `seq` 2
on for any log. Its private key was unencrypted at rest; it has
been re-wrapped passphrase-encrypted and is retained, encrypted,
on the operator's machine pending the owner's decision to
destroy it — it can sign nothing a conforming verifier will
accept past the boundary above.

The successor key (in `pubkey.txt`) signs from
`checkpoint_seq` 2 on for that `log_id`. Its first checkpoint
(seq 2, same anchored prefix: length 507) chains to the retired
key's last checkpoint via `prev_checkpoint`, so the history's
continuity is hash-chained across the rotation. Verify with the
registry:

```
python3 tools/jqanchor.py verify <presented-log.jsonl> \
    --checkpoints <path-to-this-repo-clone> \
    --keys <path-to-this-repo-clone>
```

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

The single-key form above remains valid for histories signed
under one key. The command prints a JSON result and exits 0, 1 or 3. Outcomes
(spec 15.5): `verified` (fully anchored, tail 0),
`verified_to_anchor` (anchored prefix verified; `anchored_length`
and `unanchored_tail` reported as distinct numbers — an extension
of an anchored log is this, not a forgery), `unanchored` (exit 3;
nothing here witnesses that log), `failed` (exit 1; the JSON
names the first divergent checkpoint, or the history / internal
stage that failed). Step-by-step drill, including the forged-log
exercise: `docs/anchor-drill.md` in the kernel repository.

## Registering a new log — the null-window rule (updated 2026-11-03)

The `keys.json` registry schedules checkpoints per log: each
key's `signs_from` / `signs_until` maps name the log_ids it
signs for, by checkpoint sequence, and the windows of the
participating keys must partition a log's checkpoint
sequence. To register a new log, name it in the signing key's
`signs_from` (and in the previous key's `signs_until` if the
log changes hands); keys that do not sign the log simply do
not name it.

This section previously recorded a limitation: under the
original grammar (`jacquard/anchor-keys/1`) a key with no
entry for a log defaulted to signing it from seq 0, so a log
born after a rotation — the gpu-queue live log (9cbf80c8…)
was the live example — could not be scheduled by the
registry at all and verified only via the explicit
single-key form. **Fixed in kernel v1.0.1 (spec 0.6
erratum 2):** the grammar is now `jacquard/anchor-keys/2`,
in which a `log_id` omitted from BOTH of a key's maps is a
null window. `keys.json` here has been republished as /2,
byte-identical apart from the format string — its entries
already stated the /2 reading as fact. The gpu-queue live
log now verifies via `verify --keys` (91/91 over its
checkpoints) and the pre-rotation proof log still verifies
507/507 under the same registry, in all three
implementations (Python, TypeScript, Rust). Verifiers accept
/1 registries under the legacy rule and /2 registries under
the null-window rule; tooling older than v1.0.1 refuses a /2
registry by name rather than misreading it.
