FROM boot2docker/boot2docker:base
MAINTAINER Steeve Morin "steeve.morin@gmail.com"

ENV ROOTFS          /rootfs
ENV TCL_REPO_BASE   http://tinycorelinux.net/5.x/x86
ENV TCZ_DEPS        iptables \
                    iproute2 \
                    openssh openssl-1.0.0 \
                    tar \
                    gcc_libs \
                    acpid \
                    bash \
                    ncurses-common ncurses-terminfo ncurses ncurses-utils \
                    xz liblzma \
                    git expat2 libiconv libidn libgpg-error libgcrypt libssh2 \
                    nfs-utils tcp_wrappers portmap rpcbind libtirpc \
                    curl ntpclient


RUN apt-get -y install  squashfs-tools \
                        genisoimage \
                        xorriso \
                        syslinux \
                        automake \
                        pkg-config \
                        p7zip-full

# Make the ROOTFS
RUN mkdir -p $ROOTFS

# Install the kernel modules in $ROOTFS
RUN cd /linux-kernel && \
    make INSTALL_MOD_PATH=$ROOTFS modules_install firmware_install

# Remove useless kernel modules, based on unclejack/debian2docker
RUN cd $ROOTFS/lib/modules && \
    rm -rf ./*/kernel/sound/* && \
    rm -rf ./*/kernel/drivers/gpu/* && \
    rm -rf ./*/kernel/drivers/infiniband/* && \
    rm -rf ./*/kernel/drivers/isdn/* && \
    rm -rf ./*/kernel/drivers/media/* && \
    rm -rf ./*/kernel/drivers/staging/lustre/* && \
    rm -rf ./*/kernel/drivers/staging/comedi/* && \
    rm -rf ./*/kernel/fs/ocfs2/* && \
    rm -rf ./*/kernel/net/bluetooth/* && \
    rm -rf ./*/kernel/net/mac80211/* && \
    rm -rf ./*/kernel/net/sunrpc/* && \
    rm -rf ./*/kernel/net/wireless/*

# Install libcap
RUN curl -L ftp://ftp.de.debian.org/debian/pool/main/libc/libcap2/libcap2_2.22.orig.tar.gz | tar -C / -xz && \
    cd /libcap-2.22 && \
    sed -i 's/LIBATTR := yes/LIBATTR := no/' Make.Rules && \
    sed -i 's/\(^CFLAGS := .*\)/\1 -m32/' Make.Rules && \
    make && \
    mkdir -p output && \
    make prefix=`pwd`/output install && \
    mkdir -p $ROOTFS/usr/local/lib && \
    cp -av `pwd`/output/lib64/* $ROOTFS/usr/local/lib

ADD lxc-0.8.0-boot2docker.patch /

# Download LXC, patch it with a 0.8.0 port of @sebp's patch to properly change root from a ramdisk, compile and install
# Based on https://github.com/spahl/lxc/commit/d6b2904d50cac7c44e6f490308b8dd1417281529
RUN curl -L https://github.com/lxc/lxc/archive/lxc-0.8.0.tar.gz | tar -C / -xz && \
    cd /lxc-lxc-0.8.0 && \
    patch -p2 < /lxc-0.8.0-boot2docker.patch && \
    ./autogen.sh && \
    LIBCAPLIB=/libcap-2.22/output/ CFLAGS="-m32 -I${LIBCAPLIB}/include -L${LIBCAPLIB}/lib64" ./configure prefix=/usr/local --disable-apparmor --disable-bash && \
    make && \
    make prefix=$ROOTFS install && \
    mkdir -p $ROOTFS/usr/var/lib/lxc && \
    mkdir -p $ROOTFS/usr/local/lib/lxc/rootfs && \
    mkdir -p $ROOTFS/cgroup && \
    mkdir -p $ROOTFS/dev/mqueue

# Make sure the kernel headers are installed for aufs-util, and then build it
RUN cd /linux-kernel && \
    make INSTALL_HDR_PATH=/tmp/kheaders headers_install && \
    cd / && \
    git clone git://git.code.sf.net/p/aufs/aufs-util && \
    cd /aufs-util && \
    git checkout aufs3.9 && \
    CPPFLAGS="-m32 -I/tmp/kheaders/include" CLFAGS=$CPPFLAGS LDFLAGS=$CPPFLAGS make && \
    DESTDIR=$ROOTFS make install && \
    rm -rf /tmp/kheaders

# Build VBox guest additions
RUN mkdir -p /vboxguest && \
    cd /vboxguest && \
    curl -L -o vboxguest.iso http://download.virtualbox.org/virtualbox/4.3.8/VBoxGuestAdditions_4.3.8.iso && \
    7z x vboxguest.iso -ir'!VBoxLinuxAdditions.run' && \
    sh VBoxLinuxAdditions.run --noexec --target . && \
    mkdir x86 && cd x86 && tar xvjf ../VBoxGuestAdditions-x86.tar.bz2 && cd .. && \
    mkdir amd64 && cd amd64 && tar xvjf ../VBoxGuestAdditions-amd64.tar.bz2 && cd .. && \
    cd amd64/src/vboxguest-4.3.8 && KERN_DIR=/linux-kernel/ make && cd ../../.. && \
    cp amd64/src/vboxguest-4.3.8/*.ko $ROOTFS/lib/modules/$KERNEL_VERSION-tinycore64 && \
    mkdir -p $ROOTFS/sbin && cp x86/lib/VBoxGuestAdditions/mount.vboxsf $ROOTFS/sbin/

# Make sure that all the modules we might have added are recognized
RUN depmod -a -b $ROOTFS $KERNEL_VERSION-tinycore64

# Download the rootfs, don't unpack it though:
RUN curl -L -o /tcl_rootfs.gz $TCL_REPO_BASE/release/distribution_files/rootfs.gz

# Install the TCZ dependencies
RUN for dep in $TCZ_DEPS; do \
        curl -L -o /tmp/$dep.tcz $TCL_REPO_BASE/tcz/$dep.tcz && \
        unsquashfs -f -d $ROOTFS /tmp/$dep.tcz && \
        rm -f /tmp/$dep.tcz ;\
    done

ADD isolinux /isolinux
ADD make_iso.sh /

# Copy over out custom rootfs
ADD rootfs $ROOTFS

# Make sure init scripts are executable
RUN find $ROOTFS/etc/rc.d/ -exec chmod +x {} \; && \
    find $ROOTFS/usr/local/etc/init.d/ -exec chmod +x {} \;

#get the latest docker
RUN curl -L -o $ROOTFS/usr/local/bin/docker https://get.docker.io/builds/Linux/x86_64/docker-latest && \
    chmod +x $ROOTFS/usr/local/bin/docker

CMD ["/bin/sh", "/make_iso.sh"]
