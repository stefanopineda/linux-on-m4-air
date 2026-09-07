# linux-on-m4-air (GitHub notebook)

Public-ready **notes** for the 15" M4 Air Linux bring-up. Live kernels and USB scripts stay in the studio tree (not this repo).

- Humans: `README.md` then `docs/getting-started.md` and `docs/g2b.md`
- Agents: `AGENTS.md` + `status.json`
- Stub 1TR deaths: `docs/step2-failure-matrix.md`
- Do not kmutil Macintosh HD. Do not bless the 2.5 GB stub. Do not put `.env` here.

This GitHub repo may still be **private**. Flip public only after a human reads the tree:

```
gh repo edit --visibility public --accept-visibility-change-consequences
```

`status.json` has `"public_ready": true` when the notebook is meant to survive that flip (no LAN IPs, USB serials, volume UUIDs, or credential names).
