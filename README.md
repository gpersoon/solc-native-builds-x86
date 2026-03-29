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

### Sub-project era (webthree-umbrella with git submodules)

These versions are built from the restructured webthree-umbrella where solidity, libethereum, and libweb3core are separate git submodules.

| Binary | Version | Date | Solidity Commit | Notes |
|--------|---------|------|-----------------|-------|
| solc-poc8-72f598f5b | 0.8.2 (devcore) | Feb 24, 2015 | monorepo poc-8-tag | PoC-8 era, no libevmasm |
| solc-v015-23865e39 | 0.1.5 | Oct 7, 2015 | 23865e39 | Pre-AST-refactor, has --libraries |
| solc-v016-d41f8b7c | 0.1.6 | Oct 16, 2015 | d41f8b7c | Pre-AST-refactor |
| solc-v017-b4e666cc | 0.1.7 | Nov 17, 2015 | b4e666cc | POST-AST-refactor, formal verification |
| solc-jan20-d2f18c73 | 0.2.0+110 | Jan 20, 2016 | d2f18c73 | Pre-release of v0.2.1 |
| solc-0.2.0-4dc2445ed | 0.2.0 | Dec 1, 2015 | 4dc2445ed (v0.2.0) | v0.2.0 release tag, umbrella 794ce70ea1 |
| solc-0.2.0-67c855c58 | 0.2.0 nightly | Jan 2016 | 67c855c58 (v0.2.0-138) | Nightly, matches solc-v011/solc-jan20 ARM binaries |
| solc-0.2.1-fad2d4df2 | 0.2.1 | Feb 2016 | fad2d4df2 (v0.2.1-3) | v0.2.1 release, umbrella 794ce70ea1 |
| solc-umbrella-6610add6 | 0.3.5-89 | Jul 2016 | 6610add6 | Latest sub-project era |

## Usage

### Monorepo era (0.9.x, 0.1.0, 0.1.1)
```bash
./solc-0.1.1-bff6f678b --optimize 1 --binary stdout contract.sol
```

### Sub-project era (0.1.5+)
```bash
./solc-v015-23865e39 --optimize --bin contract.sol
```

Note: CLI changed between eras. Monorepo builds use `--optimize 1 --binary stdout`, sub-project builds use `--optimize --bin`.

## Building from source

### Prerequisites

