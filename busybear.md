## busybear
RISC-Vをqemuを利用してインストールするまでの作業履歴

### 参考
- [How to Run Linux on RISC-V with QEMU Emulator](https://www.cnx-software.com/2018/03/16/how-to-run-linux-on-risc-v-with-qemu-emulator/)
- [busybear-linux](https://github.com/michaeljclark/busybear-linux)

### インストール
* riscv-gnu-toolchain
```
$ mkdir scratch
$ cd scratch
$ sudo apt -y update
$ sudo apt -y upgrade
$ sudo apt -y install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev git pkg-config libglib2.0-dev libpixman-1-dev
$ git clone --recursive https://github.com/riscv/riscv-gnu-toolchain
$ cd riscv-gnu-toolchain
$ ./configure --prefix=/opt/riscv
$ sudo make linux
$ sudo make newlib
```
* busybear-linux
```
$ cs scratch
$ git clone https://github.com/michaeljclark/busybear-linux.git
$ cd busybear
$ export RISCV=/opt/riscv
$ export PATH=/opt/riscv/bin:$PATH
$ make
```
* qemu
```
$ cd scratch
$ git clone https://github.com/riscv/riscv-qemu.git
$ cd riscv-qemu
$ ./configure --target-list=riscv64-softmmu,riscv32-softmmu
$ make
$ sudo make install
```
* riscv-linux
```
$ cd scratch
$ git clone https://github.com/riscv/riscv-linux.git
$ cd riscv-linux
$ git checkout riscv-linux-4.14
$ cp ../busybear-linux/conf/linux.config .config
$ make ARCH=riscv olddeffconfig
$ make ARCH=riscv vmlinux
```

* ここでエラー発生
```
$ make ARCH=riscv vmlinux
  CHK     include/config/kernel.release
  CHK     include/generated/uapi/linux/version.h
  CHK     include/generated/utsrelease.h
  CHK     include/generated/bounds.h
  CHK     include/generated/timeconst.h
  CHK     include/generated/asm-offsets.h
  CALL    scripts/checksyscalls.sh
  CHK     scripts/mod/devicetable-offsets.h
  CHK     include/generated/compile.h
  VDSOLD  arch/riscv/kernel/vdso/vdso.so.dbg
/opt/riscv/lib/gcc/riscv64-unknown-elf/7.2.0/../../../../riscv64-unknown-elf/bin/ld: -shared not supported
collect2: error: ld returned 1 exit status
  OBJCOPY arch/riscv/kernel/vdso/vdso.so
riscv64-unknown-elf-objcopy: 'arch/riscv/kernel/vdso/vdso.so.dbg': No such file
arch/riscv/kernel/vdso/Makefile:43: recipe for target 'arch/riscv/kernel/vdso/vdso.so' failed
make[2]: *** [arch/riscv/kernel/vdso/vdso.so] Error 1
scripts/Makefile.build:573: recipe for target 'arch/riscv/kernel/vdso' failed
make[1]: *** [arch/riscv/kernel/vdso] Error 2
Makefile:1025: recipe for target 'arch/riscv/kernel' failed
make: *** [arch/riscv/kernel] Error 2
```
* 修正ポイント
```
$ vi arch/riscv/kernel/vdso/Makefile
 31 # SYSCFLAGS_vdso.so.dbg = -shared -s -Wl,-soname=linux-vdso.so.1 \
 32 SYSCFLAGS_vdso.so.dbg = -s -Wl,-soname=linux-vdso.so.1 \
```
* bbl
```
$ cd scratch
$ git clone https://github.com/riscv/riscv-pk.git
$ cd riscv-pk
$ mkdir build
$ cd build
$ ../configure --enable-logo --host=riscv64-unknown-elf --with-payload=../../riscv-linux/vmlinux
$ make
```
* Linux走行環境
```
$ cd scratch
$ mkdir linux
$ ln -s ../riscv-pk/build/bbl linux/bbl
$ ln -s ../busybear-linux/busybear.bin linux/busybear.bin
$ cd linux
$ cat ifup
#!/bin/sh
brctl addif virbr0 $1
ifconfig $1 up
$ cat ifdown
#!/bin/sh
ifconfig $1 down
brctl delif virbr0 $1
$ sudo qemu-system-riscv64 -nographic -machine virt \
  -kernel bbl -append "root=/dev/vda ro console=ttyS0" \
  -drive file=busybear.bin,format=raw,id=hd0 \
  -device virtio-blk-device,drive=hd0 \
  -netdev type=tap,script=./ifup,downscript=./ifdown,id=net0 \
  -device virtio-net-device,netdev=net0
  ```

### riscv用binutils
* binutils用configure オプション
  ```
  $ /home/kei/scratch/riscv-gnu-toolchain/riscv-binutils-gdb/configure \
  --target=riscv64-unknown-linux-gnu --prefix=/opt/riscv  \
  --with-sysroot=/opt/riscv/sysroot --disable-multilib \
  --disable-werror --disable-nls --with-expat=yes --enable-gdb
  ```
* riscv-pk用configure オプション
  ```
  $ ../configure --enable-logo --host=riscv64-unknown-elf \
  --with-payload=../../riscv-linux/vmlinux
  ```
* 試してみたもの
```
$ ../configure --target=riscv64-unknown-linux-gnu \
 --prefix=/opt2/riscv --disable-multilib --host=riscv64-unknown-elf
```
結局失敗したので削除

### ネットワーク環境

```
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 42:01:0a:8a:00:04 brd ff:ff:ff:ff:ff:ff
    inet 10.138.0.4/32 brd 10.138.0.4 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::4001:aff:fe8a:4/64 scope link 
       valid_lft forever preferred_lft forever
$ sudo apt -y install apt-file
$ sudo apt update
$ sudo apt -y install libvirt-daemon libvirt-daemon-system libvirt-clients
$ sudo systemctl status libvirtd
$ cat virbr0.xml 
<network>
	<name>default</name>
	<forward mode='nat'>
		<nat>
			<port start='1024' end='65535'/>
		</nat>
	</forward>
	<bridge name='virbr0' stp='on' delay='0'/>
	<mac address='52:54:00:2b:ce:be'/>
	<ip address='192.168.122.1' netmask='255.255.255.0'>
		<dhcp>
			<range start='192.168.122.11' end='192.168.122.254'/>
		</dhcp>
	</ip>
</network>
$ sudo virsh net-define virbr0.xml
$ sudo virsh net-start default
$ sudo virsh net-autostart default
$ sudo virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes
 $ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1460 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 42:01:0a:8a:00:04 brd ff:ff:ff:ff:ff:ff
    inet 10.138.0.4/32 brd 10.138.0.4 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::4001:aff:fe8a:4/64 scope link 
       valid_lft forever preferred_lft forever
3: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 52:54:00:2b:ce:be brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
4: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN group default qlen 1000
    link/ether 52:54:00:2b:ce:be brd ff:ff:ff:ff:ff:ff
$ cat mnt/etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
$ sudo mount -o loop ../busybear-linux/busybear.bin mnt

```
