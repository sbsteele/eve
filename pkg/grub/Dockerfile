FROM linuxkit/alpine:8b53d842a47fce43464e15f65ee2f68b82542330 AS grub-build

RUN apk add \
  automake \
  make \
  bison \
  gettext \
  flex \
  gcc \
  git \
  libtool \
  libc-dev \
  linux-headers \
  python3 \
  autoconf \
  tar \
  xz \ 
  libressl \
  && apk add ca-certificates && update-ca-certificates

#RUN apk add dpkg --update-cache --repository http://dl-cdn.alpinelinux.org/alpine/v3.6/main/ --allow-untrusted

# because python is not available
RUN ln -s python3 /usr/bin/python

# list of grub modules that are portable between x86_64 and aarch64
ENV GRUB_MODULES_PORT="part_gpt fat ext2 iso9660 squash4 gzio linux acpi normal cpio crypto disk boot crc64 gpt \
search_disk_uuid verify xzio xfs video gfxterm efi_gop gptprio chain probe reboot regexp smbios part_msdos cat echo"
ENV GRUB_MODULES_x86_64="multiboot2 efi_uga"
ENV GRUB_MODULES_aarch64="xen_boot"
ENV GRUB_COMMIT=71f9e4ac44142af52c3fc1860436cf9e432bf764
ENV GRUB_REPO=git://git.sv.gnu.org/grub.git

COPY patches/* /patches/

RUN wget https://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu.tar.xz \
  && tar xf gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu.tar.xz \
  && rm  gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu.tar.xz

ENV PATH $PATH:/gcc-linaro-7.4.1-2019.02-x86_64_aarch64-linux-gnu/bin

RUN mkdir /grub-lib && \
  set -e && \
  git clone ${GRUB_REPO} && \
  cd grub && \
  git checkout -b grub-build ${GRUB_COMMIT} && \
  for patch in /patches/*.patch; do \
    echo "Applying $patch"; \
    patch -p1 < "$patch"; \
  done && \
  ./autogen.sh && \
  ./configure --libdir=/grub-lib --target=aarch64 --with-platform=efi CFLAGS="-Os -Wno-unused-value" && \
    TARGET_CC=aarch64-linux-gnu-gcc TARGET_CFLAGS="-march=cortex-a73" TARGET_OBJCOPY=aarch64-linux-gnu-objcopy && \
    TARGET_STRIP=aarch64-linux-gnu-strip TARGET_NM=aarch64-linux-gnu-nm TARGET_RANLIB=aarch64-linux-gnu-ranlib && \
  ls -l && ls -l cpu && \
  make -j "$(getconf _NPROCESSORS_ONLN)" && \
  make install

# create the grub core image
RUN cd grub ; case "arm64" in \
  amd64) \
    ./grub-mkimage -O x86_64-efi -d /grub-lib/grub/x86_64-efi -o /grub-lib/BOOTX64.EFI -p /EFI/BOOT ${GRUB_MODULES_PORT} ${GRUB_MODULES_x86_64} ;\
    ;; \
  arm64) \
#    wget launchpadlibrarian.net/359460582/grub-efi-arm64-bin_2.02-2ubuntu8_arm64.deb ;\
#    dpkg-deb -x grub-efi-arm64-bin_2.02-2ubuntu8_arm64.deb / ; mkdir /grub-lib/grub/arm64-efi ; cp /usr/lib/grub/arm64-efi/*.* /grub-lib/grub/arm64-efi ;\
    ./grub-mkimage -O arm64-efi -d /grub-lib/grub/arm64-efi -o /grub-lib/BOOTAA64.EFI -p /EFI/BOOT ${GRUB_MODULES_PORT} ${GRUB_MODULES_aarch64} ;\
    ln -s BOOTAA64.EFI /grub-lib/BOOTX64.EFI ;\
    ;; \
  esac

FROM scratch
ENTRYPOINT []
CMD []
WORKDIR /EFI/BOOT
COPY --from=grub-build /grub-lib/BOOT*.EFI ./
COPY rootfs.cfg grub.cfg
