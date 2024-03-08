# AArch64 multi-platform
# Maintainer: Jat-faan Wong
# Contributor: Jat-faan Wong, Guoxin "7Ji" Pu, Joshua-Riek 

pkgbase=linux-aarch64-joshua-git
pkgname=("${pkgbase}"{,-headers})
pkgver=6.7.0.r1235704.5b9831b062b0
pkgrel=2
arch=('aarch64')
license=('GPL2')
url="https://github.com/Joshua-Riek"
_desc="with patches picked by Joshua Riek focusing on RK3588" 
makedepends=('cpio' 'xmlto' 'docbook-xsl' 'kmod' 'inetutils' 'bc' 'git' 'uboot-tools' 'vboot-utils' 'dtc')
options=('!strip')
_srcname='linux'
source=(
  "git+${url}/${_srcname}.git"
  'custom_reconfig'
  'linux.preset'
)

sha512sums=(
  'SKIP'
  '35c10b164f15c8b6544d23aa01afecb716c133e889c45cce6c97e7f3aaf27c136b0140230624ba1f124ac5c57b8bd75d5c3bbe9f11004ae2eaaea9c6bb3b550f'
  '2dc6b0ba8f7dbf19d2446c5c5f1823587de89f4e28e9595937dd51a87755099656f2acec50e3e2546ea633ad1bfd1c722e0c2b91eef1d609103d8abdc0a7cbaf'
)

prepare() {
  cd "${_srcname}"

  echo "Setting version..."
  echo - > localversion.09-hyphen
  {
    printf r 
    git rev-list --count HEAD
  } > localversion.10-rev-kernel
  echo - > localversion.19-hyphen
  git rev-parse --short HEAD > localversion.20-id-kernel
  echo "-${pkgrel}" > localversion.30-pkgrel
  echo "${pkgbase#linux}" > localversion.40-pkgname
  
  # this is only for local builds so there is no need to integrity check. (if needed)
  for p in ../../custom/*.patch; do
    echo "Custom Patching with ${p}"
    patch -p1 -N -i $p || true
  done

  echo "Preparing config..."
  cat arch/arm64/configs/rockchip_defconfig > .config
  scripts/kconfig/merge_config.sh -m .config ../custom_reconfig
  make olddefconfig prepare

  make -s kernelrelease > version
  echo "Prepared for $(<version)"
}

pkgver() {
  cd "${_srcname}"
  printf '%s.%s.%s' \
    "$(make kernelversion)" \
    "$(<localversion.10-rev-kernel)" \
    "$(<localversion.20-id-kernel)"
}

build() {
  cd "${_srcname}"

  unset LDFLAGS
  make ${MAKEFLAGS} Image modules
  make ${MAKEFLAGS} DTC_FLAGS="-@" dtbs
}

_package() {
  pkgdesc="The linux kernel, ${_desc}"
  depends=('coreutils' 'kmod' 'initramfs')
  optdepends=('wireless-regdb: to set the correct wireless channels of your country')
  backup=("etc/mkinitcpio.d/${pkgbase}.preset")

  cd "${_srcname}"
  
  # install dtbs
  make INSTALL_DTBS_PATH="${pkgdir}/boot/dtbs/${pkgbase}" dtbs_install

  # install modules
  make INSTALL_MOD_PATH="${pkgdir}/usr" INSTALL_MOD_STRIP=1 modules_install

  # copy kernel
  local _dir_module="${pkgdir}/usr/lib/modules/$(<version)"
  install -Dm644 arch/arm64/boot/Image "${_dir_module}/vmlinuz"

  # remove reference to build host
  rm -f "${_dir_module}/"{build,source}

  # used by mkinitcpio to name the kernel
  echo "${pkgbase}" | install -D -m 644 /dev/stdin "${_dir_module}/pkgbase"

  # install mkinitcpio preset file
  sed "s|%PKGBASE%|${pkgbase}|g" ../linux.preset |
    install -Dm644 /dev/stdin "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the Linux kernel, ${_desc}"
  depends=("python")

  cd "${_srcname}"
  local builddir="${pkgdir}/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map version
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "$builddir" -a scripts

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  echo "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/arm64" -a arch/arm64/include
  install -Dt "$builddir/arch/arm64/kernel" -m644 arch/arm64/kernel/asm-offsets.s
  mkdir -p "$builddir/arch/arm"
  cp -t "$builddir/arch/arm" -a arch/arm/include

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
    [[ $arch = */arm64/ || $arch == */arm/ ]] && continue
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
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name -print0)

  echo "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

for _p in ${pkgname[@]}; do
  eval "package_${_p}() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#${pkgbase}}
  }"
done
