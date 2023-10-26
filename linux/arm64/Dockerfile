ARG crystal_version=1.9.2

FROM --platform=arm64 debian:bookworm AS builder

# Install Crystal dependencies
ARG DEBIAN_FRONTEND=nointeractive
RUN apt-get update && \
    apt-get install -y gcc g++ git make pkg-config \
    llvm-15-dev libevent-dev libpcre2-dev libgmp-dev libssl-dev libxml2-dev libyaml-dev zlib1g-dev

# Build the GC
WORKDIR /tmp/gc
RUN git clone --depth=1 --single-branch --branch=v8.2.4 https://github.com/ivmai/bdwgc.git . && \
    ./autogen.sh && \
    ./configure --disable-debug --disable-shared --enable-large-config --prefix=/usr && \
    make -j$(nproc) CFLAGS="-DNO_GETCONTEXT -pipe -fPIE -O3" && \
    make install

# Build Crystal
WORKDIR /tmp/crystal
RUN git clone --depth=1 --single-branch --branch=${crystal_version} https://github.com/crystal-lang/crystal.git . && \
    gzip -9 man/crystal.1 && \
    mkdir .build && \
    make crystal static=1 release=1 PREFIX=/usr FLAGS="--no-debug -Duse_pcre2" && \
    strip .build/crystal && \
    rm src/llvm/ext/llvm_ext.o

# Build Shards
WORKDIR /tmp/shards
RUN git clone --depth=1 --single-branch --branch=v0.17.3 https://github.com/crystal-lang/shards.git . && \
    gzip -9 man/shards.1 man/shard.yml.5 && \
    make bin/shards FLAGS="--release --no-debug -Duse_pcre2" && \
    strip bin/shards

# Copy binaries to bin
WORKDIR /tmp/bin
COPY /tmp/crystal/.build/crystal .
COPY /tmp/shards/bin/shards .

# Setup link for library
WORKDIR /tmp/lib/crystal
RUN ln -s /tmp/bin/ .
COPY /usr/lib/libgc.a .

# Setup shared resources
WORKDIR /tmp/share
RUN mkdir -p crystal licenses/crystal man/man1 man/man5
COPY /tmp/crystal/src/ crystal/
COPY /tmp/crystal/LICENSE licenses/crystal/
COPY /tmp/crystal/man/man1/crystal.1.gz man/man1/
COPY /tmp/shards/man/man1/shards.1.gz man/man1/
COPY /tmp/shards/man/man5/shard.yml.5.gz man/man5/

# Compress/archive sources
WORKDIR /tmp
RUN tar czf crystal-${crystal_version}-1-linux-arm64.tar.gz /tmp/bin/ /tmp/lib/ /tmp/share/

# Export from scratch
FROM scratch
COPY --from=builder /tmp/crystal-${crystal_version}-1-linux-arm64.tar.gz /