- Docker
- Clone of [webthree-umbrella](https://github.com/ethereum/webthree-umbrella) at `../solc/webthree-umbrella`

### Monorepo era (0.9.x through 0.1.1)

These commits exist directly in the webthree-umbrella repo. Use the Dockerfile above with `--build-arg COMMIT=<hash>`.

```bash
docker build -f Dockerfile.static -t solc-build --build-arg COMMIT=bff6f678b .
docker run --rm solc-build cat /src/build/solc-static > solc-0.1.1-bff6f678b
chmod +x solc-0.1.1-bff6f678b
```

### PoC-8 era (pre-evmasm-split)

The poc-8-tag uses the original monorepo cmake system (no webthree-helpers). Different cmake flags are needed:

```bash
# Inside Docker (ubuntu:16.04 with deps installed):
git checkout poc-8-tag
cd build && cmake .. -DCMAKE_BUILD_TYPE=Release -DHEADLESS=ON -DJSONRPC=OFF -DEVMJIT=OFF
make solc -j$(nproc)
# Static relink:
for lib in libsolidity libevmcore libdevcore libdevcrypto; do
  ar rcs ${lib}.a ${lib}/CMakeFiles/*.dir/*.o
done
g++ -o solc-static solc/CMakeFiles/solc.dir/*.o \
  -Wl,--whole-archive libsolidity.a libevmcore.a libdevcore.a libdevcrypto.a -Wl,--no-whole-archive \
  /usr/lib/x86_64-linux-gnu/libcryptopp.a \
  -Wl,-Bstatic -lboost_filesystem -lboost_program_options -lboost_system \
  -lboost_regex -lboost_thread -ljsoncpp -lleveldb -lsnappy -lgmp \
  -Wl,-Bdynamic -lpthread -ldl
strip solc-static
```

### Sub-project era (0.1.5 through 0.3.5)

These require cloning git submodules from separate repos. The umbrella commit determines which submodule versions are used.

```bash
# Inside Docker (ubuntu:16.04 with deps + libsnappy-dev installed):
git clone https://github.com/ethereum/webthree-umbrella.git
cd webthree-umbrella
git checkout <umbrella_commit>  # see table below

# Disable unneeded targets
sed -i 's/add_subdirectory(webthree)/#&/' CMakeLists.txt
sed -i 's/add_subdirectory(alethzero)/#&/' CMakeLists.txt
sed -i 's/add_subdirectory(mix)/#&/' CMakeLists.txt

# Clone required submodules
for mod in solidity libethereum libweb3core webthree-helpers; do
  url=$(grep -A2 "submodule \"$mod\"" .gitmodules | grep url | awk '{print $3}')
  commit=$(git ls-tree HEAD $mod | awk '{print $3}')
  git clone $url $mod && cd $mod && git checkout $commit && cd ..
done

# For v0.1.6: override solidity to exact tag (umbrella may point to nightly)
# cd solidity && git checkout d41f8b7ce702 && cd ..

mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DGUI=OFF -DTESTS=OFF \
  -DETHASHCL=OFF -DEVMJIT=OFF -DJSCONSOLE=OFF -DFATDB=OFF
make solc -j$(nproc)

# Static relink (note: .o files may be in subdirectories)
find . -path "*/libsolidity/CMakeFiles/*" -name "*.o" | xargs ar rcs libsolidity.a
find . -path "*/libevmasm/CMakeFiles/*" -name "*.o" | xargs ar rcs libevmasm.a
find . -path "*/libevmcore/CMakeFiles/*" -name "*.o" | xargs ar rcs libevmcore.a
find . -path "*/libdevcore/CMakeFiles/*" -name "*.o" | xargs ar rcs libdevcore.a

g++ -o solc-static $(find . -path "*/solc/CMakeFiles/solc.dir/*" -name "*.o") \
  -Wl,--whole-archive libsolidity.a libevmasm.a libevmcore.a libdevcore.a -Wl,--no-whole-archive \
  /usr/lib/x86_64-linux-gnu/libcryptopp.a \
  -Wl,-Bstatic -lboost_filesystem -lboost_program_options -lboost_random -lboost_system \
  -lboost_regex -lboost_thread -lboost_chrono -lboost_date_time \
  -ljsoncpp -lleveldb -lsnappy -lgmp \
  -Wl,-Bdynamic -lpthread -ldl -lrt
strip solc-static
```

#### Umbrella commits for sub-project builds

| Version | Umbrella Commit | Solidity Submodule | Notes |
|---------|----------------|--------------------|-------|
| v0.1.5 | fb27f39c4 | 23865e39 (auto) | |
| v0.1.6 | acbae1e05 | d41f8b7c (override) | Umbrella points to nightly; override to tag |
| v0.1.7 | a760b7fed | b4e666cc (auto) | First with AST refactor + formal verification |
| v0.2.0+110 | a97d01ce8 | d2f18c73 (auto) | "jan20" = ~January 20, 2016 build |
| v0.2.0 | 794ce70ea1 | 4dc2445ed (override) | v0.2.0 release tag |
| v0.2.0-138 | 794ce70ea1 | 67c855c58 (override) | Nightly, matches ARM solc-v011/solc-jan20 |
| v0.2.1 | 794ce70ea1 | fad2d4df2 (auto) | Umbrella's default submodule pointer |
| v0.3.5-89 | 1d9f651b4 | 6610add6 (auto) | Latest sub-project era before repo cleanup |

## Notes

- The `BuildInfo.h` stub is needed for commits before `bb3d31c84` (Jul 8, 2015 "Versioning for Solidity"). Later commits generate it via cmake.
- CryptoPP 5.6.2 must be built from source with `-fPIC` for static linking.
- Commits before Jun 25, 2015 (`79375056d`) lack language features needed for structs-in-mappings and may fail to compile certain contracts.
- secp256k1 is only present in some commits. The Dockerfiles handle its absence gracefully.
- The static relink step creates `.a` archives from the cmake-built `.o` files, then links everything into a single binary. Only libc, libstdc++, libpthread, libdl, and librt remain as dynamic dependencies (present on all Linux systems).
- Sub-project era builds (v0.1.5+) need `libsnappy-dev` installed for static linking (`apt-get install libsnappy-dev`).
- Some sub-project umbrella commits have submodule pointers to "deleted content" cleanup commits. Use umbrella commits from BEFORE the repo cleanup (see table above).
- v0.1.7+ has the AST refactored into subdirectories (`analysis/`, `ast/`, `codegen/`, etc.). The `find` command for `.o` files must be recursive.
