---
layout: post
title:  "Install mosh on Dreamhost"
desc: "How to install mosh on Dreamhost"
keywords: "linux,mosh"
date: 2016-11-29
categories: [Linux]
tags: [linux]
icon: fa-code
---

A little script to install mosh on Dreamhost

```bash
PREFIX=$HOME
PROTOBUF_VERSION=3.1.0
MOSH_VERSION=1.2.6

# Install Protocol Buffers
wget -O protobuf-$PROTOBUF_VERSION.zip https://github.com/google/protobuf/archive/v$PROTOBUF_VERSION.zip
unzip protobuf-$PROTOBUF_VERSION.zip
cd protobuf-$PROTOBUF_VERSION
./autogen.sh
./configure --prefix=$PREFIX
make
make install
cd ..

# You'll need these setting to have mosh find the Protocol Buffer lib and binary
export PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig
export PATH=$PATH:$HOME/bin

# Install mosh
wget https://mosh.org/mosh-$MOSH_VERSION.tar.gz
tar -xf mosh-$MOSH_VERSION.tar.gz
cd mosh-$MOSH_VERSION
./configure --prefix=$PREFIX
make
make install

echo You can run this to verify the install worked:
echo   $ export LD_LIBRARY_PATH=$PREFIX/lib
echo   $ mosh-server
echo (Running mosh-server should give you a pid and a key to use if you want to connect manually)

echo To connect to the server in the future, run this on your local machine:
echo   $ mosh --server="LD_LIBRARY_PATH=$PREFIX/lib $PREFIX/bin/mosh-server" $USER@$(hostname -f)
```
