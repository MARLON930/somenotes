## Abouot chardev
### Simple Character Utility for Loading Localities
scull是linux一个操作内存区域的字符设备驱动程序，这片内存区域就相当于一个设备。当我们查看linux系统的/dev/目录时可以看到:
```shell
brw-rw----  1 root disk      8,   0 Mar 25 16:21 sda
brw-rw----  1 root disk      8,   1 Mar 25 16:21 sda1
brw-rw----  1 root disk      8,   2 Mar 25 16:21 sda2
brw-rw----  1 root disk      8,   3 Mar 25 16:21 sda3
brw-rw----  1 root disk      8,   4 Mar 25 16:21 sda4
crw-rw-rw-  1 root tty       5,   0 Apr 22 11:16 tty
crw--w----  1 root tty       4,   0 Mar 25 16:21 tty0
crw--w----  1 root tty       4,   1 Mar 25 16:21 tty1
crw--w----  1 root tty       4,  10 Mar 25 16:21 tty10
crw--w----  1 root tty       4,  11 Mar 25 16:21 tty11
crw--w----  1 root tty       4,  12 Mar 25 16:21 tty12
crw--w----  1 root tty       4,  13 Mar 25 16:21 tty13
crw--w----  1 root tty       4,  14 Mar 25 16:21 tty14
crw--w----  1 root tty       4,  15 Mar 25 16:21 tty15
crw--w----  1 root tty       4,  16 Mar 25 16:21 tty16
crw--w----  1 root tty       4,  17 Mar 25 16:21 tty17
crw--w----  1 root tty       4,  18 Mar 25 16:21 tty18
crw--w----  1 root tty       4,  19 Mar 25 16:21 tty19
crw--w----  1 root tty       4,   2 Mar 25 16:21 tty2
crw--w----  1 root tty       4,  20 Mar 25 16:21 tty20
crw--w----  1 root tty       4,  21 Mar 25 16:21 tty21
crw--w----  1 root tty       4,  22 Mar 25 16:21 tty22
crw--w----  1 root tty       4,  23 Mar 25 16:21 tty23
crw--w----  1 root tty       4,  24 Mar 25 16:21 tty24
crw--w----  1 root tty       4,  25 Mar 25 16:21 tty25
crw--w----  1 root tty       4,  26 Mar 25 16:21 tty26
crw--w----  1 root tty       4,  27 Mar 25 16:21 tty27
crw--w----  1 root tty       4,  28 Mar 25 16:21 tty28
crw--w----  1 root tty       4,  29 Mar 25 16:21 tty29
crw--w----  1 root tty       4,   3 Mar 25 16:21 tty3
crw--w----  1 root tty       4,  30 Mar 25 16:21 tty30
crw--w----  1 root tty       4,  31 Mar 25 16:21 tty31
crw--w----  1 root tty       4,  32 Mar 25 16:21 tty32
crw--w----  1 root tty       4,  33 Mar 25 16:21 tty33
crw--w----  1 root tty       4,  34 Mar 25 16:21 tty34
```
字符设备第一列用c作为开头，而块设备用b作为开头，主设备号4,5,8次设备号就很多了。通常而言主设备号对应驱动程序，次设备号对应具体设备
linux cdev字符使用流程：

