# ingress-nginx — maintenance fork (nginx bumped)

Fork of [kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)
kept alive after the upstream project was archived, to carry a **newer, patched
NGINX** on top of a known controller release during migration off this controller.

## Support policy

This fork is maintained for **one single purpose**: to keep **this one release
alive** — controller **v1.15.1** rebuilt on a patched **NGINX 1.30.7** — with
security patches, as a bridge while we migrate away from this controller.

- Only the `v1.15.1-nginx-1.30.7` branch and the `v1.15.1-fuseone.1` images are maintained.
- **Previous controller/NGINX versions are NOT supported.** We do not backport to,
  rebuild, or accept issues for any other version.
- No upstream-style guarantees. Provided **as-is, no warranty**. If you choose to
  use it, pin to our image digests and track this branch for patch bumps.
- This is a stopgap, not a destination — it exists only until the migration completes.

## Branch `v1.15.1-nginx-1.30.7`

Based on tag `controller-v1.15.1` with a minimal, generic delta:

| Change | From | To |
|---|---|---|
| `images/nginx/rootfs/build.sh` `NGINX_VERSION` | 1.27.1 | **1.30.7** (stable; carries the Nov/2026 HTTP/2 memory-corruption fixes) |
| `images/nginx/TAG` | v2.2.9 | v1.30.7-fuseone.1 |
| `NGINX_BASE` | registry.k8s.io/…/nginx:v2.2.8 | ghcr.io/fuseone-io/ingress-nginx/nginx:v1.30.7-fuseone.1 |
| `images/nginx/rootfs/patches/` | 34 patches (`-1.27.1-`) | **33 patches rebased to 1.30.7** |

### Dropped patches (2)
- `19_*log_escape_non_ascii` — conflicts with an upstream log-bounds change in 1.30.x; cosmetic (log escaping). Left out for now; revisit if needed.
- `28_*CVE-2025-23419` — **already fixed upstream in 1.30.7** (fix landed in 1.27.4 / 1.26.3), so the backport is redundant.

The other 32 ingress patches were rebased onto 1.30.7 (git 3-way) and verified to apply cleanly in sequence; the nginx base compiles green with the full module set (lua, geoip2, modsecurity, otel, brotli, njs, http/2, http/3, stream).

## Rebasing patches to a new nginx version (reproducible)

```sh
# A = vanilla <old>, patched = A + images/nginx/rootfs/patches/*, B = vanilla <new> (parent A)
# git rebase --onto B A patched   # resolves offsets; stop only on real conflicts
# then: git show per commit -> regenerate images/nginx/rootfs/patches/*.patch
```

## Build

```sh
# 1) nginx base
make -C images/nginx push REGISTRY=ghcr.io/fuseone-io/ingress-nginx
# 2) controller (compiles the Go binary, then image FROM NGINX_BASE)
make build
make image  REGISTRY=ghcr.io/fuseone-io/ingress-nginx
docker push ghcr.io/fuseone-io/ingress-nginx/controller:$(cat TAG)
```

CI (`.github/workflows/fuseone-build.yml`) does both and pushes to GHCR.
