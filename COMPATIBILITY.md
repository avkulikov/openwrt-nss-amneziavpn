# AmneziaWG Compatibility Notes

## Applied feed patches

- `patches/amneziawg/0001-luci-proto-versioning.patch`
  - Splits `PKG_VERSION` and `PKG_RELEASE` in `luci-proto-amneziawg`
  - Avoids `0.0.1-1` in `PKG_VERSION` to keep apk metadata clean

## Kernel module compatibility

`kmod-amneziawg` builds its module by copying the in-tree WireGuard sources
from `$(LINUX_DIR)/drivers/net/wireguard` and applying
`kmod-amneziawg/files/000-initial-amneziawg.patch`.

If the main-nss kernel diverges enough to break that patch, rebase the patch
inside the AmneziaWG feed and add a corresponding patch in
`patches/amneziawg/` so the workflow applies it during checkout.
