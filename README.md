## Fedora RISC-V Installer
本实践旨在制作 RISC-V 平台上的 CD Installer，即可使用安装盘来安装 Fedora 38 on RISC-V
## 大致原理
Fedora 选用的安装工具为 Anaconda，这是一个由 RedHat 开发的框架，可以用于 Fedora 等 RHEL 发行版。\
采用的安装盘制作工具为 Lorax 和 livemedia-creator，前者可以用来创建一个最小的 Anaconda 运行时环境（以 boot.iso 封装），而后者则启动前者创建的 boot.iso 内的 Anaconda 再根据指定的 Fedora kickstart 文件安装真正的安装盘系统，可以以 iso/img/tar/oci 格式等输出
## 流程
### 安装 Lorax 和 livemedia-creator
```shell
dnf install lorax livemedia-creator
```
### 修改 tmpl 文件
tmpl 文件可以告诉 lorax 需要安装什么包，需要执行什么预处理和后处理操作，总之 tmpl 文件描述了生成 boot.iso 的整个流程。由于 RISC-V 的特性和当前 Fedora 对 RISC-V 平台的 workaround，需要修改下 tmpl 文件来适配 RISC-V
1. 将 /usr/share/lorax/templates.d/ 下的 99-generic 文件夹复制一份，取名为 1-riscv，随后修改 runtime-install.tmpl，该文件用来描述运行时所需要依赖的包
2. 此处根据原版 runtime-install.tmpl，针对 Openkoji 仓库的包情况和目前的 Fedora RISC-V 启动逻辑，注释掉了kernel modules，systemd-sysv systemd-units，为 RISC-V 增加了 extlinux-bootloader
```
## lorax template file: populate the ramdisk (runtime image)
<%page args="basearch, product"/>
<%
# This version of grub2 moves the font directory and is needed to keep the efi template from failing.
GRUB2VER="1:2.06-3"
%>

## anaconda package
installpkg anaconda anaconda-widgets kexec-tools-anaconda-addon anaconda-install-img-deps
## Other available payloads
installpkg dnf
installpkg rpm-ostree ostree
## speed up compression on multicore systems
installpkg pigz

## kernel and firmware
## NOTE: Without explicitly including kernel-modules-extra dnf will choose kernel-debuginfo-*
##       to satify a gfs2-utils kmod requirement
installpkg kernel kernel-modules kernel-modules-extra
installpkg grubby
%if basearch not in ("s390x", "riscv64"):
    ## skip the firmware for sound, video, and scanners, none of which will
    ## do much good for the installer. Also skip uhd-firmware which is not
    ## even a kernel firmware package. liquidio and netronome firmwares are
    ## for enterprise switch devices, netinst deployment does not work on
    ## these so there is no point shipping them - see
    ## https://bugzilla.redhat.com/show_bug.cgi?id=2011615
    installpkg --optional *-firmware --except alsa* --except midisport-firmware \
                           --except crystalhd-firmware --except ivtv-firmware \
                           --except cx18-firmware --except iscan-firmware \
                           --except uhd-firmware --except lulzbot-marlin-firmware \
                           --except gnome-firmware --except sigrok-firmware \
                           --except liquidio-firmware --except netronome-firmware \

                           --except mrvlprestera-firmware --except mlxsw_spectrum-firmware
    installpkg b43-openfwwf
%endif

## install all of the glibc langpacks since otherwise we get no locales
installpkg glibc-all-langpacks

## arch-specific packages (bootloaders etc.)
%if basearch == "aarch64":
    installpkg efibootmgr
    installpkg grub2-efi-aa64-cdboot>=${GRUB2VER}
    installpkg shim-aa64
    installpkg uboot-tools
%endif
%if basearch in ("arm", "armhfp"):
    installpkg efibootmgr
    installpkg grub2-efi-arm-cdboot>=${GRUB2VER}
    installpkg grubby-deprecated
    installpkg kernel-lpae
    installpkg uboot-tools
%endif
%if basearch in ("i386", "riscv64"):
    installpkg gpart
%endif
%if basearch == "riscv64":
    installpkg extlinux-bootloader
%endif
%if basearch == "x86_64":
    installpkg grub2-tools-efi>=${GRUB2VER}
    installpkg efibootmgr
    installpkg shim-x64
    installpkg grub2-efi-x64-cdboot>=${GRUB2VER}
    installpkg shim-ia32
    installpkg grub2-efi-ia32-cdboot>=${GRUB2VER}
%endif
%if basearch in ("i386", "x86_64"):
    installpkg biosdevname
    installpkg grub2-tools>=${GRUB2VER} grub2-tools-minimal>=${GRUB2VER}
    installpkg grub2-tools-extra>=${GRUB2VER}
    installpkg grub2-pc-modules>=${GRUB2VER}
%endif
%if basearch == "ppc64le":
    installpkg powerpc-utils lsvpd ppc64-diag
    installpkg grub2-tools>=${GRUB2VER} grub2-tools-minimal>=${GRUB2VER}
    installpkg grub2-tools-extra>=${GRUB2VER} grub2-${basearch}>=${GRUB2VER}
%endif
%if basearch == "ppc64le":
    installpkg extlinux-bootloader
    installpkg uboot-tools
%endif
%if basearch == "s390x":
    installpkg lsscsi s390utils-base s390utils-cmsfs-fuse s390utils-hmcdrvfs
%endif

## yay, plymouth
installpkg plymouth

## extra dracut modules
installpkg anaconda-dracut dracut-network dracut-config-generic

## import-state.service for switchroot
installpkg initscripts

## rescue needs this
installpkg cryptsetup

## rpcbind or portmap needed by dracut nfs module
installpkg rpcbind

## required for dracut
installpkg kbd kbd-misc
## required for anaconda-dracut (img-lib etc.)
installpkg tar xz curl bzip2

## basic system stuff
# Openkoji doesn't have such stuff currently
# installpkg systemd-sysv systemd-units
# installpkg rsyslog

## filesystem tools
installpkg btrfs-progs jfsutils xfsprogs ntfs-3g ntfsprogs
installpkg system-storage-manager
# installpkg device-mapper-persistent-data
installpkg xfsdump

## extra storage packages
# hostname is needed for iscsi to work, see RHBZ#1593917
installpkg udisks2 udisks2-iscsi hostname

## extra libblockdev plugins
installpkg libblockdev-lvm-dbus

## needed for LUKS escrow
installpkg volume_key
installpkg nss-tools

## blivet-gui-runtime requires PolicyKit-authentication-agent, if we
## don't tell dnf what to pick it picks lxpolkit, which drags in gtk2
installpkg polkit-gnome

## SELinux support
installpkg selinux-policy-targeted audit

## network tools/servers
installpkg ethtool openssh-server nfs-utils openssh-clients
installpkg tigervnc-server-minimal
installpkg tigervnc-server-module
installpkg net-tools
installpkg bridge-utils
installpkg nmap-ncat

## hardware utilities/libraries
installpkg pciutils usbutils ipmitool
installpkg mt-st smartmontools
installpkg hdparm
%if basearch not in ("aarch64", "ppc64le", "s390x", "riscv64"):
installpkg pcmciautils
%endif
## see bug #1483278
%if basearch not in ("arm", "armhfp", "riscv64"):
    installpkg libmlx4 rdma-core
%endif
installpkg rng-tools
%if basearch in ("i386", "x86_64", "aarch64"):
installpkg dmidecode
%endif


## fonts & themes
installpkg abattis-cantarell-vf-fonts
installpkg bitmap-fangsongti-fonts
installpkg google-noto-sans-vf-fonts google-noto-sans-mono-vf-fonts
installpkg google-noto-sans-arabic-vf-fonts
# installpkg google-noto-sans-cjk-ttc-fonts
installpkg google-noto-sans-ethiopic-vf-fonts google-noto-sans-georgian-vf-fonts
installpkg google-noto-sans-gurmukhi-vf-fonts google-noto-sans-hebrew-vf-fonts
installpkg google-noto-sans-sinhala-vf-fonts
installpkg jomolhari-fonts
installpkg khmer-os-system-fonts
installpkg lohit-assamese-fonts
installpkg lohit-bengali-fonts
installpkg lohit-devanagari-fonts
installpkg lohit-gujarati-fonts
installpkg lohit-kannada-fonts
installpkg lohit-marathi-fonts
installpkg lohit-odia-fonts
installpkg lohit-tamil-fonts
installpkg lohit-telugu-fonts
installpkg paktype-naskh-basic-fonts
installpkg sil-padauk-fonts
installpkg rit-meera-new-fonts
installpkg thai-scalable-waree-fonts

## debugging/bug reporting tools
installpkg gdb-gdbserver
installpkg libreport-plugin-bugzilla libreport-plugin-reportuploader
installpkg fpaste
installpkg python3-pyatspi

## extra tools not required by anaconda
installpkg nano nano-default-editor
# strace not available
# installpkg vim-minimal strace lsof dump xz less
installpkg vim-minimal lsof dump xz less
installpkg wget rsync bind-utils ftp mtr vconfig
installpkg spice-vdagent
installpkg gdisk hexedit sg3_utils

# Fix some packages missing
installpkg gstreamer1-plugins-base
installpkg libvorbis

## actually install all the requested packages
run_pkg_transaction
```
3. 修改 runtime-postinstall.tmpl，改动包括注释mv auditd，以及由于fedora仓库里的pyanaconda的storage driver还没对RISC-V做适配，因此在这里打个patch
```
## runtime-postinstall.tmpl
## post-install setup required to make the system work.

<%page args="root, basearch, libdir, configdir"/>
<%
configdir = configdir + "/common"
import os, time
SOURCE_DATE_EPOCH = os.environ.get('SOURCE_DATE_EPOCH', str(int(time.time())))
%>

## move_stubs()
move usr/share/anaconda/list-harddrives-stub usr/bin/list-harddrives

## move_repos()
move etc/yum.repos.d etc/anaconda.repos.d

## Setup mdadm config to turn off homehost
remove etc/mdadm.conf
append etc/mdadm.conf "HOMEHOST <ignore>\n"

## Configure systemd to start anaconda
remove etc/systemd/system/default.target
symlink /lib/systemd/system/anaconda.target etc/systemd/system/default.target

## Make sure tmpfs is enabled
mkdir etc/systemd/system/local-fs.target.wants/
symlink /lib/systemd/system/tmp.mount etc/systemd/system/local-fs.target.wants/tmp.mount

## Disable unwanted systemd services
systemctl disable systemd-readahead-collect.service \
                  systemd-readahead-replay.service \
                  lvm2-monitor.service \
                  dnf-makecache.timer
## These services can't be disabled normally (they're linked into place in
## /usr/lib/systemd rather than /etc/systemd), so we have to mask them.
systemctl mask fedora-configure.service fedora-loadmodules.service \
               fedora-autorelabel.service fedora-autorelabel-mark.service \
               fedora-wait-storage.service media.mount \
               systemd-tmpfiles-clean.service systemd-tmpfiles-clean.timer \
               ldconfig.service
remove usr/lib/systemd/system/rngd.service

## remove because it cannot be disabled
remove usr/lib/systemd/system-generators/lvm2-activation-generator


## Remove the more terrible parts of systemd-tmpfiles.
## etc.conf is written with the assumption that /etc/ is empty, which is
## ridiculous, and it also creates a broken /etc/resolv.conf, which breaks
## networking.
remove usr/lib/tmpfiles.d/etc.conf

## Make logind activate anaconda-shell@.service on switch to empty VT
symlink anaconda-shell@.service lib/systemd/system/autovt@.service
replace "#ReserveVT=6" "ReserveVT=2" etc/systemd/logind.conf

## Don't write the journal to the overlay, just keep it in RAM
remove var/log/journal

## install some basic configuration files
append etc/fstab ""
install ${configdir}/i18n etc/sysconfig
install ${configdir}/rsyslog.conf etc
install ${configdir}/bash_history root/.bash_history
install ${configdir}/profile root/.profile
install ${configdir}/libuser.conf etc
install ${configdir}/sysctl.conf etc/sysctl.d/anaconda.conf
install ${configdir}/spice-vdagentd etc/sysconfig
mkdir etc/NetworkManager/conf.d
install ${configdir}/91-anaconda-autoconnect-slaves.conf etc/NetworkManager/conf.d
install ${configdir}/vconsole.conf etc
install ${configdir}/92-anaconda-loglevel-debug.conf etc/NetworkManager/conf.d

## set up sshd
install ${configdir}/sshd_config.anaconda etc/ssh
install ${configdir}/pam.sshd etc/pam.d/sshd
install ${configdir}/pam.sshd etc/pam.d/login
install ${configdir}/pam.sshd etc/pam.d/remote

## set up inst.rngd support
install ${configdir}/inst.rngd.service etc/systemd/system/inst.rngd.service
mkdir etc/systemd/system/basic.target.wants/
symlink /etc/systemd/system/inst.rngd.service etc/systemd/system/basic.target.wants/inst.rngd.service

## set up "install" user account
append etc/passwd "install:x:0:0:root:/root:/usr/libexec/anaconda/run-anaconda"
append etc/shadow "install::14438:0:99999:7:::"
## remove root password
replace "root:\*:" "root::" etc/shadow

## gsettings settings
install ${configdir}/org.gtk.Settings.Debug.gschema.override usr/share/glib-2.0/schemas
runcmd chroot ${root} glib-compile-schemas /usr/share/glib-2.0/schemas

# RISC-V: No need to move that file
# move usr/libexec/anaconda/auditd sbin

# Fix anaconda not supporting RISC-V
runcmd /usr/bin/curl -L https://github.com/MilkiceForks/anaconda/commit/8b0fde72a1d83373cc2572010cf5d6871096e9e3.patch -o ${root}/riscv.patch
runcmd chroot ${root} /usr/bin/patch /usr/lib64/python3.11/site-packages/pyanaconda/modules/storage/platform.py riscv.patch

## for compatibility with Ancient Anaconda Traditions
symlink lib/modules /modules
symlink lib/firmware /firmware
symlink ../run/install mnt/install

## create_depmod_conf()
append etc/depmod.d/dd.conf "search updates built-in"

## create multipath.conf so multipath gets auto-started
append etc/multipath.conf "defaults {\n\tfind_multipaths smart\n\tuser_friendly_names yes\n}\n"
append etc/multipath.conf "blacklist_exceptions {\n\tproperty \"(SCSI_IDENT_|ID_WWN)\"\n}\n"

## make lvm auto-activate
remove etc/lvm/archive/*
remove etc/lvm/archive
remove etc/lvm/backup/*
remove etc/lvm/backup
remove etc/lvm/cache/*
remove etc/lvm/cache
remove etc/lvm/lvm.conf
append etc/lvm/lvm.conf "global {\n\tuse_lvmetad = 1\n}\n"

## TODO: we could run prelink here if we wanted?

## fix fonconfig cache containing timestamps
runcmd chroot ${root} /usr/bin/find /usr/share/fonts -newermt "@${SOURCE_DATE_EPOCH}" -exec \
    touch --no-dereference --date="@${SOURCE_DATE_EPOCH}" {} +
runcmd chroot ${root} /usr/bin/fc-cache -f
```

