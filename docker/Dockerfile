ARG MUSCLE_VERSION=5.3
ARG MUSCLE_SOURCE_SHA256=74b22a94e630b16015c2bd9ae83aa2be2c2048d3e41f560b2d4a954725c81968
ARG VCXPROJ_MAKE_COMMIT=806d016
ARG VCXPROJ_MAKE_SHA256=902735703004c47705ffa389329378f237fecb154945b489edf6abe260c6694f
ARG DEBIAN_MIRROR=http://deb.debian.org/debian
ARG DEBIAN_SECURITY_MIRROR=http://deb.debian.org/debian-security

FROM debian:12-slim AS builder

ARG MUSCLE_VERSION
ARG MUSCLE_SOURCE_SHA256
ARG VCXPROJ_MAKE_COMMIT
ARG VCXPROJ_MAKE_SHA256
ARG DEBIAN_MIRROR
ARG DEBIAN_SECURITY_MIRROR

ENV DEBIAN_FRONTEND=noninteractive

RUN set -eux; \
    sed -i \
      -e "s|http://deb.debian.org/debian-security|${DEBIAN_SECURITY_MIRROR}|g" \
      -e "s|http://deb.debian.org/debian|${DEBIAN_MIRROR}|g" \
      /etc/apt/sources.list.d/debian.sources; \
    for attempt in 1 2 3 4 5 6; do \
      if apt-get -o Acquire::Retries=3 update \
       && apt-get -o Acquire::Retries=3 install -y --no-install-recommends \
            ca-certificates \
            curl \
            file \
            g++ \
            make \
            python3 \
            tar; then \
        break; \
      fi; \
      if [ "$attempt" = "6" ]; then exit 1; fi; \
      apt-get clean; \
      rm -rf /var/lib/apt/lists/*; \
      sleep 5; \
    done; \
    rm -rf /var/lib/apt/lists/*

WORKDIR /tmp/build

RUN set -eux; \
    curl --retry 5 --retry-delay 2 --retry-all-errors -fsSL \
      "https://github.com/rcedgar/muscle/archive/refs/tags/v${MUSCLE_VERSION}.tar.gz" \
      -o muscle.tar.gz; \
    echo "${MUSCLE_SOURCE_SHA256}  muscle.tar.gz" | sha256sum -c -; \
    tar -xzf muscle.tar.gz --strip-components=1; \
    curl --retry 5 --retry-delay 2 --retry-all-errors -fsSL \
      "https://raw.githubusercontent.com/rcedgar/vcxproj_make/${VCXPROJ_MAKE_COMMIT}/vcxproj_make.py" \
      -o src/vcxproj_make.py; \
    echo "${VCXPROJ_MAKE_SHA256}  src/vcxproj_make.py" | sha256sum -c -; \
    sed -i \
      -e "s/ -march=native//g" \
      -e 's#rc = os.system("make 2> make.stderr | tee make.stdout")#rc = os.system("true")#' \
      src/vcxproj_make.py; \
    chmod +x src/vcxproj_make.py; \
    sed -i "s/defined(__arm64__)/defined(__arm64__) || defined(__aarch64__)/" src/myutils.h; \
    cd src; \
    ./vcxproj_make.py --openmp; \
    printf '"v%s-taffish-r2"\n' "${MUSCLE_VERSION}" > gitver.txt; \
    make -j"$(nproc)"; \
    install -D -m 0755 ../bin/muscle /opt/muscle/bin/muscle; \
    install -D -m 0644 ../README.md /opt/muscle/share/doc/muscle/README.md; \
    install -D -m 0644 ../LICENSE /opt/muscle/share/doc/muscle/LICENSE; \
    printf "%s\n" "${MUSCLE_VERSION}" > /opt/muscle/share/doc/muscle/VERSION; \
    printf "%s\n" "Adds __aarch64__ to MUSCLE's 64-bit platform macro for Linux arm64 builds; removes -march=native for portable container binaries." \
      > /opt/muscle/share/doc/muscle/TAFFISH_PATCHES; \
    /opt/muscle/bin/muscle -version 2>&1 | sed -n "1p" | grep -Fx "muscle ${MUSCLE_VERSION}.linux64 [v${MUSCLE_VERSION}-taffish-r2]" >/dev/null; \
    ldd /opt/muscle/bin/muscle 2>&1 | grep -F "not a dynamic executable" >/dev/null

RUN set -eux; \
    mkdir -p /tmp/muscle-check; \
    cd /tmp/muscle-check; \
    printf ">s1\nACGTACGTACGT\n>s2\nACGTCGTACGT\n>s3\nACGTACGACGT\n" > in.fa; \
    /opt/muscle/bin/muscle -align in.fa -output aln.afa >/dev/null 2> align.log; \
    test -s aln.afa; \
    grep -F ">s1" aln.afa >/dev/null; \
    /opt/muscle/bin/muscle -super5 in.fa -output super5.afa >/dev/null 2> super5.log; \
    test -s super5.afa; \
    /opt/muscle/bin/muscle -align in.fa -diversified -replicates 2 -output ensemble.efa >/dev/null 2> ensemble.log; \
    grep -F "<none.0" ensemble.efa >/dev/null; \
    /opt/muscle/bin/muscle -maxcc ensemble.efa -output maxcc.afa >/dev/null 2> maxcc.log; \
    test -s maxcc.afa; \
    /opt/muscle/bin/muscle -resample ensemble.efa -replicates 1 -output resampled.efa >/dev/null 2> resample.log; \
    test -s resampled.efa; \
    /opt/muscle/bin/muscle -efa_explode ensemble.efa >/dev/null 2> explode.log; \
    test -s none.0; \
    test -s abc.1; \
    cd /; \
    rm -rf /tmp/muscle-check

FROM debian:12-slim

COPY --from=builder /opt/muscle /opt/muscle

RUN set -eux; \
    ln -s /opt/muscle/bin/muscle /usr/local/bin/muscle; \
    mkdir -p /usr/local/share/doc; \
    ln -s /opt/muscle/share/doc/muscle /usr/local/share/doc/muscle; \
    muscle -version 2>&1 | sed -n "1p" | grep -Fx "muscle 5.3.linux64 [v5.3-taffish-r2]" >/dev/null; \
    muscle -h 2>&1 | grep -F "Align FASTA input" >/dev/null; \
    ldd "$(command -v muscle)" 2>&1 | grep -F "not a dynamic executable" >/dev/null

WORKDIR /root

ENV TAFFISH_ENV=TAFFISH
ENV TAFFISH_NAME=muscle
