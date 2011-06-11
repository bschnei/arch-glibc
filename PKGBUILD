# Maintainer: Allan McRae <allan@archlinux.org>

# toolchain build order: linux-api-headers->glibc->binutils->gcc->binutils->glibc
# NOTE: valgrind requires rebuilt with each new glibc version

pkgname=glibc
pkgver=2.14
pkgrel=2
_glibcdate=20110605
pkgdesc="GNU C Library"
arch=('i686' 'x86_64')
url="http://www.gnu.org/software/libc"
license=('GPL' 'LGPL')
groups=('base')
depends=('linux-api-headers>=2.6.39' 'tzdata')
makedepends=('gcc>=4.4')
backup=(etc/locale.gen
        etc/nscd.conf)
options=('!strip')
install=glibc.install
source=(ftp://ftp.archlinux.org/other/glibc/${pkgname}-${pkgver}_${_glibcdate}.tar.xz
        glibc-2.10-dont-build-timezone.patch
        glibc-2.10-bz4781.patch
        glibc-__i686.patch
        glibc-2.12.1-static-shared-getpagesize.patch
        glibc-2.12.2-ignore-origin-of-privileged-program.patch
        glibc-2.13-futex.patch
        glibc-2.14-libdl-crash.patch
        glibc-2.14-revert-4462fad3.patch
        nscd
        locale.gen.txt
        locale-gen)
md5sums=('a96742599fc8a99e52b9e344f39a1000'
         '4dadb9203b69a3210d53514bb46f41c3'
         '0c5540efc51c0b93996c51b57a8540ae'
         '40cd342e21f71f5e49e32622b25acc52'
         'a3ac6f318d680347bb6e2805d42b73b2'
         'b042647ea7d6f22ad319e12e796bd13e'
         '7d0154b7e17ea218c9fa953599d24cc4'
         'cea62cc6b903d222c5f26e05a3c0e0e6'
         '46e56492cccb1c9172ed3a235cf43c6c'
         'b587ee3a70c9b3713099295609afde49'
         '07ac979b6ab5eeb778d55f041529d623'
         '476e9113489f93b348b21e144b6a8fcf')

