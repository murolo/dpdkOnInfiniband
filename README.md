
# Outline
* [Obstacle and next](#Obstacle_and_next)
* [My Suspiction](#My_suspicion)




# Task 1 #
## 1.   认对应的交换机是否支持dpdk ##
交换机型号：MSX6710-FS2F2  
交换机介绍网页：[http://www.colfaxdirect.com/store/pc/viewprd.asp?idproduct=2784](http://www.colfaxdirect.com/store/pc/viewprd.asp?idproduct=2784)  
dpdk支持硬件网页：[https://core.dpdk.org/supported/](https://core.dpdk.org/supported/)
	
~~交换机是基于Mellanox公司的SwitchX-2  
dpdk的supported hardware里对于Mellanox表示支持的是ConnectX-3*，ConnectX-4*,ConnectX-5*,Bluefield  
现在的工作是：搞清楚SwitchX和ConnectX是不是一个类型的东西，还是说Switch是类似cpu，Connect是类似网卡；前者代表交换机不能支持dpdk，后者代表还是有可能支持的。  
ConnectX 属于 Ethernet Adapter IC 以太网适配器IC~~
	
[How can you determine if your Ethernet adapters support DPDK and SRIOV?](https://software.intel.com/en-us/forums/networking/topic/542581)

	
~~我的结论：应该**不支持**。Switch和Connect都是芯片，dpdk表示支持Connect，但是没表明支持Switch~~
	
## 2. tcp/ip -> dpdk's tcp/ip ##
dpdk-ans: [https://github.com/ansyun/dpdk-ans](https://github.com/ansyun/dpdk-ans)  
dpdk-ans user guide: [https://github.com/ansyun/dpdk-ans/blob/master/doc/guides/ans_user_guide.pdf](https://github.com/ansyun/dpdk-ans/blob/master/doc/guides/ans_user_guide.pdf)  
	
	
	
## dpdk安装
```bash
	# for normal ethernet
	tar xvf dpdk-17.11.4.tar
	cd dpdk-stable-17.11.4
	export RTE_SDK=/home/togo/work/dpdk-stable-17.11.4
	export RTE_TARGET=x86_64-native-linuxapp-gcc
	# 可以不修改网上说是历史遗留错误
	#vim lib/librte_eal/linuxapp/igb_uio/igb_uio.c
	#	if (pci_intx_mask_supported(udev->pdev)){
	#		dev_dbg(&udev->pdev->dev, "using INTX");
	#	->
	#	// if (pci_intx_mask_supported(udev-pdev)){
	#	{
	#		dev_dbg(&udev->pdev->dev, "using INTX");
	./usertools/dpdk-setup.sh
		[14]x86_64-native-linuxapp-gcc
		[34]Exit Script
	# numa.h not found: yum install numactl-devel numactl-lbis

	# seems not nesessary, only need to compile x86_64-native-linuxapp-gcc
	# make config T=x86_64-native-linuxapp-gcc DESTDIR=/home/togo/dpdk
	# make install T=x86_64-native-linuxapp-gcc DESTDIR=/home/togo/dpdk

	./usertools/dpdk-setup.sh
		[17]Insert IGB UIO module
		[20]Setup hugepage mappings for NUMA systems
			# enter 2048 twice
			# Creating /mnt/huge and mounting as hugetlbfs
		[23]Binid Ethernet/Crypto device to IGB UIO module
			0000:02:07.0
			0000:02:06.0
			#OK
			#*** is active. Not modifying -> ifconfig xxx down

	#test
	cd examples/helloworld
	make
	./build/helloworld
		{
		hello from core 1
		hello from core 2
		hello from core 3
		hello from core 0
		}
	reboot now
```
		
## dpdk-ans安装 (excluded later)
```bash
	export RTE_ANS=/root/work/dpdk-ans
	git clone git@https://github.com/ansyun/dpdk-ans
	cd dpdk-ans
	./install_deps.sh
		#Generate librte_ans.a/librte_anssock.a/librte_anscli.a for brodwell successfully
```
		
## mtcp安装(主要看重现部分)
```bash
bash setup_mtcp_dpdk_env.sh $RTE_SDK
	
Press [14] to compile x86_64-native-linuxapp-gcc version

Press [17] to install the driver

Press [21] to setup 2048 2MB hugepages

Press [23] to register the Ethernet ports

Press [34] to quit the tool
	
```
### Bug solution
1. unknown field 'ndo_change_mtu' specified in intializer  
https://bugs.dpdk.org/show_bug.cgi?id=49 说在17.11和18.02版本的dpdk上遇到这个问题，换成18.05解决  
https://github.com/F-Stack/f-stack/issues/238 说升级到18.05版本解决了这个问题。开发者说Edit dpdk/x86_64-native-linuxapp-gcc/.config, change the value of KNI related section from 'y' to 'n'. 
但是我升级到18.05dpdk还是有这个问题
	
	centos linux release 7.5.1804(core)
	https://access.redhat.com/solutions/3441281 说升级了7.5之后就报错了。 a custom device driver no longer compiles on rhel-7.5
	https://software.intel.com/en-us/node/777485 说7.4没问题，7.5（当时新发布）有问题
	https://bugzilla.redhat.com/show_bug.cgi?id=1572923 [This happens on RHEL 7.5  but not on RHEL 7.4 for some reason]
	
	https://www.centos.org/forums/viewtopic.php?f=47&t=67556 : can be fixed by **changeing .ndo_change_mtu   to   .ndo_change_mtu_rh74**
	
	
2. unknown field ‘ndo_setup_tc’ specified in initializer  
**to \*_rh74**
	
3. unknown field ‘ndo_set_vf_vlan_rh74’ specified in initializer  
**to \*_rh73**
	
4. hugepages 相关错误  
一般都是no free hugepages.  
**reboot now  ->  bash setup_mtcp_dpdk_env.sh $RTE_SDK**
	
5. No Erhernet port detected!  
**把网卡从dpdk compatibale 卸下来**  
**sudo ifconfig dpdk0 x.x.x.x netmask 255.255.255.0 up**

			
			
			
## Reproduce
```bash
	# dpdk ( normal ethernet )
	wget http://fast.dpdk.org/rel/dpdk-18.02.2.tar.xz
	tar xvf dpdk-18.02.2
	cd dpdk-stable-18.02.2
	export RTE_SDK=/home/togo/work/dpdk-stable-18.02.2
	export RTE_TARGET=x86_64-native-linuxapp-gcc
	yum install numactl-devel gmp-devel
	#http://doc.dpdk.org/guides/nics/mlx4.html
	./usertools/dpdk-setup.sh
		[14]x86_64-native-linuxapp-gcc
		[34]Exit Script
	#make config T=$RTE_TARGET DESTDIR=/home/togo/dpdk
	#make install T=$RTE_TARGET DESTDIR=/home/togo/dpdk
	
	
	# test
	sudo ifconfig ensf11f0 down
	./usertools/dpdk-setup.sh
		[17]Insert IGB UIO module
		[20]Setup hugepage mappings for non-NUMA systems
			64
		[23]Bind Ethernet/Crypto device to IGB UIO module
	cd examples/helloworld
	make
	sudo ./build/helloworld
	
	#sudo reboot now
	
	#mtcp
	git clone https:/github.com/zzqq2199/mtcp.git
	cd mtcp
	export RTE_SDK=/home/togo/work/dpdk-stable-18.02.2
	export RTE_TARGET=x86_64-native-linuxapp-gcc
	vim setup_mtcp_dpdk_env.sh
		cd dpdk-iface-kmod
		->
		cd /home/togo/work/mtcp/dpdk-iface-kmod
	vim dpdk-iface-kmod/dpdk-iface.h
		ndo_change_mtu -> ndo_change_mtu_rh74
		ndo_setup_tc -> ndo_setup_tc_rh74
		ndo_set_vf_vlan -> ndo_set_vf_vlan_rh73
	bash setup_mtcp_dpdk_env.sh $RTE_SDK
		[14] x86_64-native-linuxapp-gcc
		[17] install the driver
		[21] setup 2048 2MB hugepages
		[23] register the Ethernet ports
		[34] quit the tool
	autoreconf -ivf
	./configure --with-dpdk-lib=$RTE_SDK/$RTE_TARGET
	make
	
	#test
	sudo ifconfig ensf11f0 down
	sudo ifconfig dpdk0 192.168.1.43 netmask 255.255.255.0 up
	#for server:
		mkdir /home/togo/www
		echo hahaha >> /home/togo/www.txt
		sudo ./epserver -p /home/togo/www -f epserver.conf -N 8
	#for client:
		sudo ./epwget 192.168.1.42/a.txt 100000 -N 8 -c 80 -f epwget.conf
```
	
	
	
	 
## 测试C/S
	老板的意思应该是：
	原来的C/S模型
		client  <--- TCP/IP --->  server
	在不修改client与server的情况下实现：
		client  <--- DPDK TCP/IP ---> server
		
	确认mtcp的环境下，application是否需要对mtcp进行特定的开发
	之前是用的ens11f0 是ok的
	现在回到尝试用ib0




## 搞万兆网。。。
~~千兆和万兆都ping不同的情况下，把ens4f0先down再up试试~~

~~Q：
gpu2上  sudo ifconfig ens4f0 down 之后为啥还是能ping通192.168.1.63~~

~~打通还是1Gbps
问题：把42（1Gbps）的ens11f0卸下，把62（10Gbps）的ens4f卸下，然后以dpdk0上到62，跑出来的结果是1Gbps
把42（1Gbps）的卸下，然后以dpdk0上到82，也能进行实验。~~

~~下一步：
不把1Gbps的卸下，进行试验可以吗？~~


 
万兆网口测试了一下：

收发都是4.x Gbps，加起来9.x Gbps 程序运行有几个参数：

命令行的有：

-N 用到的CPU个数

-c 最大并发数

-f 里的配置有：

	num_mem_ch = 4	#number of memory channels per processer socket
	sndbuf = rcvbuf = 8192

对程序运行指定的几个参数分别调了一下，测出的速率没变或者有所下降。

万兆网口测试了一下：

收发都是4.x Gbps，加起来9.x Gbps

对程序运行指定的几个参数分别调了一下，测出的速率没变或者有所下降。

速率与**msg size**相关


## mtcp on infiniband
email from Asim(one author of mtcp)

	Thanks for showing interest in the mTCP project. I see that you are using Mellanox NIC for your experiments. Please note that you should enable CONFIG_RTE_LIBRTE_MLX4_PMD macro (as you have). The reason why dpdk build process is showing compile-time errors is probably because the GitHub version of dpdk stable branch is not compatible (Mellanox driver developers are sometimes slow with their updates). I suggest that should download the stable version of dpdk as suggested by the Mellanox customer service website (http://www.mellanox.com/page/products_dyn?product_family=209&mtag=pmd_for_dpdk). 

	Please do not bind the Mellanox NIC to igb_uio driver. The default mlx4 driver handles DPDK' PMD driver on its own. Running the ./dpdk_iface_main program is also not necessary (since that program should only be run if you are using Intel-specific NICs). Please start step 2 of the README file once you have correctly configured the Mellanox PMD driver NIC.

	Please let me know if you have any other (follow-up) questions.


1. download the stable version of dpdk(2.0.0)  # not compitable with linux version  
compile error: too few arguments to function 'ndo_dflt_bridge_getlink' # function changes with linux kernel upgrading  
solution:
https://www.cnblogs.com/hugetong/p/6377604.html

```bash
			#avoid to compile kni
			/root/src/thirdparty/dpdk/dpdk-stable-16.07.2/x86_64-native-linuxapp-gcc/.config 
			374,375c374,375
			< CONFIG_RTE_LIBRTE_KNI=n
			< CONFIG_RTE_KNI_KMOD=n
				---
			> CONFIG_RTE_LIBRTE_KNI=y
			> CONFIG_RTE_KNI_KMOD=y
			
			#enable MLX4
			PMD_MLX4=y
```
			

2.	do not bind the mellanox nic to igb_uio driver.  
error:	if so, i can't register the port  
solution:  the default mlx4 driver handles dpdk' pmd driver on its own.
	
3. 安装与dpdk2版本配套的mtcp

```bash
export RTE_SDK=/home/togo/work/dpdk-2.0.0
export RTE_TARGET=x86_64-native-linuxapp-gcc
cd $RTE_SDK
vim x86_64-native-linuxapp-gcc/.config # avoid to compile kni
	< CONFIG_RTE_LIBRTE_KNI=n
	< ** MLX4 = y
	---
	> CONFIG_RTE_LIBRTE_KNI=y	
	> ** MLX4 = n
#make config T=x86_64-native-linuxapp-gcc DESTDIR=/home/togo/dpdk2
#make install T=x86_64-native-linuxapp-gcc DESTDIR=/home/togo/dpdk2

# Create soft links for include/ and lib/ directories inside empty dpdk/ directory
cd dpdk/
ln -s <path_to_dpdk_2_0_0_directory>/x86_64-native-linuxapp-gcc/lib lib
ln -s <path_to_dpdk_2_0_0_directory>/x86_64-native-linuxapp-gcc/include include

#Setup mtcp library:
./configure --with-dpdk-lib=$<path_to_mtcp_release_v3>/dpdk
# And not dpdk-2.0.0!
#e.g. ./configure --with-dpdk-lib=`echo $PWD`/dpdk
cd mtcp/src
make
#  - check libmtcp.a in mtcp/lib
#  - check header files in mtcp/include

# make in util/:
make

# make in apps/example:
make
#  - check example binary files

# Check the configurations
#  - epserver.conf for server-side configuration
#  - epwget.conf for client-side configuration
#  - you may write your own configuration file for your application
#  - please see README.config for more details
#    -- for the latest version, dyanmic ARP learning is *DISABLED*

# Run the applications!
	



#重启后需要的命令：
export RTE_SDK=/home/togo/work/dpdk-2.0.0
cd /home/togo/work/mtcp2
bash setup_mtcp_dpdk_env $RTE_SDK
	[] setup 2048 2MB hugepages
	[] quit the tool
```
	

### Bug Solutions
1. infiniband/mlx4dv.h: no such file or directory  
Cause: the dpdk version is too high. use older dpdk(under 16.11 is ok)  
~~Solution: Downloaded the driver in the mlnx official website http://www.mellanox.com/page/products_dyn?product_family=27 and installed it to get mlx4dv.h~~
		
2. fatal error: infiniband/verbs.h: No such file or directory  
Cause: no such lib  
Solution: **yum install libibverbs-devel**
	
	
# Obstacle_and_next
## dpdk on infiniband
在两个相邻版本上有不同的错误  

	dpdk2.2.0
		tools/dpdk_nic_band.py: no ConnectX-3
		helloworld is ok
		./build/l2fwd -l 0-3 -n 4 -- -q 8 -p ffff
			Initializing port 0... PMD:librte_pmd_mlx4: 0x951960: TX queues number update: 0 -> 1
			PMD: librte_pmd_mlx4: 0x951960: RX queues number update: 0 -> 1
			PMD: librte_pmd_mlx4: 0x951960: QP state to IBV_QPS_INIT failed: Invalid arguments
			EAL: Error - exiting with code: 1
				Cause: rte_eth_rx_queue_setup:err=-22, port=0
		
	dpdk16.07.2
		./build/helloworld -c f -n 4
			EAL: Error - exiting with code: 1 
			Cause: Requested device 0000:82:00.0 cannot be used

# My_suspicion
我怀疑我们的交换机不支持dpdk
1. 一开始认为我们的网卡支持dpdk的原因  
[DPDK support hardware](http://core.dpdk.org/supported/)显示支持Mellanox的mlx4(ConnectX-3, ConnectX-3 Pro)系列，我们的交换机属于Mellanox ConnectX-3系列

2. 怀疑不支持的理由：  
	* [DPDK Document](http://doc.dpdk.org/guides/nics/mlx4.html#supported-nics)列出了支持MLX4的单个列表，与之对比的有MLX5的详细列表  
	```
	23.5. Supported NICs (MLX4)
	Mellanox(R) ConnectX(R)-3 Pro 40G MCX354A-FCC_Ax (2*40G)
	(Our switch type is MSX6710-FS2F2)
	```
	```
	26.7. Supported NICs (MLX5)
	Mellanox(R) ConnectX(R)-4 10G MCX4111A-XCAT (1x10G)
	Mellanox(R) ConnectX(R)-4 10G MCX4121A-XCAT (2x10G)
	Mellanox(R) ConnectX(R)-4 25G MCX4111A-ACAT (1x25G)
	Mellanox(R) ConnectX(R)-4 25G MCX4121A-ACAT (2x25G)
	Mellanox(R) ConnectX(R)-4 40G MCX4131A-BCAT (1x40G)
	Mellanox(R) ConnectX(R)-4 40G MCX413A-BCAT (1x40G)
	Mellanox(R) ConnectX(R)-4 40G MCX415A-BCAT (1x40G)
	Mellanox(R) ConnectX(R)-4 50G MCX4131A-GCAT (1x50G)
	Mellanox(R) ConnectX(R)-4 50G MCX413A-GCAT (1x50G)
	Mellanox(R) ConnectX(R)-4 50G MCX414A-BCAT (2x50G)
	Mellanox(R) ConnectX(R)-4 50G MCX415A-GCAT (2x50G)
	Mellanox(R) ConnectX(R)-4 50G MCX416A-BCAT (2x50G)
	Mellanox(R) ConnectX(R)-4 50G MCX416A-GCAT (2x50G)
	Mellanox(R) ConnectX(R)-4 50G MCX415A-CCAT (1x100G)
	Mellanox(R) ConnectX(R)-4 100G MCX416A-CCAT (2x100G)
	Mellanox(R) ConnectX(R)-4 Lx 10G MCX4121A-XCAT (2x10G)
	Mellanox(R) ConnectX(R)-4 Lx 25G MCX4121A-ACAT (2x25G)
	Mellanox(R) ConnectX(R)-5 100G MCX556A-ECAT (2x100G)
	Mellanox(R) ConnectX(R)-5 Ex EN 100G MCX516A-CDAT (2x100G)
	```
	* Mellanox官网可以很容易找到针对ConnectX-3 Pro以上的[Mellanox DPDK Quick Start Guide](http://www.mellanox.com/related-docs/prod_software/MLNX_DPDK_Quick_Start_Guide_v16.11_1.5.pdf)，与此同时找不到针对ConnectX-3的DPDK Start Guide


