# How to quickly rip apart a UEFI firmware

_2014/12/12 Cecill Etheredge // ijsf_

---

Imagine this. It's been a couple of months since you upgraded your trusty laptop with a good old SSD, the speed is fair but there's just something wrong. Hibernation is taking ages, but why? You rip your firmware apart and find out they've crippled your S-ATA controller..

Enter the [Panasonic Toughbook CF-19 MK3](http://www.panasonic.com/business/toughbook/fully-rugged-laptop-toughbook-19.asp), one of those built-as-a-tank laptops used by government agencies all around, and its [Intel ICH9M chipset](http://www.intel.com/content/www/us/en/io/io-controller-hub-9-datasheet.html), crippled but cool.

This hardened machine has an interesting thermal design: the entire case is [IP54](http://www.mpl.ch/info/IPratings.html) and [MIL-STD-810F](http://gcn.com/articles/2013/05/08/8-tests-behind-mil-std-ratings.aspx) rated for waterproofing and impact resistance, meaning that the entire system is only passively cooled. Chipsets and CPUs tend to get hot, and although a particular choice of low-power CPU is often easy, a chipset is better off throttled in cases of extremely tight thermal design.

As it appears, the S-ATA controller inside the CF-19's ICH9M chipset is normally capable of Gen 2 (3.0 Gb/s) performance, but only performs at Gen 1 (1.5 Gb/s). So how did they pull this off?

From an electric engineer's point-of-view, the solution can be quite easy: just throttle the entire system to reduce its thermal output. The chipset datasheet shows us how:

![pxsctl](http://bitlog.it/wp-content/uploads/2014/12/pxsctl.png)

Now, UEFI firmwares are built using elaborate development kits from companies such as American Megatrends or Phoenix Technologies, and contain vendor-specific driver code in something called the [Driver eXecution Environment](http://wiki.phoenix.com/wiki/index.php/DXE).

A quick dissection of the CF-19's UEFI firmware, which has been built using the American Megatrends Aptio development kit, shows us (among many other hidden gems) the following UEFI DXE driver:

![mmtool](http://bitlog.it/wp-content/uploads/2014/12/mmtool2.png)

This list, generated with AMI's Aptio MMTool, alone will give anyone a good insight into the low-level functionality of your own machine. The DXE drivers themselves are built in a PE-compatible executable format called TE, for which we will save [the reverse engineering](http://ho.ax/posts/2012/09/ida-pro-scripts-for-efi-reversing/) for another time.

For those wanting to get rid of the throttling, look no further. Just don't forget it was put there for a reason.
