pkgbase=linux-nabu
_branch=sm8150/6.15
_srcname=linux-nabu
pkgver=6.15
pkgrel=1
pkgdesc='Snapdragon 855 Mainline Linux'
url='https://gitlab.com/sm8150-mainline/linux'
arch=(aarch64)
license=(GPL-2.0-only)
makedepends=(
  bc
  cpio
  gettext
  git
  libelf
  pahole
  perl
  python
  rust
  rust-bindgen
  rust-src
  tar
  xz

  # htmldocs
  # graphviz
  # imagemagick
  # python-sphinx
  # python-yaml
  # texlive-latexextra
)
options=(
  !debug
  !strip
)
source=(
  "$_srcname::git+https://gitlab.com/sm8150-mainline/linux.git#branch=$_branch"
  "extra.config"
  "https://g.tx0.su/tx0/nabu-mainline-patches/raw/branch/main/0001-HACK-NABU-add-clk-delay-for-UFS.patch"
  "https://g.tx0.su/tx0/nabu-mainline-patches/raw/branch/main/0002-HACK-NABU-change-freq-table-for-UFS.patch"
  #"https://g.tx0.su/tx0/nabu-mainline-patches/raw/branch/main/0003-NABU-dts-enable-ln8000-charger-reduce-charge-voltage.patch"
)
sha256sums=(
  "SKIP"
  "SKIP"
  "SKIP"
  "SKIP"
  #"SKIP"
)

export KBUILD_BUILD_HOST=lon
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"
export KARCH=arm64

prepare() {
  cd $_srcname

  echo "Setting version..."
  echo "-$pkgrel" >localversion.10-pkgrel
  echo "${pkgbase#linux}" >localversion.20-pkgname

  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    src="${src%.zst}"
    [[ $src = *.patch ]] || continue
    echo "Applying patch $src..."
    patch -Np1 <"../$src"
  done

  echo "Setting config..."
  cp ../extra.config ./arch/$KARCH/configs/extra.config
  make defconfig sm8150.config extra.config

  make -s kernelrelease >version

  # Don't run depmod on "make install", we'll do that ourselves in packaging
  sed -i '2iexit 0' scripts/depmod.sh
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  unset LDFLAGS
  make Image.gz modules dtbs
}

pkgver() {
  cd $_srcname
  printf "%s.%s" "$(make kernelversion -s)" "$(git rev-parse --short=12 HEAD)"
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(
    'linux-firmware-xiaomi-nabu>=25.03.29'
    'coreutils'
    'initramfs'
    'kmod'
  )
  optdepends=(
    'linux-firmware: firmware images needed for some devices'
    'scx-scheds: to use sched-ext schedulers'
    'wireless-regdb: to set the correct wireless channels of your country'
  )
  provides=(
    WIREGUARD-MODULE
  )
  replaces=(
  )

  cd $_srcname
  local modulesdir="$pkgdir/usr/lib/modules/$(<version)"

  echo "Installing boot image..."
  install -d -m755 "$pkgdir/boot"
  install -Dm644 arch/$KARCH/boot/dts/qcom/sm8150-xiaomi-nabu.dtb "$pkgdir/boot/dtb-$(<version)"
  ln -sr "$pkgdir/boot/dtb-$(<version)" "$pkgdir/boot/dtb-$pkgbase"
  install -Dm644 arch/$KARCH/boot/Image "$pkgdir/boot/vmlinuz-$(<version)"
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 arch/$KARCH/boot/Image "$pkgdir/usr/lib/modules/$(<version)/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  ZSTD_CLEVEL=19 make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
    DEPMOD=/doesnt/exist modules_install # Suppress depmod

  rm "$modulesdir"/build
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(pahole)
  provides=(
    'linux-headers'
  )

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/$KARCH" -m644 arch/$KARCH/Makefile
  cp -t "$builddir" -a scripts
  ln -srt "$builddir" "$builddir/scripts/gdb/vmlinux-gdb.py"

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/$KARCH" -a arch/$KARCH/include
  install -Dt "$builddir/arch/$KARCH/kernel" -m644 arch/$KARCH/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$builddir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ ${arch} = */${KARCH}/ || ${arch} == */arm/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing documentation..."
  rm -r "$builddir/Documentation"

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
    application/x-sharedlib\;*) # Libraries (.so)
      ${CROSS_COMPILE}strip -v $STRIP_SHARED "$file" ;;
    application/x-archive\;*) # Libraries (.a)
      ${CROSS_COMPILE}strip -v $STRIP_STATIC "$file" ;;
    application/x-executable\;*) # Binaries
      ${CROSS_COMPILE}strip -v $STRIP_BINARIES "$file" ;;
    application/x-pie-executable\;*) # Relocatable binaries
      ${CROSS_COMPILE}strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  ${CROSS_COMPILE}strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

_package-docs() {
  pkgdesc="Documentation for the $pkgdesc kernel"

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  echo "Installing documentation..."
  local src dst
  while read -rd '' src; do
    dst="${src#Documentation/}"
    dst="$builddir/Documentation/${dst#output/}"
    install -Dm644 "$src" "$dst"
  done < <(find Documentation -name '.*' -prune -o ! -type d -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/share/doc"
  ln -sr "$builddir/Documentation" "$pkgdir/usr/share/doc/$pkgbase"
}

pkgname=(
  "$pkgbase"
  "$pkgbase-headers"
)
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
