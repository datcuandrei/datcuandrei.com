---
layout: post
title:  "Testing the new NFS LOCALIO auxiliary RPC protocol"
description: "Useful? Or just a cool concept?"
date: 2024-10-06 01:07:00 +0300
pin: false
---

# Table of Contents
{: data-toc-skip='' .mt-4 .mb-0 }

- [Introduction](#introduction)
- [Benchmarks from commiter](#benchmarks-from-commiter)
- [Testing LOCALIO](#testing-localio)
	- [Compiling-the-kernel](#compiling-the-kernel)
	- [Running a NFS Server container](#running-a-nfs-server-container)
	- [Giving LOCALIO a spin](#giving-localio-a-spin)
- [Theorizing VM and hypervisor interaction](#theorizing-vm-and-hypervisor-interaction)
	- [Why does LOCALIO (currently) only work in containerized environments?](#why-does-localio-currently-only-work-in-containerized-environments)
- [Conclusion](#conclusion)

## Introduction 
{: data-toc-skip='' .mt-4 .mb-0 }
During the last merge window for the 6.12 Linux kernel, a couple commits have been made regarding a new protocol for the NFS filesystem with documentation (see [this one](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=92945bd81ca4) and [FAQ](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f7128262b152)). This is very important, as this new protocol is supposed to allow the NFS client and server to reliably handshake to detect if they are running on the same host.

## Benchmarks from comitter
{: data-toc-skip='' .mt-4 .mb-0 }
For this section, I will quote the committer himself:

```Documentation/filesystems/nfs/localio.rst
+fio for 20 secs with directio, qd of 8, 16 libaio threads:
+- With LOCALIO:
+  4K read:    IOPS=979k,  BW=3825MiB/s (4011MB/s)(74.7GiB/20002msec)
+  4K write:   IOPS=165k,  BW=646MiB/s  (678MB/s)(12.6GiB/20002msec)
+  128K read:  IOPS=402k,  BW=49.1GiB/s (52.7GB/s)(982GiB/20002msec)
+  128K write: IOPS=11.5k, BW=1433MiB/s (1503MB/s)(28.0GiB/20004msec)
+
+- Without LOCALIO:
+  4K read:    IOPS=79.2k, BW=309MiB/s  (324MB/s)(6188MiB/20003msec)
+  4K write:   IOPS=59.8k, BW=234MiB/s  (245MB/s)(4671MiB/20002msec)
+  128K read:  IOPS=33.9k, BW=4234MiB/s (4440MB/s)(82.7GiB/20004msec)
+  128K write: IOPS=11.5k, BW=1434MiB/s (1504MB/s)(28.0GiB/20011msec)
+
+fio for 20 secs with directio, qd of 8, 1 libaio thread:
+- With LOCALIO:
+  4K read:    IOPS=230k,  BW=898MiB/s  (941MB/s)(17.5GiB/20001msec)
+  4K write:   IOPS=22.6k, BW=88.3MiB/s (92.6MB/s)(1766MiB/20001msec)
+  128K read:  IOPS=38.8k, BW=4855MiB/s (5091MB/s)(94.8GiB/20001msec)
+  128K write: IOPS=11.4k, BW=1428MiB/s (1497MB/s)(27.9GiB/20001msec)
+
+- Without LOCALIO:
+  4K read:    IOPS=77.1k, BW=301MiB/s  (316MB/s)(6022MiB/20001msec)
+  4K write:   IOPS=32.8k, BW=128MiB/s  (135MB/s)(2566MiB/20001msec)
+  128K read:  IOPS=24.4k, BW=3050MiB/s (3198MB/s)(59.6GiB/20001msec)
+  128K write: IOPS=11.4k, BW=1430MiB/s (1500MB/s)(27.9GiB/20001msec)
```

The read/writes have an enormous boost by bypassing networking protocols (XDR & RPC).

Of course, these benchmarks seem all good and dandy, but this must be put to a real word scenario.

## Testing LOCALIO
{: data-toc-skip='' .mt-4 .mb-0 }
For the purposes of this test, I will have two similar QEMU VMs with the specs of your average 5$ VPS:

- 6 vCPUs from a host Xeon E5-2690 v3
- 4096MB RAM
- 20GiB `virtio` Disk (host using off-brand SATA SSD)
- Debian GNU/Linux 12 (bookworm) with GNOME 42.

> The only difference between these two VMs is going to be the kernel itself: `debian-vm1` is running the latest stable kernel i.e. `6.11.2`, while `debian-vm2` will run the current mainline (at the time of writing this post) kernel, i.e. `6.12-rc1` with `CONFIG_NFS_LOCALIO` (only on `debian-vm2`) , `CONFIG_NFS_FS` and `CONFIG_NFSD` enabled in `.config`. The reason I will not be using the kernel that Debian comes with is because `CONFIG_NFSD` is disabled by default, and we will need `nfsd` to be injected in the container.
{: .prompt-info }

> Also, we will use OCI containers, specifically, we will use Docker as the means to configure them.The NFS Server will be installed in the container, while the NFS Client will be the host. 
{: .prompt-info }

### Compiling the kernel
{: data-toc-skip='' .mt-4 .mb-0 }

- For the kernel itself, I have used the 6.1.0 kernel config, but updated:
```bash
cp /boot/config-$(uname -r) .config && make olddefconfig
```

- Also, in Debian and it's derivatives, a certificate is required to run signed modules. Since we are using a custom kernel, we have to disable `MODULE_SIG` to run unsigned ones:
```bash
./scripts/config --file .config --disable MODULE_SIG
```

- Now we get to the LOCALIO config. Simply enable `CONFIG_NFS_LOCALIO`.
```bash
./scripts/config --file .config --enable CONFIG_NFS_LOCALIO
```

> You can also enable `NFS client and server support for LOCALIO auxiliary protocol` in `menuconfig` to achieve the same result.
{: .prompt-info }

![Enable LOCALIO in menuconfig](/assets/images/enable_nfs_localio_menuconfig.png)
_Screenshot of `menuconfig`._

> Make sure that both CONFIG_NFS_FS and CONFIG_NFSD are enabled too!
{: .prompt-warning }

- You can also change the `LOCALVERSION` if you want. For the purpose of this experiment, I've changed it to `localiotest` on `debian-vm2`, and on the stable VM, I went for `debianvm1`:
```bash
./scripts/config --file .config --set-str LOCALVERSION "-localiotest"
``` 

- Now we are ready to compile with:
```bash
make -j$(nproc) 2>&1 | tee log
```

And off it goes! Compiling the kernel means patience, so grab yourself a cup of tee, or a beer if that's more your thing and enjoy the output.
<br>
![Compiling kernel](/assets/images/compiling_rc_kernel.png)
_Compiling kernel..._

Once the kernel is done compiling, we only have a couple of things left to do.

- We need to install the kernel and it's modules and headers:
```bash
sudo make modules_install -j$(nproc) && sudo make headers_install && sudo make install
```

- And generate the new GRUB config, with the updated kernel for the boot entry:
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

- Finally, reboot.
![Output after compilation](/assets/images/output_after_compilation.png)
_Side-by-side comparison of NFS kernel config._


### Running a NFS Server container
{: data-toc-skip='' .mt-4 .mb-0 }
To save time (and annoyances), we will be using an already created image, specifically [ghcr.io/obeone/nfs-server](https://github.com/obeone/docker-nfs-server). It is a simple, lightweight Alpine Linux image that supports NFS v3,v4 with the option to map `nfsd` inside the container for the server to run.

The following command was used to start the container:
```bash
sudo docker run -dit --privileged -v /nfstest -e NFS_EXPORT_0='/nfstest *(rw,no_subtree_check,no_root_squash)' -p 2049:2049 --name nfs-server ghcr.io/obeone/nfs-server
```

### Giving LOCALIO a spin.
{: data-toc-skip='' .mt-4 .mb-0 }
After mounting the NFS servers in `/media/nfstest` and testing whether they work or not, we can measure the performance of LOCALIO using `fio`:
```bash
fio --name=DEBVM[no] --size=1G --filename=DEBVM[no]FILE --ioengine=libaio --direct=1 --bs=4k --iodepth=64 --readwrite=randrw --rwmixread=75 --gtod_reduce=1 --randrepeat=1
```
> `no` represents the VM number. Doing random read/write operations with a 4k block size to really notice the difference. 
{: .prompt-info }

And here are the results:
![FIO results](/assets/images/fio_results.png)
_Running `fio-3.33` on both VMs._

With a whopping `13.9k IOPS read / 4630 IOPS write` on the LOCALIO-enabled kernel, compared to the less impressive `4413 IOPS read / 1474 IOPS write` on the stable kernel, you can see that LOCALIO makes an impactful difference, drastically reducing the operation from 44.522 seconds to just 14.177 seconds.

But this is all in a containerized environment. How about testing on, let's say, a hypervisor and a VM?

## Theorizing VM and hypervisor interaction
{: data-toc-skip='' .mt-4 .mb-0 }
Since Docker is basically a virtualized environment relying on the host for the network, wouldn't it be possible to achieve the same results between a VM and hypervisor?

Well, let's look at the container setup we've made earlier. The network that the Docker environment was using is the default one, with the port shared between the container and VM. On the QEMU VM, we have a virtual network for the host too by using the default NAT named `virbr0`. So in theory, it should work, right?

Let's put it to the test. We are now going to create a NFS server on the second VM with the LOCALIO enabled kernel and mount it on my host.

For that, I just need to install `nfs-kernel-server` and create an export. I'll use the same export as in the container for this test. I already have `nfs-common` on my host, so it's just a matter of mounting it.

Now let's test it to see if that worked as theorized:
![FIO host](/assets/images/fio_host.png)

Hmm... that doesn't seem to be working. Maybe since there is no way for the VM to tell that they are running on the same host due to the virtualized NAT. The difference is that the Docker virtual network is a bridged one, so maybe by creating a bridged network between the host and VM will actually make it work?

Unfortunately, no, even with a bridged network connection, we are achieving the exact same results. That is because it defeats the purpose of this protocol.

### Why does LOCALIO (currently) only work in containerized environments?
{: data-toc-skip='' .mt-4 .mb-0 }
If you've played around with OCI containers for a bit, you most likely have noticed that they work totally different from a fully fledged VM. That is because, compared to a VM, they lack many components of a real OS, like standalone kernel modules. Earlier, I've mentioned that the kernel that Debian comes with doesn't have `CONFIG_NFSD` enabled, and that is a crucial component in this whole experiment, because `nfsd` is a kernel module. You probably already can tell where I'm going with this.

If you've tried before to create a NFS server in a container, you have likely been hit by tons of errors. That's because you cannot use `nfsd` in a container, _unless_ you mount it from your host. The handshake that is done to reliably determine if the client and server run on the same host is done through it. [here is how the handshake works in a technical manner](https://www.kernel.org/doc/html/latest/filesystems/nfs/localio.html#nfs-common-and-client-server-handshake).

I am not a software engineer by any means, but here is the way I see the handshake work: when establishing the connection, the LOCALIO module comes in; the NFS client then generates an UUID that will be checked by the NFS server for security reasons. Once verified that it is a unique and secure connection, it is passed through a module named `nfs_common`, which maintains a list of these UUIDs that point to the server's memory. This is basically how `UUID_IS_LOCAL` works, if it turns out as true, then the client has direct access to the server's memory, bypassing RPC.

So this is why the network itself doesn't matter, it is `nfsd` that matters. If it is the same on both the server and client (which works between a container and it's host due to the container's lack of kernel modules), then LOCALIO works. This is why VMs and hypervisors won't work. 

## Conclusion
{: data-toc-skip='' .mt-4 .mb-0 }
There are certain use cases for this protocol, especially when you have a container that needs to run an IO job local to the host, you'll gain an enormous boost in performance. But if you are looking for more, or you were hoping for VM interactions, then I'm sorry to disappoint you, it's not there *yet*.

<script src="https://giscus.app/client.js"
        data-repo="datcuandrei/datcuandrei.com"
        data-repo-id="R_kgDOMYCjvw"
        data-category="General"
        data-category-id="DIC_kwDOMYCjv84Ci52E"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="noborder_dark"
        data-lang="en"
        crossorigin="anonymous"
        async>
</script>
