# Sealed Post

**Time capsules on Gno.land.** Seal a message today so that nobody — not even you — can change what it says, and let anyone open it once its time has come. The message itself stays secret until the reveal: the chain only ever stores a SHA-256 commitment.

Sealed Post is built in two layers:

1. **`commitreveal`** — a small, reusable Gno package implementing the classic commit–reveal scheme. Any realm can import it: sealed bids, sealed votes, hidden game moves, commit-now-prove-later attestations.
2. **`sealedpost`** — a realm that turns the primitive into a product: public time capsules with a countdown, an optional "addressed to" recipient, and a permanent, verifiable reveal.

## Live on the Gno.land testnet (Onyx / Test17, chain `onyx-1`)

| Piece | Path | Status |
|---|---|---|
| Package | `gno.land/p/g1gu8mdvjg8l5wj2ytknhs7pmyu0rmeclyvvlsd8/commitreveal` | Live |
| Realm | `gno.land/r/g1gu8mdvjg8l5wj2ytknhs7pmyu0rmeclyvvlsd8/sealedpost` | Deployed (addpkg tx at height 1348160), pending GPAS activation |

Browse the realm: https://latest.testnets.gno.land/r/g1gu8mdvjg8l5wj2ytknhs7pmyu0rmeclyvvlsd8/sealedpost

The home page lists every capsule with its title, recipient, and blocks remaining until unlock. A sealed capsule's page shows the commitment and the countdown — never the payload. Once revealed, the page shows the message and anyone can re-verify it against the commitment.

## How it works

1. **Commit (off-chain, by you).** Pick a message (`payload`) and a random `salt`. The commitment is the lowercase hex SHA-256 of `salt + ":" + payload`:

   ```sh
   printf '%s:%s' 'YOUR_SALT' 'YOUR MESSAGE' | sha256sum | cut -d' ' -f1
   ```

   Keep the payload and salt somewhere safe. They are **not** stored on-chain, and they are the only way to open the capsule.

2. **Seal (on-chain).** Call `Seal(title, commitment, unlockHeight, recipient)` on the realm. The chain stores the title, the commitment, the unlock height, and the optional recipient label. Because only the hash is public, the message is secret — but it is also frozen: no different message can ever be revealed against that commitment.

3. **Reveal (on-chain, by anyone who knows the secret).** Once the chain reaches `unlockHeight`, call `Reveal(id, payload, salt)`. The realm recomputes the commitment with the `commitreveal` package and opens the capsule only if it matches. Knowledge of payload + salt *is* the authorization — there is no owner key to trust or lose.

### Rules enforced by the realm

- `unlockHeight` must be in the future at seal time.
- Title ≤ 80 characters; revealed payload ≤ 2000 characters.
- `recipient` is an optional "addressed to" label (empty, or a valid bech32 address). It is informational — it does not restrict who can reveal.
- A capsule can be revealed at most once, and never before its unlock height.
- A reveal with the wrong payload or salt is rejected; the capsule stays sealed.

## Using Sealed Post

You need a Gno key (`gnokey add`) funded on the Onyx testnet (faucet: https://faucet.gno.land, network `onyx-1`).

**Seal a capsule** (unlock 500 blocks from now, addressed to a friend):

```sh
gnokey maketx call \
  -pkgpath gno.land/r/g1gu8mdvjg8l5wj2ytknhs7pmyu0rmeclyvvlsd8/sealedpost \
  -func Seal \
  -args "A letter to the future" \
  -args "<64-hex commitment from the recipe above>" \
  -args "<unlock height>" \
  -args "g1..." \
  -gas-fee 2000ugnot -gas-wanted 2000000 \
  -chainid onyx-1 \
  -remote https://rpc.onyx.testnets.gno.land:443 \
  YOUR_KEY_NAME
```

**Read capsules** — just open the realm in gnoweb (link above). Capsule pages live at `.../sealedpost/capsule/<id>`.

**Reveal a capsule** once its countdown reaches zero:

```sh
gnokey maketx call \
  -pkgpath gno.land/r/g1gu8mdvjg8l5wj2ytknhs7pmyu0rmeclyvvlsd8/sealedpost \
  -func Reveal \
  -args "<capsule id>" \
  -args "YOUR MESSAGE" \
  -args "YOUR_SALT" \
  -gas-fee 2000ugnot -gas-wanted 2000000 \
  -chainid onyx-1 \
  -remote https://rpc.onyx.testnets.gno.land:443 \
  YOUR_KEY_NAME
```

## The `commitreveal` package — reuse it in your own realm

```gno
import "gno.land/p/g1gu8mdvjg8l5wj2ytknhs7pmyu0rmeclyvvlsd8/commitreveal"

// Store this at commit time:
c := commitreveal.Commit(payload, salt)        // hex SHA-256 of salt + ":" + payload

// Later, when the user reveals:
ok := commitreveal.Verify(c, payload, salt)    // true only for the original pair
```

API:

- `Commit(payload string, salt string) string` — the commitment to store.
- `Verify(commitment string, payload string, salt string) bool` — recompute and compare.
- `IsCommitment(s string) bool` — format check (64 lowercase hex chars).

Ideas it unlocks: sealed-bid auctions, commit–reveal voting, hidden-move games (rock-paper-scissors without a trusted dealer), prediction locks, and "prove you knew it first" timestamps.

## Repository layout

```
gno.land/p/g1gu8mdvjg8l5wj2ytknhs7pmyu0rmeclyvvlsd8/commitreveal/   the reusable package (+ tests)
gno.land/r/g1gu8mdvjg8l5wj2ytknhs7pmyu0rmeclyvvlsd8/sealedpost/     the time-capsule realm (+ render + tests)
```

The directory names are the deployer address because Gno.land package paths are namespaced by the publisher's address — the on-chain paths above are exactly the paths in this repo.

## Run the tests

With the Gno toolchain (v1.8.0) installed and `GNOROOT` pointing at a checkout of the `gnolang/gno` sources:

```sh
gno test ./gno.land/p/g1gu8mdvjg8l5wj2ytknhs7pmyu0rmeclyvvlsd8/commitreveal
gno test ./gno.land/r/g1gu8mdvjg8l5wj2ytknhs7pmyu0rmeclyvvlsd8/sealedpost
```

Coverage includes: seal → reveal at exactly the unlock height, too-early reveal, wrong payload/salt, double reveal, invalid commitment/recipient, non-future unlock height, overlong title, and a render test proving a sealed capsule's page shows the countdown and never leaks the payload. The package tests pin `Commit` against a SHA-256 vector verified outside Gno.

---

Built for the **GNOME Tavern Hackathon** (Gno.land, on DoraHacks).
