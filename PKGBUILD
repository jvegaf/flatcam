# Maintainer: your name <you@example.com>
# FlatCAM Evo: PyQt6 CAM app (Gerber/Excellon -> G-Code) for PCB milling.
#
# Packaging strategy: the app is uv-managed (pyproject.toml + uv.lock) and runs
# from its source tree with flat imports, so we ship a frozen venv under
# /opt/flatcam built from uv.lock at package time. This mirrors the dev flow
# `uv run flatcam.py` and avoids the 8 runtime deps that have no Arch package
# (rasterio, ortools, vispy, ezdxf, svgtrace, svg-path, pyopengl, pyppeteer).
#
# Build from the repo root:  makepkg -f
# Install:                   sudo pacman -U flatcam-evo-*.pkg.tar.zst

pkgname=flatcam-evo
pkgver=0.1.0
pkgrel=1
pkgdesc="FlatCAM Evo - CAM app that turns Gerber/Excellon files into G-Code for PCB milling"
arch=('x86_64')
url="https://github.com/jvegaf/flatcam"
license=('MIT')
# The venv's python is a symlink to the system interpreter, hence python>=3.14
# (pyproject.toml: requires-python = ">=3.14"). Everything else comes from the
# frozen venv built out of uv.lock.
depends=('python>=3.14')
makedepends=('uv' 'git')
optdepends=(
  'mesa: OpenGL rendering for the 2D/3D viewers'
  'chromium: headless browser used by pyppeteer (PDF import)'
)
# Source is the local git tree (branch dev), so the build stays reproducible
# against the exact checked-in lockfile. No untracked junk (.venv, __pycache__,
# errors.txt) enters the package.
source=("${pkgname}::git+file://$(pwd)#branch=dev")
sha256sums=('SKIP')

build() {
  cd "$srcdir/$pkgname"
  # Frozen install of the locked dependency set into an in-tree venv.
  # --no-install-project: we run flatcam.py from source, the [project.scripts]
  # entry point points at the uv scaffold stub, not the real app.
  # --python pinned to the system interpreter so the venv survives the move
  # from $srcdir to /opt.
  uv sync --frozen --no-install-project --python /usr/bin/python3
}

package() {
  cd "$srcdir/$pkgname"

  install -d "$pkgdir/opt/flatcam"
  cp -a . "$pkgdir/opt/flatcam/"
  rm -rf "$pkgdir/opt/flatcam/.git"

  # Launcher: /opt/flatcam/flatcam.py puts /opt/flatcam on sys.path, which is
  # what the app's flat imports (appMain, camlib, ...) require.
  install -Dm755 /dev/stdin "$pkgdir/usr/bin/flatcam-evo" <<'EOF'
#!/bin/sh
exec /opt/flatcam/.venv/bin/python /opt/flatcam/flatcam.py "$@"
EOF

  # Desktop entry with absolute Exec/Icon (the repo ships relative paths).
  install -Dm644 assets/linux/flatcam-beta.desktop \
    "$pkgdir/usr/share/applications/flatcam-evo.desktop"
  sed -i "s|^Exec=.*|Exec=/usr/bin/flatcam-evo|" \
    "$pkgdir/usr/share/applications/flatcam-evo.desktop"
  sed -i "s|^Icon=.*|Icon=/usr/share/pixmaps/flatcam-evo.png|" \
    "$pkgdir/usr/share/applications/flatcam-evo.desktop"
  install -Dm644 assets/linux/icon.png "$pkgdir/usr/share/pixmaps/flatcam-evo.png"

  # Man page, gzipped as pacman expects.
  install -Dm644 <(gzip -n -c flatcam-beta.1) \
    "$pkgdir/usr/share/man/man1/flatcam-evo.1.gz"

  install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}
