# Trustycar roots

The signed daily roots of [Trustycar](https://trustycar.app) vehicle passports: an append-only public log.

Each day, the Trustycar server collects the fingerprints (SHA-256 hashes) the app sent that day, builds one Merkle tree over them, and signs its root. The server publishes every root here and at `https://trustycar.app/roots/<year>.json`, and both copies hold the same bytes. A buyer can then check that a passport record existed in its printed form on a given day without trusting the seller or Trustycar: we can't rewrite a published day without it showing in this repository's history.

What this proves and how to check a passport, in plain words and in full technical detail: **https://trustycar.app/how-to-check/**

## Files

`roots/<year>.json` is one file per year (UTC days), one day per line, in ascending order:

```json
{
  "2026-11-21": { "root": "51b0…7e", "sig": "…", "key_id": "9b4e063fea61e081" },
  "2026-11-22": { "root": "e41d…5a", "sig": "…", "key_id": "9b4e063fea61e081" }
}
```

- `root`: the Merkle root of the day, as 64 lowercase hex characters.
- `sig`: Ed25519 (RFC 8032), as 128 lowercase hex characters, over the UTF-8 bytes of `["trustycar/root/v1","<day>","<root>"]`, written with no spaces.
- `key_id`: the first 16 hex characters of SHA-256 of the public key. The keys are at https://trustycar.app/keys.json. Trust only `"env": "production"` keys, for days from their `validFrom` on, and before their `revokedFrom` if they have one.

The bot only ever appends a day. It never changes or removes one. Any commit that edits a published day is the thing to look for.

## Check a signature

With Python and [PyNaCl](https://pynacl.readthedocs.io/):

```python
import json, urllib.request
from nacl.signing import VerifyKey

keys = json.load(urllib.request.urlopen("https://trustycar.app/keys.json"))["keys"]
roots = json.load(open("roots/2026.json"))
for day, r in roots.items():
    key = next(k for k in keys if k["keyId"] == r["key_id"] and k["env"] == "production")
    assert key["validFrom"] <= day < key.get("revokedFrom", "9999-12-31"), day
    message = json.dumps(["trustycar/root/v1", day, r["root"]], separators=(",", ":")).encode()
    VerifyKey(bytes.fromhex(key["publicKey"])).verify(message, bytes.fromhex(r["sig"]))  # raises if wrong
    print(day, "ok")
```

Checking that a passport record belongs to a day's root (hash format, Merkle paths, golden vectors) is described at https://trustycar.app/how-to-check/.

## License

CC0 1.0: no rights reserved.
