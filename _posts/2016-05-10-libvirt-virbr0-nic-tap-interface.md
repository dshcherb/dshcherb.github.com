---
layout: post
title: "Linux bridging with tun/tap and loopback (dummy) devices, libvirt's use case with virbr0-nic tap interface"
date: 2016-05-10
description: An insight into how a virtual bridge created by libvirt operates
---

I have not been aware about the internals of libvirt's approach of building virtual networks using bridges up until the point when I started looking at the sources for the libvirt, Linux kernel and OpenVPN. So here is what I found.

If a bridge is created via `brctl addbr <br_name>` and there are no interfaces connected to it is going to have a [randomly generated MAC address](https://github.com/torvalds/linux/blob/9256d5a308c95a50c6e85d682492ae1f86a70f9b/net/bridge/br_device.c#L369) but as soon as there is a new interface connected, it is going to [pick](https://github.com/torvalds/linux/blob/c05c2ec96bb8b7310da1055c7b9d786a3ec6dc0c/net/bridge/br_if.c#L568) the interface's MAC address for its bridge id (if there are multiple interfaces connected, the one with the lowest MAC is picked). This is why libvirt has some code ([src/network/bridge_driver.c](https://github.com/libvirt/libvirt/blob/ca33d1747251e61a281f0535b6b2fd556be1f121/src/network/bridge_driver.c#L2492-L2513)) to create a tap device ([drivers/net/tun.c](https://github.com/torvalds/linux/blob/master/drivers/net/tun.c)) to have the bridge id fixed (though there is no guarantee that this logic is not going to break if a smaller value for a MAC address of an interface is selected).

{% highlight C %}
{% raw %}

/* To set a mac for the bridge, we need to define a dummy tap
 * device, set its mac, then attach it to the bridge. As long
 * as its mac address is lower than any other interface that
 * gets attached, the bridge will always maintain this mac
 * address.
 */

// ...

/* Keep tun fd open and interface up to allow for IPv6 DAD to happen */

...
/* DAD has finished, dnsmasq is now bound to the
 * bridge's IPv6 address, so we can set the dummy tun down.
 */

if (tapfd >= 0) {
if (virNetDevSetOnline(macTapIfName, false) < 0)
    goto err4;
VIR_FORCE_CLOSE(tapfd);
}

{% endraw %}
{% endhighlight C %}

The above shows that libvirt creates a dummy tap device which it keeps in the 'Up' state (the file descriptor is kept open, see tun/tap carrier discussion below) until the Duplicate Address Detection (DAD) for IPv6 finishes. Then the tap interface is set to the 'Down' state.

In general, there is a difference between the Administrative State and the Operating State of a network device ([RFC 2863, section 3.1.13](https://tools.ietf.org/html/rfc2863#section-3.1.13), [kernel.org: Documentation/networking/operstates.txt](https://www.kernel.org/doc/Documentation/networking/operstates.txt)): `ip link set <dev_name> up` can be issued to bring an interface up administratively but it may not come up operationally for various reasons (e.g. a cable is not plugged in). Both states can be checked by looking at a netlink message represented by `struct ifinfomsg`: `ifinfomsg::if_flags & IFF_UP` for the administrative state and `ifinfomsg::if_flags & IFF_RUNNING` for the operating state (the message is received either by polling or subscribing to related messages).

{% highlight C %}
{% raw %}

struct ifinfomsg {
	unsigned char	ifi_family;
	unsigned char	__ifi_pad;
	unsigned short	ifi_type;		/* ARPHRD_* */
	int		ifi_index;		/* Link index	*/
	unsigned	ifi_flags;		/* IFF_* flags	*/
	unsigned	ifi_change;		/* IFF_* change mask */
};

{% endraw %}
{% endhighlight C %}

In the case of a tun/tap device there must be a user space process with an open handle to the device to keep it operationally up, otherwise `NO-CARRIER` state is going to be shown. Note that when a process is killed, all of its open file descriptors are closed, therefore: either a process closes a file destriptor itself, exits or is being killed, which results in a loss of a carrier for a tun/tap device. If it was the only device connected to a bridge, the bridge itself is going to lose carrier as well. This can be seen by looking at the tun/tap source code: there are three functions `tun_chr_open`, `tun_chr_ioctl` and `tun_chr_close` which do the required operations:

* `tun_chr_open` allocates the required data structures;
* `tun_chr_ioctl` handles user space commands issued via ioctl interface. Depending on a command `tun_set_iff` might be called followed by `netif_carrier_on` for a specific device depending on the branching in the code;
* `tun_chr_close` calls `tun_detach` which in turn calls `__tun_detach` and eventually `netif_carrier_off` (as the name suggests, this leads to a NO-CARRIER state).

These functions are mentioned in the code as follows:

{% highlight C %}
{% raw %}

static const struct file_operations tun_fops = {
	.owner	= THIS_MODULE,
	.llseek = no_llseek,
	.read_iter  = tun_chr_read_iter,
	.write_iter = tun_chr_write_iter,
	.poll	= tun_chr_poll,
	.unlocked_ioctl	= tun_chr_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = tun_chr_compat_ioctl,
#endif
	.open	= tun_chr_open,
	.release = tun_chr_close,
	.fasync = tun_chr_fasync,
#ifdef CONFIG_PROC_FS
	.show_fdinfo = tun_chr_show_fdinfo,
#endif
};

{% endraw %}
{% endhighlight C %}

As a result, if you want a 'completely virtual' bridge which has a carrier all the time without depending on any user space processes, you might need something better than the tun/tap device but for the purposes of libvirt the developers decided to use tun/tap. The reason is probably that there is a loopback interface for local communication, otherwise, it only makes sense to keep a bridge up if there are devices that are able to transmit. With QEMU/KVM the VMs are processes with tap interfaces connected to bridges therefore for host-only case your virbr[x] interface is going to have no carrier if all tap interfaces are down.

An alternative to tun/tap devices are loopback devices but there is only a single loopback device (actually, per a [network namespace](https://lwn.net/Articles/580893/)). A workaround for loopback devices in this case are dummy devices ([drivers/net/dummy.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/net/dummy.c)) - they are always up (both operationally and administratively) unless set administratively down.
