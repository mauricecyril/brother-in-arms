# Brother printer drivers for Raspberry Pi and other ARM devices

Why? The Brother provided Linux drivers have parts that are compiled for i386 and have no source code. Although there is an [open source Brother Printer driver](https://github.com/pdewacht/brlaser), it is not optimized for cheap printers with their small cache.
It fails part way into a print on [large images or busy pages](https://github.com/pdewacht/brlaser/issues/95).

So, how do we port the closed-source driver to another architecture? There are several methods:

1. Run i386 code on ARM [through an emulator](https://wiki.alphaframe.net/doku.php?id=raspberry_pi:brotherh1110). It works, but testing on RPi 4, the emulation layer overhead made printing terribly slow.
1. Reverse engineer, recreate and recompile the code for ARM. This has only been done for [rawtobr3](https://github.com/k1-801/rawtobr3) as far as I know. There is also an original source available for `brcupsconfig4`, but other necessary part are missing.
1. Scavange and collect Brother's own ARM code into the full ARM install package. This is the approach taken here.

## Quick start

Run the `oh_brother.zsh` script with the model of the printer that you have, like this `./oh_brother.zsh HL-4570CDW`

If this script fails, you need to do the steps manually. Read below:

## Background

Many Brother printers are designed for LPR and lack support for common page description languages (PDF, PCL, etc). They require a CUPS wrapper to translate the print options to their native protocol.

There are at least four Brother provided pre-compiled x86 32-bit binaries. Specifically:

* `/usr/local/Brother/Printer/HL2270DW/cupswrapper/brcupsconfig4` - wrapper that calls on `brprintconflsr3` to set up `/usr/share/Brother/inf` on every print to pass the current options.
* `/usr/local/Brother/Printer/HL2270DW/lpd/rawtobr3` - converter for the job to a raster format. Seems to be model-agnostic. Called by `/opt/brother/Printers/BrGenPrintML2/lpd/lpdfilter`
* `/usr/local/Brother/Printer/HL2270DW/inf/brprintconflsr3` - translates job options to Brother format. Seems to be backward compatible. Called by `/opt/brother/Printers/BrGenPrintML2/cupswrapper/lpdwrapper` from `brgenprintml2pdrv-4.0.0-1.i386.deb` which is itself in the `opt/brother/Printers/BrGenPrintML2/cupswrapper/brother-BrGenPrintML2-cups-en.ppd`
* `/usr/local/Brother/Printer/HL2270DW/inf/braddprinter` - adds alphanumeric printer model name to `/usr/share/Brother/inf/brPrintList` upon installation. This can be safely ignored.

Below is an example of how to create ARM packages for Brother HL2270DW, but it should work for other printers, both on ARM64 and ARM32 RPis. Substitute the proper packages for your printer. To find these packages:

* Head out to the Brother website and grab the native i386 Linux deb pacakges. Or
* Use `tools/web_brother.sh` from [a Brother copyist](https://github.com/illwieckz/debian_copyist_brother) or `curl http://www.brother.com/pub/bsc/linux/infs/<uppercasedmodelhere>` and then grab these packages from http://www.brother.com/pub/bsc/linux/packages/. Or
* Run the [linux-brprinter-installer](https://download.brother.com/welcome/dlf006893/linux-brprinter-installer-2.2.2-1.gz) with `set -e` injected to see all the URLs that its probing before it confirming the installation.

Oh, yeah and next time just buy a printer that has [Driverless printing](https://wiki.debian.org/DriverlessPrinting#The_Concept_of_Driverless_Printing) or [IPP Everywhere](https://wiki.debian.org/IPPEverywhere)

## Creating the ARM packages for MCF440CN

We'll be grabbing 2 native arm32 executables and compile one other. All the steps can be done on RPi or on x86 Doing it on x86 will require cross-compilation of `brcupsconfig` for armhf.

### Repackage LPR drivers

Get the original i386 LPR driver, unpack the files and the control information

	wget https://download.brother.com/welcome/dlf006128/mfc440cnlpr-1.0.1-1.i386.deb
	dpkg -x mfc440cnlpr-1.0.1-1.i386.deb mfc440cnlpr-1.0.1-1.armhf.extracted
	dpkg-deb -e mfc440cnlpr-1.0.1-1.i386.deb mfc440cnlpr-1.0.1-1.armhf.extracted/DEBIAN
	sed -i 's/Architecture: i386/Architecture: armhf/' mfc440cnlpr-1.0.1-1.armhf.extracted/DEBIAN/control
	echo true > mfc440cnlpr-1.0.1-1.armhf.extracted/usr/local/Brother/Printer/mfc440cn/inf/braddprinter

Grab the Brother ARM drivers from a generic armhf archive they provide and copy the ARM code into the unpacked folders. Note that HL2270 does not use `brprintconflsr3`.

	wget http://download.brother.com/welcome/dlf103361/brgenprintml2pdrv-4.0.0-1.armhf.deb
	dpkg -x brgenprintml2pdrv-4.0.0-1.armhf.deb brgenprintml2pdrv-4.0.0-1.armhf.extracted
	# cp brgenprintml2pdrv-4.0.0-1.armhf.extracted/opt/brother/Printers/BrGenPrintML2/lpd/armv7l/brprintconflsr3 mfc440cnlpr-1.0.1-1.armhf.extracted/usr/local/Brother/Printer/mfc440cn/lpd
	cp brgenprintml2pdrv-4.0.0-1.armhf.extracted/opt/brother/Printers/BrGenPrintML2/lpd/armv7l/rawtobr3 mfc440cnlpr-1.0.1-1.armhf.extracted/usr/local/Brother/Printer/mfc440cn/lpd

Now repackage it

	cd mfc440cnlpr-1.0.1-1.armhf.extracted
	find . -type f ! -regex '.*.hg.*' ! -regex '.*?debian-binary.*' ! -regex '.*?DEBIAN.*' -printf '%P ' | xargs md5sum > DEBIAN/md5sums
	cd ..
	chmod 755 mfc440cnlpr-1.0.1-1.armhf.extracted/DEBIAN/p* mfc440cnlpr-1.0.1-1.armhf.extracted/usr/local/Brother/Printer/mfc440cn/inf/* mfc440cnlpr-1.0.1-1.armhf.extracted/usr/local/Brother/Printer/mfc440cn/lpd/*
	dpkg-deb -b mfc440cnlpr-1.0.1-1.armhf.extracted mfc440cnlpr-1.0.1-1.armhf.deb

### Repackage CUPS wrapper

Grab and extract the `brcupsconfig4` sources

	wget https://download.brother.com/welcome/dlf006733/brhl2270dwcups_src-2.0.4-2.tar.gz
	tar zxvf brhl2270dwcups_src-2.0.4-2.tar.gz
	cd brhl2270dwcups_src-2.0.4-2

Compile `brcupsconfig4`

	gcc brcupsconfig3/brcupsconfig.c -o brcupsconfig4


 or get 
 	wget https://download.brother.com/welcome/dlf006675/ink3_GPL_src_101-1.tar.gz
  	tar zxvf ink3_GPL_src_101-1.tar.gz
   	cd ink3_GPL_src_101-1

	gcc cupswrappermfc440cn_src/brcupsconfig/brcupsconfig.c -o brcupsconfig


If you are running these steps on the arm64 platform do cross complilation by first getting `arm-linux-gnueabihf-gcc-9` via `sudo apt install gcc-12-arm-linux-gnueabihf` and then running `arm-linux-gnueabihf-gcc-12 brcupsconfig3/brcupsconfig.c -o brcupsconfig`

	sudo apt-get install libc6-dev-i386-amd64-cross libc6-dev-i386-cross build-essential gcc-multilib*

	binutils-arm-linux-gnueabihf

	arm-linux-gnueabihf-gcc-12 cupswrappermfc440cn_src/brcupsconfig/brcupsconfig.c -o brcupsconfig
 
Grab the original i386 CUPS wrapper and unpack it

	wget https://download.brother.com/welcome/dlf006130/mfc440cncupswrapper-1.0.1-1.i386.deb
	dpkg -x mfc440cncupswrapper-1.0.1-1.i386.deb mfc440cncupswrapper-1.0.1-1.armhf.extracted
	dpkg-deb -e mfc440cncupswrapper-1.0.1-1.i386.deb mfc440cncupswrapper-1.0.1-1.armhf.extracted/DEBIAN
	sed -i 's/Architecture: i386/Architecture: armhf/' mfc440cncupswrapper-1.0.1-1.armhf.extracted/DEBIAN/control

Copy the compiled code into the unpacked folder

cp ink3_GPL_src_101-1/cupswrappermfc440cn_src/brcupsconfig/brcupsconfig mfc440cncupswrapper-1.0.1-1.armhf.extracted/usr/local/Brother/Printer/mfc440cn/cupswrapper


Repack it

	cd mfc440cncupswrapper-1.0.1-1.armhf.extracted
	find . -type f ! -regex '.*.hg.*' ! -regex '.*?debian-binary.*' ! -regex '.*?DEBIAN.*' -printf '%P ' | xargs md5sum > DEBIAN/md5sums
	cd ..
	chmod 755 mfc440cncupswrapper-1.0.1-1.armhf.extracted/DEBIAN/p* mfc440cncupswrapper-1.0.1-1.armhf.extracted/usr/local/Brother/Printer/mfc440cn/cupswrapper/*
	dpkg-deb -b mfc440cncupswrapper-1.0.1-1.armhf.extracted mfc440cncupswrapper-1.0.1-1.armhf.deb

### Installing

> Note: if you are using RPi4 64 bit OS (`AARCH64` aka `ARM64`) you need to install `libc6:armhf`. No need to do it on `ARM32` aka `AARCH32` aka `armhf` aka `armv7l`

	sudo dpkg --add-architecture armhf
	sudo apt update
	sudo apt install libc6:armhf

Install prereqs and install the drivers

	sudo apt install psutils cups
	sudo dpkg -i mfc440cnlpr-1.0.1-1.armhf.deb mfc440cncupswrapper-1.0.1-1.armhf.deb

> You may need to install  other runtime dependencies such as a2ps, glibc-32bit, ghostscript

### How to get the PPD and the filter

In case you're missing the printer defeniton you can extract it from the cupswrapper for from a Windows driver distribution using the [official brother method](https://help.brother-usa.com/app/answers/detail/a_id/164936/~/how-to-create-a-brother-ppd-file-for-installation---linux)

Top extract extract the filter and the PPD file directly from the cupswrapper run this:

    sed -e '0,/cat <<!ENDOFWFILTER! >/d' -e '/^!ENDOFWFILTER!/,$d' './mfc440cncupswrapper-1.0.1-1' -e 's|\\||g' > ./brlpdwrapperMFC440CN
    sed -e '0,/cat <<ENDOFPPDFILE >/d'   -e '/^ENDOFPPDFILE/,$d'   './mfc440cncupswrapper-1.0.1-1'              > ./MFC440CN.ppd


### How to install on an x86 platform

Either use the driver installer, which is just a bash script that downloads the debs, or download the debs manually and install

	curl -o - https://download.brother.com/welcome/dlf006893/linux-brprinter-installer-2.2.2-1.gz | gunzip > linux-brprinter-installer-2.2.2-1
	echo -e "HL2270-DW\ny\nn" | sudo bash linux-brprinter-installer-2.2.2-1
	(HL2270-DW as "model name", then y to continue, "no" for "Will you specify the DeviceURI?" choose "No" for USB connection or "Yes" for network connection.

or manually download debs and configure them.

## References

* https://wiki.archlinux.org/index.php/Packaging_Brother_printer_drivers
* https://blog.serverdensity.com/how-to-create-a-debian-deb-package/
* https://wiki.debian.org/CUPSDriverlessPrinting
