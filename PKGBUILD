# Maintainer: Allan McRae <allan@archlinux.org>

# toolchain build order: linux-api-headers->glibc->binutils->gcc->binutils->glibc
# NOTE: valgrind requires rebuilt with each major glibc version

pkgname=glibc
pkgver=2.16.0
pkgrel=2
pkgdesc="GNU C Library"
arch=('i686' 'x86_64')
url="http://www.gnu.org/software/libc"
license=('GPL' 'LGPL')
groups=('base')
depends=('linux-api-headers>=3.4' 'tzdata')
makedepends=('gcc>=4.7')
backup=(etc/gai.conf
        etc/locale.gen
        etc/nscd.conf)
options=('!strip')
install=glibc.install
source=(http://ftp.gnu.org/gnu/libc/${pkgname}-${pkgver}.tar.xz{,.sig}
        glibc-2.15-fix-res_query-assert.patch
        glibc-2.15-revert-c5a0802a.patch
        nscd.rcd
        nscd.service
        nscd.tmpfiles
        locale.gen.txt
        locale-gen)
md5sums=('80b181b02ab249524ec92822c0174cf7'
         '2a1221a15575820751c325ef4d2fbb90'
         '31f415b41197d85d3bbee3d1eecd06a3'
         '0a0383d50d63f1c02919fe9943b82014'
         '589d79041aa767a5179eaa4e2737dd3f'
         'ad8a9af15ab7eeaa23dc7ee85024af9f'
         'bccbe5619e75cf1d97312ec3681c605c'
         '07ac979b6ab5eeb778d55f041529d623'
         '476e9113489f93b348b21e144b6a8fcf')

build() {
  cd ${srcdir}/${pkgname}-${pkgver}

  # fix res_query assertion
  # http://sourceware.org/bugzilla/show_bug.cgi?id=13013
  patch -p1 -i ${srcdir}/glibc-2.15-fix-res_query-assert.patch

  # revert commit c5a0802a - causes various hangs
  # https://bugzilla.redhat.com/show_bug.cgi?id=552960
  patch -p1 -i ${srcdir}/glibc-2.15-revert-c5a0802a.patch

  cd ${srcdir}
  mkdir glibc-build
  cd glibc-build

  if [[ ${CARCH} = "i686" ]]; then
    # Hack to fix NPTL issues with Xen, only required on 32bit platforms
    # TODO: make separate glibc-xen package for i686
    export CFLAGS="${CFLAGS} -mno-tls-direct-seg-refs"
  fi

  echo "slibdir=/lib" >> configparms

  # remove hardening options from CFLAGS for building libraries
  CFLAGS=${CFLAGS/-fstack-protector/}
  CFLAGS=${CFLAGS/-D_FORTIFY_SOURCE=2/}

  ${srcdir}/${pkgname}-${pkgver}/configure --prefix=/usr \
      --libdir=/usr/lib --libexecdir=/usr/lib \
      --with-headers=/usr/include \
      --enable-add-ons=nptl,libidn \
      --enable-obsolete-rpc \
      --enable-kernel=2.6.32 \
      --enable-bind-now --disable-profile \
      --enable-stackguard-randomization \
      --enable-multi-arch

  # build libraries with hardening disabled
  echo "build-programs=no" >> configparms
  make
  
  # re-enable hardening for programs
  sed -i "/build-programs=/s#no#yes#" configparms
  echo "CC += -fstack-protector -D_FORTIFY_SOURCE=2" >> configparms
  echo "CXX += -fstack-protector -D_FORTIFY_SOURCE=2" >> configparms
  make

  # remove harding in preparation to run test-suite
  sed -i '2,4d' configparms
}

check() {
  cd ${srcdir}/glibc-build
  make -k check
}

package() {
  cd ${srcdir}/glibc-build

  ln -s usr/lib ${pkgdir}/lib

  install -dm755 ${pkgdir}/etc
  touch ${pkgdir}/etc/ld.so.conf

  make install_root=${pkgdir} install

  rm -f ${pkgdir}/etc/ld.so.{cache,conf}

  install -dm755 ${pkgdir}/{etc/rc.d,usr/{sbin,lib/{,locale,systemd/system,tmpfiles.d}}}

  install -m644 ${srcdir}/${pkgname}-${pkgver}/nscd/nscd.conf ${pkgdir}/etc/nscd.conf
  sed -i -e 's/^\tserver-user/#\tserver-user/' ${pkgdir}/etc/nscd.conf
  install -m755 ${srcdir}/nscd.rcd ${pkgdir}/etc/rc.d/nscd
  install -m644 ${srcdir}/nscd.service ${pkgdir}/usr/lib/systemd/system
  install -m644 ${srcdir}/nscd.tmpfiles ${pkgdir}/usr/lib/tmpfiles.d/nscd.conf

  install -m644 ${srcdir}/${pkgname}-${pkgver}/posix/gai.conf ${pkgdir}/etc/gai.conf

  install -m755 ${srcdir}/locale-gen ${pkgdir}/usr/sbin

  # create /etc/locale.gen
  install -m644 ${srcdir}/locale.gen.txt ${pkgdir}/etc/locale.gen
  sed -e '1,3d' -e 's|/| |g' -e 's|\\| |g' -e 's|^|#|g' \
    ${srcdir}/glibc-2.16.0/localedata/SUPPORTED >> ${pkgdir}/etc/locale.gen

  if [[ ${CARCH} = "x86_64" ]]; then
    # fix paths and compliance with binary blobs...
    sed -i '/RTLDLIST/s%lib64%lib%' ${pkgdir}/usr/bin/ldd
    ln -s usr/lib ${pkgdir}/lib64
  fi

  # Do not strip the following files for improved debugging support
  # ("improved" as in not breaking gdb and valgrind...):
  #   ld-${pkgver}.so
  #   libc-${pkgver}.so
  #   libpthread-${pkgver}.so
  #   libthread_db-1.0.so

  cd $pkgdir
  strip $STRIP_BINARIES sbin/{ldconfig,sln} \
                        usr/bin/{gencat,getconf,getent,iconv,locale,localedef} \
                        usr/bin/{makedb,pcprofiledump,pldd,rpcgen,sprof} \
                        usr/lib/getconf/* \
                        usr/sbin/{iconvconfig,nscd}
  [[ $CARCH = "i686" ]] && strip $STRIP_BINARIES usr/bin/lddlibc4

  strip $STRIP_STATIC usr/lib/*.a

  strip $STRIP_SHARED usr/lib/{libanl,libBrokenLocale,libcidn,libcrypt}-*.so \
                      usr/lib/libnss_{compat,db,dns,files,hesiod,nis,nisplus}-*.so \
                      usr/lib/{libdl,libm,libnsl,libresolv,librt,libutil}-*.so \
                      usr/lib/{libmemusage,libpcprofile,libSegFault}.so \
                      usr/lib/{pt_chown,{audit,gconv}/*.so}
}
