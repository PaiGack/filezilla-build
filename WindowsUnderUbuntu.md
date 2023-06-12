Cross Compiling FileZilla 3 for Windows under Ubuntu
===

## Setting up the build environment
### Install Depends
```sh
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install automake autoconf libtool make gettext lzip
sudo apt install mingw-w64 pkg-config wx-common wine wine64 wine32 wine-binfmt subversion git
```

### env
```sh
mkdir ~/prefix
mkdir ~/src

export PATH="$HOME/prefix/bin:$PATH"
export LD_LIBRARY_PATH="$HOME/prefix/lib:$LD_LIBRARY_PATH"
export PKG_CONFIG_PATH="$HOME/prefix/lib/pkgconfig:$PKG_CONFIG_PATH"
export TARGET_HOST=x86_64-w64-mingw32
```

### wine
```sh
wine reg add HKCU\\Environment /f /v PATH /d "`x86_64-w64-mingw32-g++ -print-search-dirs | grep ^libraries | sed 's/^libraries: =//' | sed 's/:/;z:/g' | sed 's/^\\//z:\\\\\\\\/' | sed 's/\\//\\\\/g'`"
```

## Download
- GMP
```sh
wget https://gmplib.org/download/gmp/gmp-6.2.1.tar.lz
tar xf gmp-6.2.1.tar.lz
```

- Nettle
```sh
wget https://ftp.gnu.org/gnu/nettle/nettle-3.7.3.tar.gz
tar xf nettle-3.7.3.tar.gz
```
- GnuTLS
```sh
wget https://www.gnupg.org/ftp/gcrypt/gnutls/v3.8/gnutls-3.8.0.tar.xz
tar xvf gnutls-3.8.0.tar.xz
```

- SQLite
```sh
wget https://www.sqlite.org/2022/sqlite-autoconf-3400100.tar.gz
tar xvzf sqlite-autoconf-3400100.tar.gz
```

- wxWidgets
```sh
git clone --branch WX_3_0_BRANCH --single-branch https://github.com/wxWidgets/wxWidgets.git wx3
```

- libfilezilla (latest stable - 0.41.0 as of this guide against FileZilla v3.63.1):
```sh
svn co -r 10842 https://svn.filezilla-project.org/svn/libfilezilla/trunk libfilezilla
```
- FileZilla (latest stable - 3.63.1 as of this guide against libfilezilla 0.35):
```sh
svn co -r 10853 https://svn.filezilla-project.org/svn/FileZilla3/trunk filezilla
```
- NSIS
```sh
wget https://prdownloads.sourceforge.net/nsis/nsis-3.04-setup.exe
```

## Build Depends

### GMP
```sh
cd ~/src/gmp-6.2.1
CC_FOR_BUILD=gcc ./configure --host=$TARGET_HOST --prefix="$HOME/prefix" --disable-static --enable-shared --enable-fat
make -j4 && make install
```

### Nettle
```sh
cd ~/src/nettle-3.7.3
./configure --host=$TARGET_HOST --prefix="$HOME/prefix" --enable-shared --disable-static --enable-fat LDFLAGS="-L$HOME/prefix/lib" CPPFLAGS="-I$HOME/prefix/include"
make -j4 && make install
```

### GnuTLS
```sh
cd ~/src/gnutls-3.8.0
./configure --host=$TARGET_HOST --prefix="$HOME/prefix" --enable-shared --disable-static --without-p11-kit --with-included-libtasn1 --with-included-unistring --enable-local-libopts --disable-srp-authentication --disable-dtls-srtp-support --disable-heartbeat-support --disable-psk-authentication --disable-anon-authentication --disable-openssl-compatibility --without-tpm --without-brotli --disable-cxx --disable-doc LDFLAGS="-L$HOME/prefix/lib" CPPFLAGS="-I$HOME/prefix/include"
make -j4 && make install
```

### SQLite
```sh
cd sqlite-autoconf-3400100
./configure --host=$TARGET_HOST --prefix="$HOME/prefix" --enable-shared --disable-static --disable-dynamic-extensions
make -j4 && make install
```

### wxWidgets
```sh
cd ~/src/wx3
./configure --host=$TARGET_HOST --prefix="$HOME/prefix" --enable-shared --disable-static
make -j4 && make install
cp $HOME/prefix/lib/wx*.dll $HOME/prefix/bin
```

> 需要将类型的后缀名称去掉（使用软链接）
```sh
cd ~/prefix/lib
# wx_mswu_aui-3.0
# wx_mswu_xrc-3.0
# wx_mswu_adv-3.0
# wx_mswu_core-3.0
# wx_baseu_xml-3.0
# wx_baseu-3.0
ln -s libwx_mswu_xrc-3.0-x86_64-w64-mingw32.dll.a libwx_mswu_xrc-3.0.dll.a
```

### NSIS
```sh
cd ~/src
wine nsis-3.04-setup.exe /S
[ -f "$HOME/.wine/drive_c/Program Files (x86)/NSIS/makensis.exe" ] && echo "Success!"
```


## Build libfilezilla
```sh
cd ~/src/libfilezilla
autoreconf -i
./configure --host=$TARGET_HOST --prefix="$HOME/prefix" --enable-shared --disable-static 
make -j4 && make install
```

## Build filezilla
```sh
cd ~/src/filezilla
autoreconf -i
./configure --host=$TARGET_HOST --prefix="$HOME/prefix" --enable-shared --disable-static --with-pugixml=builtin LDFLAGS="-L$HOME/prefix/lib" CPPFLAGS="-I$HOME/prefix/include"
make -j4
```

### strip debug symbols
```sh
$TARGET_HOST-strip src/interface/.libs/filezilla.exe
$TARGET_HOST-strip src/putty/.libs/fzsftp.exe
$TARGET_HOST-strip src/putty/.libs/fzputtygen.exe
$TARGET_HOST-strip src/fzshellext/64/.libs/libfzshellext-0.dll
$TARGET_HOST-strip src/fzshellext/32/.libs/libfzshellext-0.dll
$TARGET_HOST-strip data/dlls/*.dll
```

### package
```sh
cd data
wine "$HOME/.wine/drive_c/Program Files (x86)/NSIS/makensis.exe" install.nsi
```


