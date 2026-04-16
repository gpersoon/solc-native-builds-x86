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
| solc-0.0.9.24-e66383994 | 0.9.24 | Jun 5, 2015 | e66383994 | Dynamic memory (first with freeMemoryPointer) |
| solc-0.0.9.25-9db5fb5bb | 0.9.23* | Jun 6, 2015 | 9db5fb5bb | Improved "Stack too deep" (pre-ISZERO opt) |
| solc-0.0.9.26-7229fad1b | 0.9.23* | Jun 6, 2015 | 7229fad1b | Optimize double ISZERO (CSE rule change) |
| solc-0.0.9.26+1-2d2c9b3a7 | 0.9.23* | Jun 2015 | 2d2c9b3a7 | Quick fix inaccessible sequences (earliest in CSE gap) |
| solc-0.0.9.26+2-6c45ac6ab | 0.9.25 | Jun 2015 | 6c45ac6ab | Optimize RETURN x 0 → STOP |
| solc-0.0.9.26+3-5eb31c228 | 0.9.25 | Jun 2015 | 5eb31c228 | Source locations for new memory management |
| solc-0.0.9.26+4-5848faeb4 | 0.9.26 | Jun 2015 | 5848faeb4 | Pre-jump output size for contracts in this window |
| solc-0.0.9.26+5-536bd3618 | 0.9.26 | Jun 2015 | 536bd3618 | Type conversion specialities (post-jump) |
| solc-0.0.9.26+6-c75c72a99 | 0.9.26 | Jun 2015 | c75c72a99 | Storage reference conversions |
| solc-0.0.9.26+7-5ba138f09 | 0.9.27 | Jun 2015 | 5ba138f09 | Memory-storage copy (late in CSE gap) |
| solc-0.0.9.27-79375056d | 0.9.27 | Jun 25, 2015 | 79375056d | Memory arrays merge |
| solc-0.0.9.27-766c3ee37 | 0.9.27 | Jun 26, 2015 | 766c3ee37 | Calldata array fixes |
| solc-0.0.9.27-f95baf2cb | 0.9.27 | Jun 26, 2015 | f95baf2cb | Memory object delete |
| solc-0.0.9.28-d3ff38144 | 0.9.28 | Jun 30, 2015 | d3ff38144 | Memory structs |
| solc-0.0.9.28-905da13c3 | 0.9.28 | Jun 30, 2015 | 905da13c3 | Struct constructors |
| solc-0.0.9.28-97dc2d61f | 0.9.28 | Jun 30, 2015 | 97dc2d61f | Array-to-storage fix |
| solc-0.1.0-248eb8866 | 0.1.0 | Jul 14, 2015 | 248eb8866 | Bytes comparison fix (pre-storage-ref) |
| solc-0.1.0-f8a3824f0 | 0.1.0 | Jul 14, 2015 | f8a3824f0 | Single stack slot for storage refs |
| solc-0.1.0-fb8e1b382 | 0.1.0 | Jul 14, 2015 | fb8e1b382 | Merge of storage ref + byte fixes |
| solc-0.1.0-aa4f6bb6f | 0.1.0 | Jul 15, 2015 | aa4f6bb6f | Storage strings fix |
| solc-0.1.0-e457d74d1 | 0.1.0 | Jul 15, 2015 | e457d74d1 | Stack slot allowance |
| solc-0.1.0-0ed47e902 | 0.1.0 | Jul 28, 2015 | 0ed47e902 | Gas computation fix |
| solc-0.1.0-e892152ba | 0.1.0 | Jul 31, 2015 | e892152ba | Clone contracts |
| solc-0.0.9.29-9c3983d1c | 0.9.29 | Jul 9, 2015 | 9c3983d1c | Flexible string literals (pre-versioning, reports 0.9.29) |
| solc-0.1.0-8221a3c4c | 0.1.0 | Jul 16, 2015 | 8221a3c4c | Structs with mappings in memory |
| solc-0.1.0-056180fb2 | 0.1.0 | Aug 3, 2015 | 056180fb2 | Strings as mapping keys |
| solc-0.1.0-40ab01edd | 0.1.0 | Aug 4, 2015 | 40ab01edd | Bytes<->string conversion (post-0.1.1, reports 0.1.0) |
| solc-0.1.0-534cdce9a | 0.1.0 | Aug 4, 2015 | 534cdce9a | Clone value transfer fix (post-0.1.1, reports 0.1.0) |
| solc-0.1.1-bff6f678b | 0.1.1 | Aug 4, 2015 | bff6f678b | v0.1.1 release |
| solc-0.1.1-f23a60c70 | 0.1.1 | Aug 5, 2015 | f23a60c70 | Merge of bytes conversion + clone fix + strings mapping fix |
| solc-0.1.1-4661d4773 | 0.1.1 | Aug 7, 2015 | 4661d4773 | Disallow comparison for reference types |
| solc-0.1.1-888909bee | 0.1.1 | Aug 7, 2015 | 888909bee | Disallow boolean operators for integers |

