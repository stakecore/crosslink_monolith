# stakecore packaging

A container image of this monorepo's `zebrad`, run headless as a service. Upstream
ships binaries and a Dockerfile inherited from Zebra whose build context cannot see
the sibling crates; this directory is the missing piece and nothing else. No source
file outside `stakecore/` and `.github/workflows/stakecore-image.yml` is touched.

| File | What it is |
|---|---|
| [Dockerfile](Dockerfile) | Two stages, repository root as context: `cargo build --locked --release -p zebrad`, then upstream's runtime layout and entrypoint on `debian:trixie-slim`. |
| [Dockerfile.dockerignore](Dockerfile.dockerignore) | The build context as an allow-list: the `zebra-crosslink` workspace and the siblings it reaches with `../`. |
| [../.github/workflows/stakecore-image.yml](../.github/workflows/stakecore-image.yml) | Builds, starts the node for 90 s, then publishes to `ghcr.io/<owner>/zebra-crosslink`. |

## Branches

- `stakecore` holds the packaging and should be the fork's **default branch**: the
  manual "Run workflow" button only exists for workflows on the default branch.
- The source that gets built is a separate ref (`DEFAULT_REF` in the workflow, or the
  `ref` input). A new upstream release is a one-line bump; our own patch is a branch
  off the release tag, built with `ref: <branch>` and `tag: v14-stakecore.1`.

## Running it

```bash
docker build -f stakecore/Dockerfile -t zebra-crosslink:dev .
docker run --rm zebra-crosslink:dev zebrad generate     # the built-in config
docker run --rm \
  -e ZEBRA_MINING__INTERNAL_MINER=false \
  -e ZEBRA_CROSSLINK__DISABLE_THE_HEADLESS_WALLET=true \
  -v crosslink:/home/zebra/.cache/zebra \
  zebra-crosslink:dev
```

With no config file and no `ZEBRA_NETWORK__NETWORK`, the node joins the release's
own test network: peers, magic and genesis are compiled in. Do not set the network.

Things the defaults do that a server operator should know:

- **The miner is on** (`mining.internal_miner`). Turn it off unless you mean it.
- **`<cache_dir>/<network dir>/secret.seed`** is created on first start. The finalizer
  key, the wallet and the miner address all derive from those 32 bytes.
- **Listening sockets**, with the default P2P port 8233: 8233/tcp (P2P), a random UDP
  port for block transport unless `state.network_local_port` is set, 18233/tcp and
  18234/tcp on every interface (the built-in light-wallet server: P2P port + 10000
  and + 10001, unauthenticated), and 127.0.0.1:8232 (JSON-RPC, cookie auth off).
- `crosslink.public_address` defaults to `127.0.0.1:<random>`; a node that should take
  part in BFT needs its real `ip:port` there.
