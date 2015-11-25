# Building and running RDK-V for Intel Galileo gen 2

## Prerequisites:
* RDK account
* Intel Galileo gen 2 board with FTDI cable or FTDI converter
* VMware Workstation / Player
* 2G or more microSD card, card reader or USB adapter
* 100G of available space on HDD

## Build:

### Get Yocto Build Appliance

Download and extract Yocto Build Appliance virtual machine from [https://www.yoctoproject.org/downloads/tools](https://www.yoctoproject.org/downloads/tools)

### Customize virtual machine configuration

1. Open VMware, add Yocto Build Appliance
2. Customize virtual machine settings: memory, # of processor cores. For more information see [https://www.yoctoproject.org/documentation/build-appliance-manual](https://www.yoctoproject.org/documentation/build-appliance-manual)
3. Select hard disk, expand it from 45G to 100G
4. Download ubuntu iso image, add CD/DVD drive and point to the image. Check connect at power on option
5. Start the machine, quickly click the machine window to place focus there and quickly press F2 to enter BIOS. Change boot order to boot from CD/DVD first
6. When asked, select Try Ubuntu without installing
7. When Ubuntu comes up, run gparted and resize the hard disk from 45G to the maximum size of ~100G.
8. Wait until it completes and restart the virtual machine.
9. Enter BIOS and move CD/DVD drive below Hard Drive.

### Start the machine

1. Turn the virtual machine on
2. Hob (bitbake UI) appears. Close it.
3. There are two console window running, you can switch between them with Alt-Tab. If you cannot, make sure Num lock and Caps lock are off
4. With whoami command verify that you're on builder's console and not on the root's
5. cd; mkdir bin; echo "PATH=~/bin:$PATH" >> .profile
6. /sbin/ifconfig

### SSH to the machine

1. from the host computer, make ssh connection to the virtual machine, and use from now on instead of VM console. It allows to use copy/paste (it does not seem easy to install VMware tools on Yocto Build Appliance) and configure scrollback. Login/password are builder/builder
2. set current time, for ex.  sudo date -s "18 Nov 2016 16:05:00"
3. rm -rf ~/poky
4. Python 2.7.3 which is installed in Yocto Build Appliance does not have a module named formatter. Get this module from Python 2.7.10 and copy to Python 2.7.3 (installing Python 2.7.10 brings other problems, so we just copy the file)
5. wget https://www.python.org/ftp/python/2.7.10/Python-2.7.10.tgz; tar xzf Python-2.7.10.tgz; sudo cp Python-2.7.10/Lib/formatter.py /usr/lib/python2.7/

### Download RDK

Instructions based on [https://rdkwiki.com/rdk/display/Emulator/RDK+Emulator+Users+Guide](https://rdkwiki.com/rdk/display/Emulator/RDK+Emulator+Users+Guide)

1. curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo; chmod a+x ~/bin/repo
2. Create a file ~/.netrc with content:

        machine code.rdkcentral.com
        login <rdk_gerrird_userid>
        password <rdk_gerrit_password>

3. git config --global user.email "you@example.com"; git config --global user.name "Your Name"
4. mkdir galileo-rdk && cd galileo-rdk
5. repo init -u https://code.rdkcentral.com/r/manifests -b 2.1 -m emulator.xml
6. repo sync

### Download Intel iotdk

Instructions based on [https://software.intel.com/en-us/blogs/2015/03/04/creating-a-yocto-image-for-the-intel-galileo-board-using-split-layers](https://software.intel.com/en-us/blogs/2015/03/04/creating-a-yocto-image-for-the-intel-galileo-board-using-split-layers)

1. git clone --branch dizzy git://git.yoctoproject.org/meta-intel-quark
2. git clone --branch dizzy git://git.yoctoproject.org/meta-intel-galileo
3. git clone git://git.yoctoproject.org/meta-intel-iot-devkit

### Create configuration

1. In meta-cmf/setup_environment comment out adding layers to bblayer.conf

        #cat >> conf/bblayers.conf <<EOF
        #BBLAYERS =+ "\${RDKROOT}/meta-cmf-bsp-emulator"
        #BBLAYERS =+ "\${RDKROOT}/meta-cmf"
        #EOF

2. In meta-rdk/setup_environment remove excluding intel machines from the list of available machines

         #_options=$(\ls -1 *{,/*}/conf/machine/*.conf 2>/dev/null | grep -v '\(^meta-linaro\|^meta-intel\|/qemumips\|/qemuppc\)')
         _options=$(\ls -1 *{,/*}/conf/machine/*.conf 2>/dev/null | grep -v '\(/qemumips\|/qemuppc\)')

3. In the same file, meta-rdk/setup_environment make machine configuration searched through all layers

        #_MACH_CONF=`find ${_PWD_PREV}/meta-rdk* -name ${MACHINE}.conf`
        _MACH_CONF=`find ${_PWD_PREV}/meta* -name ${MACHINE}.conf`

4. Change directory to galileo-rdk
5. source meta-cmf/setup-environment. When prompted, select quark.conf. This script has created build folder galileo-rdk/build-quark
6. append the following lines to build-quark/conf/bblayers.conf

        BBLAYERS =+ "${RDKROOT}/meta-rdk-bsp-emulator"
        BBLAYERS =+ "${RDKROOT}/meta-cmf-bsp-emulator"
        BBLAYERS =+ "${RDKROOT}/meta-cmf"

        BBLAYERS += "${RDKROOT}/meta-intel-quark"
        BBLAYERS += "${RDKROOT}/meta-intel-galileo"
        BBLAYERS += "${RDKROOT}/meta-intel-iot-devkit"

	Yes, using 2 different BSP layers is weird, but I faced problems with dependencies that I could only overcome by putting emulator BSP layers back to the build.

7. In build-quark/conf/local.conf change DISTRO from rdk to iot-devkit-multilibc

        #DISTRO ?= "rdk"
        DISTRO ?= "iot-devkit-multilibc"

### Modify recipes

1. Add this lines to meta-rdk/recipes-core/images/rdk-generic-hybrid-image.bb

        EXTRA_IMAGEDEPENDS_append_quark = " grub-conf "

        IMAGE_INSTALL += "alsa-lib alsa-utils"                                                                                            
        IMAGE_INSTALL += "wpa-supplicant"                                                                                                 
        IMAGE_INSTALL += "linux-firmware-iwlwifi-6000g2a-6"                                                                               
        IMAGE_INSTALL += "linux-firmware-iwlwifi-135-6"                                                                                   
        IMAGE_INSTALL += "bluez5"                                                                                                         
        IMAGE_INSTALL += "avahi avahi-autoipd"                                                                                            
        IMAGE_INSTALL += "connman connman-client connman-tests"                                                                           
        IMAGE_INSTALL += "tzdata"                                                                                                         
        IMAGE_INSTALL += "ca-certificates"                                                                                                
        IMAGE_INSTALL += "icu"                                                                                                            

2. Include rdk.conf instead of poky.conf in meta-intel-iot-devkit/conf/distro/iot-devkit.conf and add MACHINEOVERRIDES

        #require conf/distro/poky.conf
        require conf/distro/rdk.conf

        MACHINEOVERRIDES .= ":hybrid" 

3. add grub-efi to GPLv3 whitelist in meta-cmf/conf/distro/rdk.conf

        #WHITELIST_GPLv3 = "gdb rsync"                                                                                                    
        WHITELIST_GPLv3 = "gdb rsync grub-efi"                                                                                            

### Increase priorities of Intel layers

1. meta-intel-galileo/conf/layer.conf

        #BBFILE_PRIORITY_galileo = "9"
        BBFILE_PRIORITY_galileo = "109"

2. meta-intel-iot-devkit/conf/layer.conf

        #BBFILE_PRIORITY_intel-iot-devkit = "10"
        BBFILE_PRIORITY_intel-iot-devkit = "110"

3. meta-intel-quark/conf/layer.conf

        #BBFILE_PRIORITY_quark-bsp = "6"
        BBFILE_PRIORITY_quark-bsp = "106"

### Build image

1. bitbake rdk-generic-hybrid-image
2. To workaround error with fetching linux-fusion from directfb.org: cd ~/galileo-rdk/downloads; wget http://www.oipf.tv/opensource-mirror/linux-fusion-9.0.3.tar.gz; touch linux-fusion-9.0.3.tar.gz.done; cd -
3. To fix lighttpd fetching error modify meta-rdk/recipes-extended/lighttpd/lighttpd_1.5.bb:

        #SRC_URI = "git://github.com/lighttpd/lighttpd1.4.git;branch=lighttpd-1.5.x
        SRC_URI = "git://git.lighttpd.net/lighttpd/lighttpd-1.x;branch=lighttpd-1.5-dead \

4. Pay attention to remove the trailing back-slash in the first commented line, otherwise it will confuse bitbake
5. During the build some warnings appear, this is expected

### Prepare .direct image

1. cd ~/galileo-rdk; git clone --branch dizzy git://git.yoctoproject.org/poky poky
2. make changes to meta-intel-iot-devkit/scripts/wic_monkey

        #wic_script = os.path.abspath(os.path.dirname(os.path.abspath(sys.argv[0]))) + '/../../scripts/wic'
        wic_script = os.path.abspath(os.path.dirname(os.path.abspath(sys.argv[0]))) + '/../../poky/scripts/wic

        #monkey_scripts_path = monkey_curr_path + '/../../scripts'
        monkey_scripts_path = monkey_curr_path + '/../../poky/scripts'

        #(rootfs_dir, kernel_dir, bootimg_dir, native_sysroot) = \
        (rootfs_dir, kernel_dir, bootimg_dir, hdddir, staging_area_dir, native_sysroot) = \

3. cd ~/galileo-rdk/build-quark; export PATH=/usr/sbin/:$PATH; ../meta-intel-iot-devkit/scripts/wic_monkey create -e rdk-generic-hybrid-image ../meta-intel-iot-devkit/scripts/lib/image/canned-wks/iot-devkit.wks
4. The above command will create .direct image at /var/tmp/wic/build/ and it will be named like iot-devkit-201611181726-mmcblkp0.direct. Save it for later use, because /var/tmp may be removed during system restart: mkdir ~/image; cp /var/tmp/wic/build/*.direct ~/image/

### Write .direct image to microSD card

More information at [https://software.intel.com/en-us/programming-blank-sd-card-with-yocto-linux-image-linux](https://software.intel.com/en-us/programming-blank-sd-card-with-yocto-linux-image-linux)

1. Insert microSD card into USB adapter (or card reader), plug into the computer which is running virtual machine
2. Connect the device to the virtual machine using VMware menu
3. In virtual machine, run sudo dmesg to see which device you just added. In my case, this is /dev/sdb
4. cd ~/image; sudo dd if=iot-devkit-201611181726-mmcblkp0.direct of=/dev/sdb bs=3M conv=fsync
5. disconnect the device from the virtual machine, then safely eject it from the host OS

### Boot Galileo

1. Make serial connection to the Galileo board
2. Insert the microSD card into the Galileo board and reboot it
3. Allow to boot the default grub boot entry. It should automatically login to root account

        galileo login: root (automatic login)
        root@galileo:/# 

## Run RDK

### Copy sample media from the virtual machine

1. scp builder@10.0.0.105:/home/builder/galileo-rdk/build-quark/tmp/work/all-rdk-linux/linaro-samplemedia/1.0-r0/image/opt/www/BBBshort.ts /opt/www/
2. cd /opt/www; ln -s BBBshort.ts received_spts1.ts

### Create new partition on the SD card

1. fdisk /dev/mmcblk0
2. n, accept all defaults
3. w
4. reboot
5. mkfs.vfat /dev/mmcblk0p3
6. umount /opt/data
7. mount /dev/mmcblk0p3 /opt/data

### Record simulated broadcast program

1. export USE_GENERIC\_GSTREAMER\_TEE=TRUE; cd /usr/bin; rmfApp. Note, for starting rmfApp current folder must be /usr/bin
2. press Enter to get a command prompt
3. record -id 1234 -title recordTest1 ocap://0x125d Wait some minutes
4. While recording in still progress, press Enter again to get the prompt
5. type ls to see the list of recordings
6. Type quit to exit from rmfApp

### Stream from the device

In order to stream from the device, need to have Emulator built. Run RDK Emulator

1. On the board restart rmfStreamer: killall -9 rmfStreamer; cd /usr/bin; ./rmfStreamer

	on the Emulator:

2. export USE\_GENERIC\_GSTREAMER_TEE=TRUE; cd /usr/bin; rmfApp
3. launch -source hnsource -sink mediaplayersink http://10.0.0.100:8080/vldms/dvr?rec_id=1234