### Sub-project era (webthree-umbrella with git submodules)

These versions are built from the restructured webthree-umbrella where solidity, libethereum, and libweb3core are separate git submodules.

| Binary | Version | Date | Solidity Commit | Notes |
|--------|---------|------|-----------------|-------|
| solc-0.0.8.2-72f598f5b | 0.8.2 (devcore) | Feb 24, 2015 | monorepo poc-8-tag | PoC-8 era, no libevmasm |
| solc-0.1.3-028f561da | 0.1.3 | Sep 23, 2015 | 028f561da | v0.1.3 release, umbrella e04b786b8 |
| solc-0.1.5-23865e39 | 0.1.5 | Oct 7, 2015 | 23865e39 | Pre-AST-refactor, has --libraries |
| solc-0.1.5-e11e10f8 | 0.1.5 | Oct 7, 2015 | e11e10f8 | Second 0.1.5 build (different commit, same codegen) |
| solc-0.1.6-d41f8b7c | 0.1.6 | Oct 16, 2015 | d41f8b7c | Pre-AST-refactor |
| solc-0.1.7-b4e666cc | 0.1.7 | Nov 17, 2015 | b4e666cc | POST-AST-refactor, formal verification |
| solc-0.2.0-d2f18c73 | 0.2.0+110 | Jan 20, 2016 | d2f18c73 | Pre-release of v0.2.1 |
| solc-0.2.0-4dc2445ed | 0.2.0 | Dec 1, 2015 | 4dc2445ed (v0.2.0) | v0.2.0 release tag, umbrella 794ce70ea1 |
| solc-0.2.0-67c855c58 | 0.2.0 nightly | Jan 2016 | 67c855c58 (v0.2.0-138) | Nightly, matches solc-v011/solc-jan20 ARM binaries |
| solc-0.2.1-fad2d4df2 | 0.2.1 | Feb 2016 | fad2d4df2 (v0.2.1-3) | v0.2.1 release, umbrella 794ce70ea1 |
| solc-0.2.2-ef92f5661 | 0.2.2 | Feb 17, 2016 | ef92f5661 (v0.2.2) | v0.2.2 release, umbrella v1.2.0 |
| solc-0.3.0-1f9578ce | 0.3.0 | Mar 11, 2016 | 1f9578ce | v0.3.0 release, breaking changes (types) |
| solc-0.3.2-81ae2a78 | 0.3.2 | Apr 18, 2016 | 81ae2a78 | v0.3.2 release |
| solc-0.3.3-4dc1cb14 | 0.3.3 | May 27, 2016 | 4dc1cb14 | |
| solc-0.3.4-7dab8902 | 0.3.4 | Jun 10, 2016 | 7dab8902 | v0.3.4 release |
| solc-0.3.5-5f97274a | 0.3.5 | Jun 10, 2016 | 5f97274a | v0.3.5 release tag |
| solc-0.3.5-6610add6 | 0.3.5-89 | Jul 2016 | 6610add6 | Latest sub-project era nightly |
| solc-0.3.6-988fe5e5a | 0.3.6 | Aug 10, 2016 | 988fe5e5a (v0.3.6) | Standalone build (post-umbrella removal) |

## Usage

### Monorepo era (0.0.9.x, 0.1.0, 0.1.1)
```bash
./solc-0.1.1-bff6f678b --optimize 1 --binary stdout contract.sol
```

### Sub-project era (0.1.5+)
```bash
./solc-0.1.5-23865e39 --optimize --bin contract.sol
```

Note: CLI changed between eras. Monorepo builds use `--optimize 1 --binary stdout`, sub-project builds use `--optimize --bin`.

## Building from source

### Prerequisites

