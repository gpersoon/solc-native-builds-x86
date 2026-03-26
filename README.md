# solc-native-builds-x86
Native C++ Solidity Compiler Builds for x86

Collected to be used for https://www.ethereumhistory.com


Single-file x86_64 Linux binaries (~4MB each) with no external library dependencies.

## Prerequisites

- Docker
- Clone of [webthree-umbrella](https://github.com/ethereum/webthree-umbrella) at `../solc/webthree-umbrella`

## Dockerfile

Save as `Dockerfile.static`:

```dockerfile
FROM ubuntu:16.04
RUN apt-get update && apt-get install -y \
    build-essential cmake git \
    libboost-all-dev libjsoncpp-dev \
    libcurl4-openssl-dev libleveldb-dev libminiupnpc-dev \
    libjsonrpccpp-dev libmicrohttpd-dev qtbase5-dev \
    libsnappy-dev libgmp-dev \
    && rm -rf /var/lib/apt/lists/*
RUN cd /tmp && git clone https://github.com/weidai11/cryptopp.git && \
    cd cryptopp && git checkout CRYPTOPP_5_6_2 && \
    make -j$(nproc) CXXFLAGS="-DNDEBUG -g2 -O2 -fPIC" && \
    mkdir -p /usr/include/cryptopp && \
    cp *.h /usr/include/cryptopp/ && \
    cp libcryptopp.a /usr/lib/x86_64-linux-gnu/ && ldconfig
ARG COMMIT=bff6f678b
COPY webthree-umbrella /src
WORKDIR /src
RUN git checkout ${COMMIT} || true
RUN rm -rf build && mkdir build && \
    printf '#pragma once\n#define ETH_PROJECT_VERSION "0.0.0"\n#define ETH_COMMIT_HASH 0\n#define ETH_CLEAN_REPO 0\n#define ETH_BUILD_TYPE "Release"\n#define ETH_BUILD_PLATFORM "Linux/g++"\n' > build/BuildInfo.h
RUN cd build && \
    cmake .. -DCMAKE_BUILD_TYPE=Release \
    -DGUI=OFF -DTESTS=OFF -DETHASHCL=OFF -DEVMJIT=OFF -DJSCONSOLE=OFF && \
    make solc -j$(nproc)
RUN cd build && \
    for lib in libsolidity libevmasm libevmcore libdevcore libdevcrypto libscrypt; do \
      ar rcs ${lib}/${lib##lib}.a ${lib}/CMakeFiles/*.dir/*.o 2>/dev/null; \
    done && \
    find secp256k1 -name '*.o' 2>/dev/null | xargs ar rcs secp256k1.a 2>/dev/null; \
    SECP="" && test -s secp256k1.a && SECP="secp256k1.a"; \
    g++ -o solc-static \
      solc/CMakeFiles/solc.dir/*.o \
      -Wl,--whole-archive \
      libsolidity/solidity.a libevmasm/evmasm.a libevmcore/evmcore.a \
      libdevcore/devcore.a libdevcrypto/devcrypto.a \
      libscrypt/scrypt.a $SECP \
      -Wl,--no-whole-archive \
      /usr/lib/x86_64-linux-gnu/libcryptopp.a \
      -Wl,-Bstatic \
      -lboost_filesystem -lboost_program_options -lboost_random \
      -lboost_system -lboost_regex -lboost_thread \
      -ljsoncpp -lleveldb -lsnappy -lgmp \
      -Wl,-Bdynamic -lpthread -ldl && \
    strip solc-static
```

## Build a specific commit

```bash
cd /path/to/parent/of/webthree-umbrella
docker build -f Dockerfile.static -t solc-build --build-arg COMMIT=bff6f678b .
docker run --rm solc-build cat /src/build/solc-static > solc-0.1.1-bff6f678b
chmod +x solc-0.1.1-bff6f678b
```

## Tested commits

| Binary | Version | Date | Commit | Notes |
|--------|---------|------|--------|-------|
| solc-0.9.27-79375056d | 0.9.27 | Jun 25, 2015 | 79375056d | Memory arrays merge |
| solc-0.9.27-766c3ee37 | 0.9.27 | Jun 26, 2015 | 766c3ee37 | Calldata array fixes |
| solc-0.9.27-f95baf2cb | 0.9.27 | Jun 26, 2015 | f95baf2cb | Memory object delete |
| solc-0.9.28-d3ff38144 | 0.9.28 | Jun 30, 2015 | d3ff38144 | Memory structs |
| solc-0.9.28-905da13c3 | 0.9.28 | Jun 30, 2015 | 905da13c3 | Struct constructors |
| solc-0.9.28-97dc2d61f | 0.9.28 | Jun 30, 2015 | 97dc2d61f | Array-to-storage fix |
| solc-0.1.0-248eb8866 | 0.1.0 | Jul 14, 2015 | 248eb8866 | Bytes comparison fix (pre-storage-ref) |
| solc-0.1.0-f8a3824f0 | 0.1.0 | Jul 14, 2015 | f8a3824f0 | Single stack slot for storage refs |
| solc-0.1.0-fb8e1b382 | 0.1.0 | Jul 14, 2015 | fb8e1b382 | Merge of storage ref + byte fixes |
| solc-0.1.0-aa4f6bb6f | 0.1.0 | Jul 15, 2015 | aa4f6bb6f | Storage strings fix |
| solc-0.1.0-e457d74d1 | 0.1.0 | Jul 15, 2015 | e457d74d1 | Stack slot allowance |
| solc-0.1.0-0ed47e902 | 0.1.0 | Jul 28, 2015 | 0ed47e902 | Gas computation fix |
| solc-0.1.0-e892152ba | 0.1.0 | Jul 31, 2015 | e892152ba | Clone contracts |
| solc-0.1.0-056180fb2 | 0.1.0 | Aug 3, 2015 | 056180fb2 | Strings as mapping keys |
| solc-0.1.1-bff6f678b | 0.1.1 | Aug 4, 2015 | bff6f678b | v0.1.1 release |

## Usage

```bash
./solc-0.1.1-bff6f678b --optimize on --binary stdout contract.sol
```

## Notes

- The `BuildInfo.h` stub is needed for commits before `bb3d31c84` (Jul 8, 2015 "Versioning for Solidity"). Later commits generate it via cmake.
- CryptoPP 5.6.2 must be built from source with `-fPIC` for static linking.
- Commits before Jun 25, 2015 (`79375056d`) lack language features needed for structs-in-mappings and may fail to compile certain contracts.
- secp256k1 is only present in later commits. The Dockerfile handles its absence gracefully.
- The static relink step creates `.a` archives from the cmake-built `.o` files, then links everything into a single binary. Only libc, libstdc++, libpthread, and libdl remain as dynamic dependencies (present on all Linux systems).