4. 修改 runtime-cleanup.tmpl，注释了清理boot的命令，同时由于/lib64/lp64d软链接到了/lib64导致find命令在编译/usr目录会报错，因此加了个 || true 强制通过检测
```
## lorax template file: cleanup for the ramdisk (runtime image)
<%page args="libdir, branding, root"/>

## remove the sources
remove usr/share/i18n

## not required packages installed as dependencies
## perl is needed on s390x
## perl needed for powerpc-utils
## perl is needed by /usr/bin/rxe_cfg from libibverbs

## no sound support, thanks
removepkg flac-libs libsndfile pipewire pulseaudio* rtkit sound-theme-freedesktop wireplumber*
## we don't create new initramfs/bootloader conf inside anaconda
## (that happens inside the target system after we install dracut/grubby)
removepkg dracut-network grubby anaconda-dracut
## In order to execute the /usr move on upgrades we need convertfs from dracut
## We also need dracut-shutdown.service and dracut-initramfs-restore to reboot
removefrom dracut --allbut /usr/lib/dracut/modules.d/30convertfs/convertfs.sh \
                  /usr/lib/dracut/modules.d/99base/dracut-lib.sh \
                  /usr/lib/systemd/* /usr/lib/dracut/modules.d/98dracut-systemd/*.service \
                  /usr/lib/dracut/dracut-initramfs-restore
## we don't run SELinux (not in enforcing, anyway)
removepkg selinux-policy libselinux-utils

## selinux checks for the /etc/selinux/config file's existance
## The removepkg above removes it, create an empty one. See rhbz#1243168
append etc/selinux/config ""

## keep enough of shadow-utils to create accounts
removefrom shadow-utils --allbut /usr/bin/chage /usr/sbin/chpasswd \
                        /usr/sbin/groupadd /usr/sbin/useradd

## no services to turn on/off (keep the /etc/init.d link though)
removefrom initscripts /usr/sbin/* /usr/share/locale/* /usr/share/doc/* /usr/share/man/*

## no storage device monitoring
removepkg device-mapper-event
## logrotate isn't useful in anaconda
remove /etc/logrotate.d
## anaconda needs this to do media check
removefrom isomd5sum --allbut /usr/bin/checkisomd5

## there's no need for a bunch of zsh files without zsh,
## systemd-analyze is quite large and not essential
removefrom systemd /usr/bin/systemd-analyze /usr/share/zsh/site-functions/*

## we only need syslinux to make the installer image bootable, we don't
## run anything from it that uses mtools, and that's the only thing
## that pulls in glibc-gconv-extra
removepkg mtools glibc-gconv-extra

## various other things we remove to save space
removepkg diffutils file
removepkg libasyncns
removepkg libtiff
removepkg lvm2-libs
removepkg mobile-broadband-provider-info
removepkg rmt rpcbind squashfs-tools
removepkg tigervnc-license xml-common
removepkg mkfontscale fonttosfnt
removepkg xorg-x11-server-common
# do not remove this, required for ppc64le and s390x !!!
removepkg ncurses

## other removals
remove /home /media /opt /srv /tmp/*
remove /usr/etc /usr/games /usr/local /usr/tmp
remove /usr/share/doc /usr/share/info /usr/share/man /usr/share/gnome
remove /usr/share/mime/application /usr/share/mime/audio /usr/share/mime/image
remove /usr/share/mime/inode /usr/share/mime/message /usr/share/mime/model
remove /usr/share/mime/multipart /usr/share/mime/packages /usr/share/mime/text
remove /usr/share/mime/video /usr/share/mime/x-content /usr/share/mime/x-epoc
remove /var/db /var/games /var/tmp /var/yp /var/nis /var/opt /var/local
remove /var/mail /var/spool /var/preserve /var/report
remove /usr/lib/sysimage/rpm/* /var/lib/rpm/* /var/lib/yum /var/lib/dnf
## clean up the files created by various '> /dev/null's
remove /dev/*

## icons cache
remove /usr/share/icons/*/icon-theme.cache

## clean up kernel modules
removekmod sound drivers/media drivers/hwmon drivers/iio \
           net/atm net/bluetooth net/sched net/sctp \
           net/rds net/l2tp net/decnet net/netfilter net/ipv4 net/ipv6 \
           drivers/watchdog drivers/rtc drivers/input/joystick \
           drivers/bluetooth drivers/edac drivers/staging \
           drivers/usb/serial drivers/usb/host drivers/usb/misc \
           fs/ocfs2 fs/ceph fs/nfsd fs/ubifs fs/nilfs2 \
           arch/x86/kvm
## Need to keep virtio_console.ko and ipmi stuff in drivers/char
## Also keep virtio-rng so that the installer can get sufficient randomness for
## LUKS setup. As of 2020-09 this is not built as a module, but keep it in here
## in case that changes again
removekmod drivers/char --allbut virtio_console hw_random \
                                  virtio-rng ipmi hmcdrv
removekmod drivers/hid --allbut hid-logitech-dj hid-logitech-hidpp

## As of 2020-09 most of this are built-in too, but again, keep them listed
removekmod drivers/video --allbut hyperv_fb syscopyarea sysfillrect sysimgblt fb_sys_fops
remove lib/modules/*/{build,source,*.map}
## NOTE: depmod gets re-run after cleanup finishes

## remove unused themes, theme engines, icons, etc.
removefrom gtk3 /usr/${libdir}/gtk-3.0/*/printbackends/*
removefrom gtk3 /usr/share/themes/*

## filesystem tools
removefrom e2fsprogs /usr/share/locale/*
removefrom xfsprogs /usr/share/locale/* /usr/share/doc/* /usr/share/man/*
removefrom xfsdump --allbut /usr/sbin/*

## other package specific removals
removefrom gsettings-desktop-schemas /usr/share/locale/*
removefrom NetworkManager-libnm /usr/share/locale/*/NetworkManager.mo
removefrom nm-connection-editor /usr/share/applications/*
removefrom atk /usr/share/locale/*
removefrom audit /etc/* /usr/sbin/auditctl /usr/sbin/aureport
removefrom audit /usr/sbin/ausearch /usr/sbin/autrace /usr/bin/*
removefrom audit-libs /etc/* /usr/${libdir}/libauparse*
removefrom bash /etc/* /usr/bin/bashbug* /usr/share/*
removefrom bind-utils /usr/bin/host /usr/bin/nsupdate
removefrom bitmap-fangsongti-fonts /usr/share/fonts/*
removefrom ca-certificates /etc/pki/java/*
removefrom ca-certificates /etc/pki/tls/certs/ca-bundle.trust.crt
removefrom coreutils /usr/bin/link /usr/bin/nice /usr/bin/stty /usr/bin/unlink
removefrom coreutils /usr/bin/[ /usr/bin/base64 /usr/bin/chcon
removefrom coreutils /usr/bin/cksum /usr/bin/csplit
removefrom coreutils /usr/bin/dir /usr/bin/dircolors
removefrom coreutils /usr/bin/expand /usr/bin/factor
removefrom coreutils /usr/bin/fold /usr/bin/groups /usr/bin/hostid
removefrom coreutils /usr/bin/install /usr/bin/join /usr/bin/logname
removefrom coreutils /usr/bin/mkfifo /usr/bin/nl /usr/bin/nohup /usr/bin/nproc
removefrom coreutils /usr/bin/pathchk
removefrom coreutils /usr/bin/pinky /usr/bin/pr /usr/bin/printenv
removefrom coreutils /usr/bin/printf /usr/bin/ptx /usr/bin/runcon
removefrom coreutils /usr/bin/sha224sum /usr/bin/sha384sum
removefrom coreutils /usr/bin/sha512sum /usr/bin/shuf /usr/bin/stat
removefrom coreutils /usr/bin/stdbuf /usr/bin/sum /usr/bin/test
removefrom coreutils /usr/bin/timeout /usr/bin/truncate /usr/bin/tsort
removefrom coreutils /usr/bin/unexpand /usr/bin/users /usr/bin/vdir
removefrom coreutils /usr/bin/who /usr/bin/whoami /usr/bin/yes
removefrom coreutils-common /etc/* /usr/share/*
removefrom cpio /usr/share/*
removefrom cracklib /usr/sbin/*
removefrom cracklib-dicts /usr/${libdir}/* /usr/sbin/*
removefrom cryptsetup /usr/share/*
removefrom cryptsetup-libs /usr/share/locale/*
removefrom cyrus-sasl-lib /usr/sbin/* /usr/bin/*
removefrom dbus-x11 /etc/X11/*
removefrom dnf /usr/share/locale/*
removefrom dump /etc/*
removefrom elfutils-libelf /usr/share/locale/*
removefrom expat /usr/bin/*
removefrom fcoe-utils /usr/libexec/fcoe/dcbcheck.sh
removefrom fcoe-utils /usr/libexec/fcoe/fcc.sh /usr/libexec/fcoe/fcoe-setup.sh
removefrom fcoe-utils /usr/libexec/fcoe/fcoedump.sh /usr/sbin/fcnsq
removefrom fcoe-utils /usr/sbin/fcoeadm /usr/sbin/fcping /usr/sbin/fcrls
removefrom file-libs /usr/share/*
removefrom findutils /usr/share/*
removefrom fontconfig /usr/bin/*
removefrom gawk /usr/libexec/* /usr/share/*
removefrom gdb /usr/share/* /usr/include/*
removefrom gdb-headless /usr/share/* /etc/gdbinit*
removefrom gdisk /usr/share/*
removefrom gdk-pixbuf2 /usr/share/locale*
removefrom glib2 /usr/bin/* /usr/share/locale/*
removefrom glibc /etc/gai.conf /etc/rpc
removefrom glibc /${libdir}/libBrokenLocale*
removefrom glibc /${libdir}/libanl*
removefrom glibc /${libdir}/libnss_compat*
# python-pyudev uses ctypes.util.find_library, which uses /sbin/ldconfig
removefrom glibc /usr/libexec/* /usr/sbin/*
removefrom glibc-common /usr/bin/gencat
removefrom glibc-common /usr/bin/getent
removefrom glibc-common /usr/bin/locale /usr/bin/sprof
# NB: we keep /usr/bin/localedef so anaconda can inspect payload locale info
removefrom glibc-common /usr/bin/tzselect
removefrom glibc-common /usr/sbin/*
removefrom gnutls /usr/share/locale/*
removefrom google-noto-sans-cjk-ttc-fonts /usr/share/fonts/google-noto-cjk/NotoSansCJK-{Black,Bold,*Light,Medium,Thin}.ttc
removefrom google-noto-sans-vf-fonts /usr/share/fonts/google-noto-vf/NotoSans-Italic-VF.ttf
removefrom grep /etc/* /usr/share/locale/*
removefrom gtk3 /usr/${libdir}/gtk-3.0/*
removefrom guile22 /usr/${libdir}/guile/2.2/ccache*
removefrom gzip /usr/bin/{gzexe,zcmp,zdiff,zegrep,zfgrep,zforce,zgrep,zless,zmore,znew}
removefrom hwdata /usr/share/hwdata/oui.txt /usr/share/hwdata/pnp.ids
removefrom iproute --allbut /usr/sbin/{ip,routef,routel,rtpr}
removefrom kbd --allbut */bin/{dumpkeys,kbd_mode,loadkeys,setfont,unicode_*,chvt}
removefrom less /etc/*
removefrom libX11-common /usr/share/X11/XErrorDB
removefrom libcanberra /usr/${libdir}/libcanberra-*
removefrom libcanberra-gtk3 /usr/bin/*
removefrom libcap /usr/sbin/*
removefrom libconfig /usr/${libdir}/libconfig++*
removefrom libgpg-error /usr/bin/* /usr/share/locale/*
removefrom libibverbs /usr/${libdir}/libmlx4*
removefrom libidn2 /usr/share/locale/*
removefrom libnotify /usr/bin/*
removefrom libsemanage /etc/selinux/*
removefrom libstdc++ /usr/share/*
removefrom libvorbis /usr/${libdir}/libvorbisenc.*
removefrom libxml2 /usr/bin/*
removefrom linux-firmware /usr/lib/firmware/dvb*
removefrom linux-firmware /usr/lib/firmware/*_12mhz*
removefrom linux-firmware /usr/lib/firmware/v4l*
removefrom linux-firmware /usr/lib/firmware/brcm/BCM-*
removefrom linux-firmware /usr/lib/firmware/ttusb-budget/dspbootcode.bin*
removefrom linux-firmware /usr/lib/firmware/emi26/*
removefrom linux-firmware /usr/lib/firmware/emi62/*
removefrom linux-firmware /usr/lib/firmware/cpia2/*
removefrom linux-firmware /usr/lib/firmware/dabusb/*
removefrom linux-firmware /usr/lib/firmware/vicam/*
removefrom linux-firmware /usr/lib/firmware/dsp56k/*
removefrom linux-firmware /usr/lib/firmware/sun/*
removefrom linux-firmware /usr/lib/firmware/av7110/*
removefrom linux-firmware /usr/lib/firmware/usbdux*
removefrom linux-firmware /usr/lib/firmware/f2255usb.bin*
removefrom linux-firmware /usr/lib/firmware/lgs8g75.fw*
removefrom linux-firmware /usr/lib/firmware/TDA7706*
removefrom linux-firmware /usr/lib/firmware/tlg2300_firmware.bin*
removefrom linux-firmware /usr/lib/firmware/s5p-mfc*
removefrom linux-firmware /usr/lib/firmware/go7007/*
removefrom linux-firmware /usr/lib/firmware/intel/IntcSST2.bin*
removefrom linux-firmware /usr/lib/firmware/intel/fw_sst*
removefrom linux-firmware /usr/lib/firmware/intel/dsp*
removefrom linux-firmware /usr/lib/firmware/as102*
removefrom linux-firmware /usr/lib/firmware/qcom/apq8096/*
removefrom linux-firmware /usr/lib/firmware/qcom/sdm845/*
removefrom linux-firmware /usr/lib/firmware/qcom/sm8250/*
removefrom linux-firmware /usr/lib/firmware/qcom/venus*/*
removefrom linux-firmware /usr/lib/firmware/qcom/vpu*/*
removefrom linux-firmware /usr/lib/firmware/meson/vdec/*
removefrom linux-firmware /usr/lib/firmware/phanfw.bin*
%if basearch != "aarch64":
    removefrom linux-firmware /usr/lib/firmware/dpaa2/*
%endif
removefrom lldpad /etc/*
removefrom mdadm /etc/* /usr/lib/systemd/system/mdmonitor*
## gallium-pipe stuff is for compute (opencl), not needed for video
removefrom mesa-dri-drivers /usr/${libdir}/dri/*_video.so /usr/lib64/gallium-pipe/*
removefrom mt-st /usr/sbin/*
removefrom mtools /etc/*
removefrom ncurses-libs /usr/${libdir}/libform*
## libmenu.so is needed by lp_diag binary from ppc64-diag which is a PowerPC specific package
%if basearch != "ppc64le":
    removefrom ncurses-libs /usr/${libdir}/libmenu*
%endif
removefrom ncurses-libs /usr/${libdir}/libpanel.* /usr/${libdir}/libtic*
removefrom net-tools */bin/netstat */sbin/ether-wake */sbin/ipmaddr
removefrom net-tools */sbin/iptunnel */sbin/mii-diag */sbin/mii-tool
removefrom net-tools */sbin/nameif */sbin/plipconfig */sbin/slattach
removefrom net-tools /usr/share/locale/*
removefrom nfs-utils /etc/nfsmount.conf
removefrom nfs-utils /usr/lib/systemd/system/*
removefrom nfs-utils /sbin/rpc.statd /usr/sbin/exportfs
removefrom nfs-utils /usr/sbin/mountstats /usr/sbin/nfsiostat
removefrom nfs-utils /usr/sbin/nfsstat /usr/sbin/rpc.gssd /usr/sbin/rpc.idmapd
removefrom nfs-utils /usr/sbin/rpc.mountd /usr/sbin/rpc.nfsd
removefrom nfs-utils /usr/sbin/rpcdebug
removefrom nfs-utils /usr/sbin/showmount /usr/sbin/sm-notify
removefrom nfs-utils /usr/sbin/start-statd /var/lib/nfs/etab
removefrom nfs-utils /var/lib/nfs/rmtab /var/lib/nfs/statd/state
removefrom nss-softokn /usr/${libdir}/nss/*
removefrom openldap /etc/openldap/*
removefrom openssh /usr/libexec/*
removefrom openssh-clients /etc/ssh/* /usr/bin/ssh-*
removefrom openssh-clients /usr/libexec/*
removefrom openssh-server /etc/ssh/* /usr/libexec/openssh/sftp-server
removefrom pam /usr/sbin/* /usr/share/locale/*
removefrom policycoreutils /etc/* /usr/bin/* /usr/share/locale/*
removefrom polkit /usr/bin/*
removefrom popt /usr/share/locale/*
removefrom procps-ng /usr/bin/free /usr/bin/pgrep /usr/bin/pkill
removefrom procps-ng /usr/bin/pmap /usr/bin/pwdx /usr/bin/skill /usr/bin/slabtop
removefrom procps-ng /usr/bin/snice /usr/bin/tload /usr/bin/uptime
removefrom procps-ng /usr/bin/vmstat /usr/bin/w /usr/bin/watch
removefrom psmisc /usr/share/locale/*
removefrom python3-kickstart /usr/lib/python*/site-packages/pykickstart/locale/*
removefrom readline /usr/${libdir}/libhistory*
removefrom libreport /usr/share/locale/*
removefrom rdma-core /etc/rdma/mlx4.conf
removefrom rpm /usr/bin/* /usr/share/locale/*
removefrom rsync /etc/*
removefrom sed /usr/share/locale/*
removefrom smartmontools /etc/* /usr/sbin/smartd
removefrom smartmontools /usr/sbin/update-smart-drivedb
removefrom smartmontools /usr/share/smartmontools/*
removefrom tar /usr/share/locale/*
removefrom usbutils /usr/bin/*
removefrom util-linux --allbut \
    /usr/bin/{chmem,eject,getopt,hexdump,login,lscpu,lsmem,lsblk} \
    /etc/pam.d/login /etc/pam.d/remote \
    /usr/sbin/{clock,fdisk,fsfreeze,fstrim,hwclock,nologin,sfdisk,swaplabel,wipefs,zramctl}
removefrom util-linux-core --allbut \
    /usr/bin/{dmesg,findmnt,flock,kill,logger,more,mount,mountpoint,umount,unshare} \
    /etc/mtab \
    /usr/sbin/{agetty,blkid,blockdev,fsck,losetup,mkswap,partx,swapoff,swapon}
removefrom volume_key-libs /usr/share/locale/*
removefrom wget /etc/* /usr/share/locale/*
removefrom wpa_supplicant /usr/sbin/eapol_test
removefrom xorg-x11-drv-intel /usr/${libdir}/libI*
removefrom xorg-x11-drv-wacom /usr/bin/*
removefrom yelp /usr/share/yelp/mathjax*

%if branding.release:
    removefrom ${branding.logos} /usr/share/plymouth/*
    removefrom ${branding.logos} /etc/*
    removefrom ${branding.logos} /usr/share/icons/{Bluecurve,oxygen}/*
    removefrom ${branding.logos} /usr/share/{kde4,pixmaps}/*
%endif

## cleanup /boot/ leaving vmlinuz, and .*hmac files
# runcmd chroot ${root} find /boot \! -name "vmlinuz*" \
#                            -and \! -name ".vmlinuz*" \
#                            -and \! -name boot -delete

## remove any broken links in /etc or /usr
## (broken systemd service links lead to confusing noise at boot)
## NOTE: not checking /var because we want to keep /var/run
## NOTE: Excluding /etc/mtab which links to /proc/self/mounts for systemd
runcmd chroot ${root} bash -c 'find -L /etc /usr -xdev -type l -and \! -name "mtab" -printf "removing broken symbolic link %p -> %l\n" -delete || true'

## Remove compiled python files, they are recreated as needed anyway
runcmd find ${root} -name "*.pyo" -type f -delete
runcmd find ${root} -name "*.pyc" -type f -delete

## Clean up some of the mess pulled in by webkitgtk via yelp
## libwebkit2gtk links to a handful of libraries in gstreamer and
## gstreamer-plugins-base. Remove the rest of them.
removefrom gstreamer1 --allbut /usr/${libdir}/libgstbase-1.0.* \
                               /usr/${libdir}/libgstreamer-1.0.*
removefrom gstreamer1-plugins-base --allbut \
        /usr/${libdir}/libgst{allocators,app,audio,fft,gl,pbutils,tag,video}-1.0.*

## We have enough geoip libraries, thanks
removepkg geoclue2

## And remove the packages that those extra libraries pulled in
removepkg cdparanoia-libs opus libtheora libvisual flac-libs gsm avahi-glib avahi-libs \
          ModemManager-glib

## metacity requires libvorbis and libvorbisfile, but enc/dec are no longer needed
removefrom libvorbis --allbut /usr/${libdir}/libvorbisfile.* /usr/${libdir}/libvorbis.*

## Remove build-id links, they are used with debuginfo
remove /usr/lib/.build-id

## make the image more reproducible

## make machine-id empty but present to avoid systemd populating /etc with
## preset settings
remove /etc/machine-id
append /etc/machine-id ""
## journalctl message catalog, non-deterministic
remove /var/lib/systemd/catalog/database
## non-reproducible caches
remove /var/cache/ldconfig/aux-cache
remove /etc/pki/ca-trust/extracted/java/cacerts

## sort groups
runcmd chroot ${root} /bin/sh -c "LC_ALL=C sort /etc/group > /etc/group- && mv /etc/group- /etc/group"
runcmd chroot ${root} /bin/sh -c "LC_ALL=C sort /etc/gshadow > /etc/gshadow- && mv /etc/gshadow- /etc/gschadow"
```
5. 运行 lorax 命令创建 boot.iso `lorax -p Fedora -v 38 -r 38 --buildarch riscv64 -s http://openkoji.iscas.ac.cn/kojifiles/repos/f38-build-side-42-init-devel/latest/riscv64/ --nomacboot --logfile ~/lorax.log --noverify --workdir ~/lorax_ws ~/lorax_output`
6. 使用 livemedia-creator 创建启动磁盘，这里文档提供了很多方案，经过测试后采用 [Mock + no-virt](https://weldr.io/lorax/livemedia-creator.html#using-mock-and-no-virt-to-create-images) 来创建启动磁盘
7. 修改 mock 配置文件
```
config_opts['chroot_setup_cmd'] = 'install @buildsys-build anaconda-tui lorax'

# build results go into /home/builder/results/
config_opts['plugin_conf']['bind_mount_opts']['dirs'].append(('/home/builder/results','/results/'))
```
8. 执行 `mock -r fedora-38-riscv-lmc --init`
9. 准备一份 kickstart 文件用来描述安装盘需要安装的包和配置项，内容如下，主要是根据fedora默认的kickstart文件再加上RISC-V flavor
```
#version=DEVEL
# X Window System configuration information
xconfig  --startxonboot
# Keyboard layouts
keyboard 'us'

# System timezone
timezone US/Eastern
# System language
lang en_US.UTF-8
# Firewall configuration
firewall --enabled --service=mdns
url --url="http://openkoji.iscas.ac.cn/kojifiles/repos/f38-build-side-42-init-devel/latest/riscv64/"
# Network information
network  --bootproto=dhcp --device=link --activate

# SELinux configuration
selinux --enforcing

# System services
services --disabled="sshd" --enabled="NetworkManager,livesys,livesys-late"

# riscv
zerombr
# livemedia-creator modifications.
shutdown
# riscv: System bootloader configuration
bootloader --location=none --extlinux --timeout=1
# Clear blank disks or all existing partitions
clearpart --all --initlabel
rootpw rootme
# Disk partitioning information
reqpart
part / --size=6656

%post
# enable tmpfs for /tmp
systemctl enable tmp.mount

# make it so that we don't do writing to the overlay for things which
# are just tmpdirs/caches
# note https://bugzilla.redhat.com/show_bug.cgi?id=1135475
cat >> /etc/fstab << EOF
vartmp   /var/tmp    tmpfs   defaults   0  0
EOF

# work around for poor key import UI in PackageKit
#rm -f /var/lib/rpm/__db*
#releasever=$(rpm -q --qf '%{version}\n' --whatprovides system-release)
#basearch=$(uname -i)
#rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-$releasever-$basearch
echo "Packages within this LiveCD"
rpm -qa --qf '%{size}\t%{name}-%{version}-%{release}.%{arch}\n' |sort -rn
# Note that running rpm recreates the rpm db files which aren't needed or wanted
rm -f /var/lib/rpm/__db*

# go ahead and pre-make the man -k cache (#455968)
/usr/bin/mandb

# make sure there aren't core files lying around
rm -f /core*

# remove random seed, the newly installed instance should make it's own
rm -f /var/lib/systemd/random-seed

# convince readahead not to collect
# FIXME: for systemd

echo 'File created by kickstart. See systemd-update-done.service(8).' \
    | tee /etc/.updated >/var/.updated

# Remove the rescue kernel and image to save space
# Installation will recreate these on the target
rm -f /boot/*-rescue*

# Remove machine-id on pre generated images
rm -f /etc/machine-id
touch /etc/machine-id

%end

# Architecture specific packages
# The bootloader package requirements are different, and workstation-product-environment
# is only available on x86_64
%pre
PKGS=/tmp/arch-packages.ks
echo > $PKGS
ARCH=$(uname -m)
case $ARCH in
    x86_64)
        echo "%packages" >> $PKGS
        echo "@^workstation-product-environment" >> $PKGS
        echo "shim" >> $PKGS
        echo "shim-ia32" >> $PKGS
        echo "grub2" >> $PKGS
        echo "grub2-efi" >> $PKGS
        echo "grub2-efi-ia32" >> $PKGS
        echo "grub2-efi-*-cdboot" >> $PKGS
        echo "efibootmgr" >> $PKGS
        echo "%end" >> $PKGS
    ;;
    aarch64)
        echo "%packages" >> $PKGS
        echo "efibootmgr" >> $PKGS
        echo "grub2-efi-aa64-cdboot" >> $PKGS
        echo "shim-aa64" >> $PKGS
        echo "%end" >> $PKGS
    ;;
    ppc64le)
        echo "%packages" >> $PKGS
        echo "powerpc-utils" >> $PKGS
        echo "grub2-tools" >> $PKGS
        echo "grub2-tools-minimal" >> $PKGS
        echo "grub2-tools-extra" >> $PKGS
        echo "grub2-ppc64le" >> $PKGS
        echo "%end" >> $PKGS
    ;;
    s390x)
        echo "%packages" >> $PKGS
        echo "s390utils-base" >> $PKGS
        echo "%end" >> $PKGS
    ;;
    riscv64)
        echo "%packages" >> $PKGS
        echo "extlinux-bootloader" >> $PKGS
        echo "%end" >> $PKGS
    ;;
esac
%end

%include /tmp/arch-packages.ks

%packages
# Use https://pagure.io/livesys-scripts to configure the system
livesys-scripts

@core

#@hardware-support
# Intel wireless firmware assumed never of use for disk images
#-iwl*
#-ipw*
#-usb_modeswitch

uboot-tools
-grubby
grubby-deprecated

initscripts
chkconfig
glibc-all-langpacks

dracut-network
tar

@anaconda-tools
aajohan-comfortaa-fonts
anaconda
anaconda-install-env-deps
anaconda-live
dracut-config-generic
dracut-live
glibc-all-langpacks
kernel
# Make sure that DNF doesn't pull in debug kernel to satisfy kmod() requires
#kernel-modules
#kernel-modules-extra
#-@dial-up
-@input-methods
-@standard
-gfs2-utils
-gnome-boxes
%end
```
10. 将 kickstart 文件丢进 mock 环境 `mock -r fedora-38-riscv-lmc.cfg --copyin fedora-riscv-anaconda-livecd.ks /root/`
11. 将之前生成的 boot.iso 丢进 mock 环境 `mock -r fedora-38-riscv-lmc.cfg --copyin lorax_output/images/boot.iso /root/`
12. 因为 mock 内的 pyanaconda 也没适配 RISC-V，所以还得进 mock 的文件目录下手动打 patch \
具体而言是先 `mock -r fedora-38-riscv-lmc.cfg --install vim` 而后 `mock -r fedora-38-riscv-lmc.cfg --shell` 进 shell 之后再 `vi /lib64/python3.11/site-packages/pyanaconda/modules/storage/platform.py`，主要内容同上文的 patch，随后保存
13. 执行 `mock -r fedora-38-riscv-lmc --enable-network --chroot -- livemedia-creator --resultdir=/results/try-1 --logfile=/results/logs/try-1/try-1.log --make-fsimage --ks /root/fedora-riscv-anaconda-installer.ks --iso /root/boot.iso --no-virt --nomacboot` 生成启动分区，注意格式为 raw partition
14. 此时生成的启动磁盘还不能使用，需要制作一个磁盘映像，把分区塞进去，还要做个 boot 分区，来符合目前的启动方案
15. 先用 fallocate 生成一个空的文件 `fallocate -l 4G extlinux.img`
16. 使用 parted 对刚刚生成的文件进行分区，使其变成一个磁盘映像文件，需要做的操作包括 \
  - 创建一个MSDOS分区表
  - 创建一个200M大小的ext4分区，增加 boot flag
  - 用剩下的空间再创建一个ext4分区用来系统
17. 使用 losetup 挂载映像到 loop 设备 `sudo losetup -P loop3 extlinux.img`
18. 把两个分区分别挂载到 /mnt 下  `sudo mount -o loop /dev/loop3p1 /mnt/3` `sudo mount -o loop /dev/loop3p2 /mnt/4`
19. 将刚刚生成的启动分区文件挂载 `sudo mount -o loop lmc-disk-z1b0vmlv.img /mnt/5`
20. 用 rsync 把系统从启动分区复制到刚刚创建的空分区 `sudo rsync -aX /mnt/5/* /mnt/4/`
21. 将启动系统的 boot 文件夹复制到刚刚生成的启动磁盘的第一个boot分区 `sudo cp -R /mnt/5/boot/* /mnt/3/`
22. 在 boot 分区下创建 extlinux/extlinux.conf，内容如下，需要修改vmlinuz文件名和系统分区的UUID，可以通过`blkid`等查看
```
# extlinux.conf generated by appliance-creator
# ui menu.c32
# menu autoboot Welcome to fedora-disk-minimal-f37-20230401-171433.n.0. Automatic boot in # second{,s}. Press a key for options.
menu title fedora-disk-minimal-f37-20230401-171433.n.0 Boot Options.
# menu hidden
timeout 20
# totaltimeout 600

label fedora-disk-minimal-f37-20230401-171433.n.0 (6.2.16-300.0.riscv64.fc38.riscv64)
        kernel /vmlinuz-6.2.16-300.0.riscv64.fc38.riscv64
        append ro root=UUID=a5076c04-7d68-4cc8-ba33-7708452953ba rhgb console=tty0 console=ttyS0,115200 earlycon=sbi rootwait selinux=0 LANG=en_US.UTF-8 verbose rd.break=pre-trigger
#       fdtdir /dtb-6.0.10-300.0.riscv64.fc37.riscv64/
        initrd /initramfs-6.2.16-300.0.riscv64.fc38.riscv64.img
```
23. unmount上述的两个分区 `umount /mnt/3 && umount /mnt/4`
24. 使用 losetup 卸载 loop 设备 `sudo losetup -d /dev/loop3`
25. 思路是，挂载安装盘为第一磁盘，再挂载一个空盘为第二磁盘，启动安装磁盘后将 Fedora 安装到第二磁盘
26. 使用 fallocate 快速创建新盘 `fallocate -l 30G blank.raw`
27. 使用 qemu 启动
```
#!/bin/bash
qemu-system-riscv64    -nographic \
                       -machine virt \
                       -smp 8 \
                       -m 16G \
                       -bios fw_dynamic.bin \
                       -kernel u-boot.bin \
                       -object rng-random,filename=/dev/urandom,id=rng0 \
                       -device virtio-rng-device,rng=rng0 \
                       -device virtio-blk-device,drive=hd0 \
                       -device virtio-blk-device,drive=hd1 \
                       -drive file=extlinux.img,format=raw,id=hd0 \
                       -drive file=blank.raw,format=raw,id=hd1 \
                       -device virtio-net-device,netdev=usernet \
                       -netdev user,id=usernet,hostfwd=tcp:127.0.0.1:10099-:22,hostfwd=tcp:127.0.0.1:11451-:5900
```
28. 进入系统，用户名为 root，密码为 feedme 或 rootme
29. 启动 anaconda，发现无论是在 text mode 还是使用 ks 文件进行无人值守安装都存在找不到盘的问题，后来尝试在安装盘的磁盘后追加一个新的分区，并在ks文件内指定anaconda安装到此分区，但anaconda输出需要unmount当前安装盘才可继续安装，且目前观察下来只要该问题排除后续就理应顺畅，仍在调查问题中