- Docker
- Clone of [webthree-umbrella](https://github.com/ethereum/webthree-umbrella) at `../solc/webthree-umbrella`

### Monorepo era (0.0.9.x through 0.1.1)

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
| v0.1.5 | fb27f39c4 | 23865e39 (auto) | First 0.1.5 build |
| v0.1.5 | fb27f39c4 | e11e10f8 (override) | Second 0.1.5 build, same codegen |
| v0.1.6 | acbae1e05 | d41f8b7c (override) | Umbrella points to nightly; override to tag |
| v0.1.7 | a760b7fed | b4e666cc (auto) | First with AST refactor + formal verification |
| v0.2.0+110 | a97d01ce8 | d2f18c73 (auto) | ~January 20, 2016 build |
| v0.1.3 | e04b786b8 | 028f561da (override) | v0.1.3 release |
| v0.2.0 | 794ce70ea1 | 4dc2445ed (override) | v0.2.0 release tag |
| v0.2.0-138 | 794ce70ea1 | 67c855c58 (override) | Nightly, matches ARM solc-v011/solc-jan20 |
| v0.2.1 | 794ce70ea1 | fad2d4df2 (auto) | Umbrella's default submodule pointer |
| v0.2.2 | v1.2.0 | ef92f5661 (override) | v0.2.2 release |
| v0.3.0 | (standalone) | 1f9578ce | v0.3.0 release, standalone solidity repo |
| v0.3.2 | (standalone) | 81ae2a78 | v0.3.2 release |
| v0.3.3 | (standalone) | 4dc1cb14 | v0.3.3 release |
| v0.3.4 | (standalone) | 7dab8902 | v0.3.4 release |
| v0.3.5 | (standalone) | 5f97274a | v0.3.5 release tag |
| v0.3.5-89 | 1d9f651b4 | 6610add6 (auto) | Latest sub-project era nightly |
| v0.3.6 | standalone | 988fe5e5a | Standalone build, no umbrella needed |

## JavaScript soljson Wrapper Scripts

Moved to `../solc/`. See `../solc/SOLJSON_ORIGINS.md` for documentation.

## Notes

- The `BuildInfo.h` stub is needed for commits before `bb3d31c84` (Jul 8, 2015 "Versioning for Solidity"). Later commits generate it via cmake.
- CryptoPP 5.6.2 must be built from source with `-fPIC` for static linking.
- Commits before Jun 25, 2015 (`79375056d`, i.e. pre-0.0.9.27) lack language features needed for structs-in-mappings and may fail to compile certain contracts.
- The 0.0.9.24-26 binaries were built natively (not via Docker) by extracting object files from cmake's shared library build and statically linking them. They replace libdevcrypto's CryptoPP-dependent SHA3 with a standalone keccak256 stub linked against libethash's sha3.c. These binaries report version 0.9.23/0.9.24 because the version string wasn't updated between these micro-commits (all from June 5-6, 2015).
- The 3 intermediate commits (e66383994, 9db5fb5bb, 7229fad1b) represent key CSE optimizer transitions: e66383994 introduces freeMemoryPointer (constructor pattern `6060604052`), 9db5fb5bb improves stack-too-deep handling, and 7229fad1b adds double-ISZERO elimination rules in ExpressionClasses. Contracts deployed between these dates may require one of these specific builds for exact bytecode matching.
- secp256k1 is only present in some commits. The Dockerfiles handle its absence gracefully.
- The static relink step creates `.a` archives from the cmake-built `.o` files, then links everything into a single binary. Only libc, libstdc++, libpthread, libdl, and librt remain as dynamic dependencies (present on all Linux systems).
- Sub-project era builds (v0.1.5+) need `libsnappy-dev` installed for static linking (`apt-get install libsnappy-dev`).
- Some sub-project umbrella commits have submodule pointers to "deleted content" cleanup commits. Use umbrella commits from BEFORE the repo cleanup (see table above).
- v0.1.7+ has the AST refactored into subdirectories (`analysis/`, `ast/`, `codegen/`, etc.). The `find` command for `.o` files must be recursive.

## Intermediate CSE-gap builds (`solc-0.0.9.26+N-<hash>`)

The binaries in the `solc-0.0.9.26+N-<hash>` family fill the ~20-commit gap
between `solc-0.0.9.26-7229fad1b` and `solc-0.0.9.27-766c3ee37` where the CSE
(CommonSubexpressionEliminator) optimizer added stack-reuse rules that change
the bytecode output size on many 2015-era contracts.

| Suffix | Commit | Chronological order in the gap |
|---|---|---|
| `+1` | `2d2c9b3a7` | earliest — "Quick fix to not access inaccessible sequences" |
| `+2` | `6c45ac6ab` | "Optimize RETURN x 0 to STOP" |
| `+3` | `5eb31c228` | "Source locations for new memory management" |
| `+4` | `5848faeb4` | pre-jump boundary for many contracts |
| `+5` | `536bd3618` | post-jump boundary |
| `+6` | `c75c72a99` | "Type conversion specialities for storage references" |
| `+7` | `5ba138f09` | "Memory-storage copy" — latest in the gap |

Each is a **single-file ~4.7 MB static ELF** with the same dynamic-dependency
set as the other static binaries in this directory (`libjsoncpp.so.25`,
`libstdc++.so.6`, `libgcc_s.so.1`, `libm.so.6`, `libc.so.6`). Internal solidity
libraries, libcryptopp, libleveldb, libsnappy, and boost are all statically
baked in. No `/tmp` dependency, no runtime unpacking — drop the single file
anywhere and it works.

### Build recipe (native, no Docker)

These 7 binaries are built natively from a local `~/solc/webthree-umbrella-010/`
worktree using the existing Dockerfile static-relink approach but on the host.
Full recipe in `sourcify_data/DECOMPILER_ROUND_TRIP_GUIDE.md` §11 — summary:

```bash
cd ~/solc/webthree-umbrella-010
git worktree add /tmp/solc-build-<hash> <hash>
cd /tmp/solc-build-<hash>

# Modern-cmake / C++11 compatibility patches (in addition to the ones already
# documented in §11)
sed -i 's/cmake_minimum_required(VERSION [0-9.]*)/cmake_minimum_required(VERSION 3.5)/' CMakeLists.txt
find . -name CMakeLists.txt -exec sed -i 's/cmake_policy(SET CMP0042 OLD)/cmake_policy(SET CMP0042 NEW)/g' {} \;
echo 'set(JSON_RPC_CPP_FOUND FALSE)' > cmake/Findjson_rpc_cpp.cmake

# New patch for these intermediate commits — libstdc++'s stricter overload
# resolution rejects the original ambiguous `list<list<T>>` constructor:
sed -i 's|list<list<ContractDefinition const\*>> input(1, {});|list<list<ContractDefinition const*>> input{list<ContractDefinition const*>{}};|' libsolidity/NameAndTypeResolver.cpp

# Build (shared-library pass first to produce the .o files we relink later)
mkdir -p build && cd build
printf '#pragma once\n#define ETH_PROJECT_VERSION "0.0.9.26+N"\n' > BuildInfo.h
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
    -DFATDB=OFF -DEVMJIT=OFF -DGUI=OFF -DETHASHCL=OFF -DTESTS=OFF -DJSONRPC=OFF \
    -DCMAKE_CXX_FLAGS="-std=c++11 -O3 -DNDEBUG -Wno-error -Wno-unused-result -Wno-unused-variable -Wno-unused-parameter -Wno-deprecated-declarations"
make -j$(nproc) solidity

# Static relink: archive each internal lib's .o files and whole-archive them
for lib in libsolidity libevmasm libevmcore libdevcore libdevcrypto libscrypt; do
    rm -f $lib.a
    find $lib -name "*.o" -print0 | xargs -0 -r ar rcs $lib.a
done

g++ -o solc-static \
    solc/CMakeFiles/solc.dir/*.o \
    -Wl,--whole-archive \
    libsolidity.a libevmasm.a libevmcore.a libdevcore.a libdevcrypto.a libscrypt.a \
    -Wl,--no-whole-archive \
    /usr/lib/x86_64-linux-gnu/libcryptopp.a \
    -Wl,-Bstatic -lboost_filesystem -lboost_program_options -lboost_system \
    -lboost_random -lboost_regex -lboost_thread -lboost_chrono \
    -lleveldb \
    -Wl,-Bdynamic /usr/lib/x86_64-linux-gnu/libsnappy.so.1 \
    -ljsoncpp -lstdc++ -lgcc_s -lpthread -ldl

strip solc-static
```

Host prerequisites: `libcryptopp-dev`, `libleveldb-dev`, `libsnappy1v5`,
`libjsoncpp-dev`, `libboost-all-dev`, `patchelf` (only needed if you also want
the non-static bundle form). The resulting `solc-static` is a ~4.7 MB ELF
that can be copied to any x86_64 Linux host with a recent libstdc++.

Ubuntu 22.04's `/usr/lib/x86_64-linux-gnu/libsnappy.so.1` is dynamic-only;
the link uses the full path to pick it up as a normal shared dep (matching
the existing 0.0.9.24-26 binaries which do the same).