mksource() {
  git clone git://sourceware.org/git/glibc.git
  pushd glibc
  git checkout -b glibc-2.14-arch origin/master
  # git checkout -b glibc-2.14-arch origin/release/2.14/master
  popd
  tar -cvJf glibc-${pkgver}_${_glibcdate}.tar.xz glibc/*
}

build() {
  cd ${srcdir}/glibc

  # timezone data is in separate package (tzdata)
  patch -Np1 -i ${srcdir}/glibc-2.10-dont-build-timezone.patch

  # http://sources.redhat.com/bugzilla/show_bug.cgi?id=4781
  patch -Np1 -i ${srcdir}/glibc-2.10-bz4781.patch

  # http://sources.redhat.com/bugzilla/show_bug.cgi?id=411
  # http://sourceware.org/ml/libc-alpha/2009-07/msg00072.html
  patch -Np1 -i ${srcdir}/glibc-__i686.patch

  # http://sourceware.org/bugzilla/show_bug.cgi?id=11929
  # using Fedora "fix" as patch in that bug report causes breakages...
  patch -Np1 -i ${srcdir}/glibc-2.12.1-static-shared-getpagesize.patch

  # http://www.exploit-db.com/exploits/15274/
  # http://sourceware.org/git/?p=glibc.git;a=patch;h=d14e6b09 (only fedora branch...)
  patch -Np1 -i ${srcdir}/glibc-2.12.2-ignore-origin-of-privileged-program.patch

  # http://sourceware.org/bugzilla/show_bug.cgi?id=12403
  patch -Np1 -i ${srcdir}/glibc-2.13-futex.patch

  # http://sourceware.org/git/?p=glibc.git;a=commitdiff;h=675155e9 (only fedora branch...)
  # http://sourceware.org/ml/libc-alpha/2011-06/msg00006.html
  patch -Np1 -i ${srcdir}/glibc-2.14-libdl-crash.patch

  # revert fix for http://sourceware.org/bugzilla/show_bug.cgi?id=12684
  # as it causes crashes  (FS#24615)
  patch -Np1 -i ${srcdir}/glibc-2.14-revert-4462fad3.patch

  install -dm755 ${pkgdir}/etc
  touch ${pkgdir}/etc/ld.so.conf

  cd ${srcdir}
  mkdir glibc-build
  cd glibc-build

  if [[ ${CARCH} = "i686" ]]; then
    # Hack to fix NPTL issues with Xen, only required on 32bit platforms
    export CFLAGS="${CFLAGS} -mno-tls-direct-seg-refs"
  fi

  echo "slibdir=/lib" >> configparms

  ${srcdir}/glibc/configure --prefix=/usr \
      --libdir=/usr/lib --libexecdir=/usr/lib \
      --with-headers=/usr/include \
      --enable-add-ons=nptl,libidn \
      --enable-kernel=2.6.27 \
      --with-tls --with-__thread \
      --enable-bind-now --without-gd \
      --without-cvs --disable-profile \
      --disable-multi-arch
        
  make
}

check() {
  cd ${srcdir}/glibc-build

  # some errors are expected - manually check log files
  make -k check || true
}

package() {
  cd ${srcdir}/glibc-build
  make install_root=${pkgdir} install

  rm ${pkgdir}/etc/ld.so.{cache,conf}

  install -dm755 ${pkgdir}/etc/rc.d
  install -dm755 ${pkgdir}/usr/sbin
  install -dm755 ${pkgdir}/usr/lib/locale
  install -m644 ${srcdir}/glibc/nscd/nscd.conf ${pkgdir}/etc/nscd.conf
  install -m755 ${srcdir}/nscd ${pkgdir}/etc/rc.d/nscd
  install -m755 ${srcdir}/locale-gen ${pkgdir}/usr/sbin
  install -m644 ${srcdir}/glibc/posix/gai.conf ${pkgdir}/etc/gai.conf

  sed -i -e 's/^\tserver-user/#\tserver-user/' ${pkgdir}/etc/nscd.conf

  # create /etc/locale.gen
  install -m644 ${srcdir}/locale.gen.txt ${pkgdir}/etc/locale.gen
  sed -i "s|/| |g" ${srcdir}/glibc/localedata/SUPPORTED
  sed -i 's|\\| |g' ${srcdir}/glibc/localedata/SUPPORTED
  sed -i "s|SUPPORTED-LOCALES=||" ${srcdir}/glibc/localedata/SUPPORTED
  cat ${srcdir}/glibc/localedata/SUPPORTED >> ${pkgdir}/etc/locale.gen
  sed -i "s|^|#|g" ${pkgdir}/etc/locale.gen

  if [[ ${CARCH} = "x86_64" ]]; then
    # fix for the linker
    sed -i '/RTLDLIST/s%lib64%lib%' ${pkgdir}/usr/bin/ldd
    # Comply with multilib binaries, they look for the linker in /lib64
    mkdir ${pkgdir}/lib64
    cd ${pkgdir}/lib64
    ln -v -s ../lib/ld* .
  fi
  
  # manually strip files as stripping libpthread-*.so and libthread_db.so
  # with the default $STRIP_SHARED breaks gdb and stripping ld-*.so breaks
  # valgrind on x86_64

  cd $pkgdir
  strip $STRIP_BINARIES sbin/{ldconfig,sln} \
                        usr/bin/{gencat,getconf,getent,iconv,locale} \
                        usr/bin/{localedef,pcprofiledump,rpcgen,sprof} \
                        usr/lib/getconf/* \
                        usr/sbin/{iconvconfig,nscd}
  [[ $CARCH = "i686" ]] && strip $STRIP_BINARIES usr/bin/lddlibc4

  strip $STRIP_STATIC usr/lib/*.a \
                      lib/{{ld,libpthread}-${pkgver},libthread_db-1.0}.so

  strip $STRIP_SHARED lib/{libanl,libBrokenLocale,libc,libcidn,libcrypt}-${pkgver}.so \
                      lib/libnss_{compat,dns,files,hesiod,nis,nisplus}-${pkgver}.so \
                      lib/{libdl,libm,libnsl,libresolv,librt,libutil}-${pkgver}.so \
                      lib/{libmemusage,libpcprofile,libSegFault}.so \
                      usr/lib/{pt_chown,{audit,gconv}/*.so}
}
