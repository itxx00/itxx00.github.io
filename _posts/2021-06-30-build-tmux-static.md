---
layout: post
title: "how to build a static tmux bin"
categories: [shell]
tags: [bash, ]
---

* Kramdown table of contents
{:toc .toc}

build-tmux-static.sh
```
#!/bin/bash
TARGETDIR=$1
if [ "$TARGETDIR" = "" ]; then
TARGETDIR=$(python -c 'import os; print os.path.realpath("local")')
fi
mkdir -p $TARGETDIR

libevent() {
  curl -LO https://github.com/libevent/libevent/releases/download/release-2.0.22-stable/libevent-2.0.22-stable.tar.gz
  tar -zxvf libevent-2.0.22-stable.tar.gz
  cd libevent-2.0.22-stable
  ./configure --prefix=$TARGETDIR && make && make install
  cd ..
}

ncurses() {
  curl -LO https://ftp.gnu.org/pub/gnu/ncurses/ncurses-6.0.tar.gz
  tar zxvf ncurses-6.0.tar.gz
  cd ncurses-6.0

  ./configure --with-termlib --prefix $TARGETDIR \
              --with-default-terminfo-dir=/usr/share/terminfo \
              --with-terminfo-dirs="/etc/terminfo:/lib/terminfo:/usr/share/terminfo" \
              --enable-pc-files \
              --with-pkg-config-libdir=$TARGETDIR/lib/pkgconfig \
  && make && make install
  cd ..
}

tmux() {
  curl -LO https://github.com/tmux/tmux/releases/download/3.2a/tmux-3.2a.tar.gz
  tar zxvf tmux-3.2a.tar.gz
  cd tmux-3.2a
  PKG_CONFIG_PATH=$TARGETDIR/lib/pkgconfig ./configure --enable-static --prefix=$TARGETDIR && make && make install
  cd ..
  cp $TARGETDIR/bin/tmux .
}

libevent
ncurses
tmux
```
