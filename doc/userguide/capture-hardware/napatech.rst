

Napatech Suricata Installation Guide
=============================================================

Contents
--------

	* Introduction

	* Package Installation

	* Basic Configuration

	* Advanced Multithreaded Configuration

Introduction
------------

Napatech packet capture accelerator cards can greatly improve the performance of your Suricata deployment using these
hardware based features:

	* On board burst buffering (up to 12GB)

	* Zero-copy kernel bypass DMA (specific kernel versions or patches not required)

	* Non-blocking PCIe performance

	* Port merging

	* Symetrical load distribution to up 128 host buffers (RSS)

	* Precise timestamping

	* Accurate time synchronization
	
	
You can get the lastest version of the Napatech driver software here: https://www.napatech.com/downloads/


Package Installation
--------------------

*Note that make, gcc, and the kernel headers are required for installation*

*Root privileges are also required*

*These instructions are intended for Suricata 4.0.1 or later*


Napatech Package Installation
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

*Note that you will be prompted to install the Napatech libpcap library. Answer "yes" if you would like to
use the Napatech card to capture packets in WIreshark, tcpdump, or another pcap based application.
Libpcap is not needed for Suricata as native Napatech API support is included*

Red Hat Based Distros::

    $ yum install kernel-devel-$(uname -r) gcc make
	$ ./package_install_3gd.sh

Debian Based Distros::

    $ apt-get install linux-headers-$(uname .r) gcc make
	$ ./package_install_3gd.sh

To complete installation for all distros ntservice::

	$ /opt/napatech3/bin/ntstart.sh -m

Suricata Installation
---------------------

After downloading and extracting the Suricata tarball, you need to run configure to enable Napatech support and
prepare for compilation::

	$ ./configure --enable-napatech --with-napatech-includes=/opt/napatech3/include --with-napatech-libraries=/opt/napatech3/lib
	$ make
	$ make install-full

Now edit the suricata.yaml file to configure the maximum number of streams to use. If you plan on using the load distribution
(RSS) feature in the Napatech accelerator, then the list should contain the same number of streams as host buffers defined in
ntservice.ini::

	Napatech:
		# The Host Buffer Allowance for all streams
		# (-1 = OFF, 1 - 100 = percentage of the host buffer that can be held back)
		hba: -1

		# use_all_streams set to "yes" will query the Napatech service for all configured
		# streams and listen on all of them. When set to "no" the streams config array
		# will be used.
		use-all-streams: yes

		# The streams to listen on
		streams: [0 - 7]

Note: hba is useful only when a stream is shared with another application.  When hba is enabled packets will be dropped
(i.e. not delivered to suricata) when the host-buffer utilization reaches the high-water mark indicated by the hba value.
This insures that, should suricata get behind in it's packet processing, the other application will still receive all
of the packets.  If this is enabled without another application sharing the stream it will result in sub-optimal packet
buffering.


Basic Configuration
-------------------

For the basic installation we will setup the Napatech capture accelerator to merge all physical
ports into single stream that Suricata can read from. for this configuration, Suricata will
handle the packet distribution to multiple threads.

Here are the lines that need changing in /opt/napatech3/bin/ntservice.ini for best single buffer performance::

	TimeSyncReferencePriority = OSTime	# Timestamp clock synchronized to the OS
	HostBuffersRx = [1,16,0]		# [number of host buffers, Size(MB), NUMA node]

Stop and restart ntservice after making changes to ntservice::

	$ /opt/napatech3/bin/ntstop.sh -m
	$ /opt/napatech3/bin/ntstart.sh -m

Now we need to execute a few NTPL (Napatech Programming Language) commands to complete the setup. Create
a file will the following commands::

	Delete=All				# Delete any existing filters
	Setup[numaNode=0] = streamid==0		# Set stream ID 0 to NUMA 0
	Assign[priority=0; streamid=0]= all	# Assign all phisical ports to stream ID 0

Next execute those command using the ntpl tool::

	$ /opt/napatech3/bin/ntpl -f <my_ntpl_file>

Now you are ready to start suricata::

	$ suricata -c /usr/local/etc/suricata/suricata.yaml --napatech --runmode workers

Advanced Multithreaded Configuration
------------------------------------

Now let's do a more advanced configuration where we will use the load distribution (RSS - like) capability in the
accelerator. We will create 8 streams and setup the accelerator to distribute the load based on a 5 tuple hash.
Increasing buffer size will minimize packet loss only if your CPU cores are fully saturated. Setting the minimum
buffer size (16MB) will gave the best performance (minimize L3 cache hits) if your CPU cores are keeping up.

*Note that it is extremely important that the NUMA node the host buffers are define in is the same physical CPU
socket that the Napatech accelerator is plugged into*

First let's modify the ntservice.ini file to increase the number and size of the host buffers::

	HostBuffersRx = [8,256,0]		# [number of host buffers, Size (MB), NUMA node]

Stop and restart ntservice after making changes to ntservice::

	$ /opt/napatech3/bin/ntstop.sh -m
	$ /opt/napatech3/bin/ntstart.sh -m

Now let's assign the streams to host buffers and configure the load distribution. The load distribution will be
setup to support both tunneled and non-tunneled traffic. Create a file that contains the ntpl commands below::

	Delete=All				# Delete any existing filters
	Setup[numaNode=0] = streamid==0
	Setup[numaNode=0] = streamid==1
	Setup[numaNode=0] = streamid==2
	Setup[numaNode=0] = streamid==3
	Setup[numaNode=0] = streamid==4
	Setup[numaNode=0] = streamid==5
	Setup[numaNode=0] = streamid==6
	Setup[numaNode=0] = streamid==7
	HashMode[priority=4]=Hash5TupleSorted
	Assign[priority=0; streamid=(0..7)]= all

Next execute those command using the ntpl tool::

	$ /opt/napatech3/bin/ntpl -f <my_ntpl_file>

Now you are ready to start Suricata::

	$ suricata -c /usr/local/etc/suricata/suricata.yaml --napatech --runmode workers

Information Available in Suricata Logs
--------------------------------------

For each stream that is being processed the following counters will be output in stats.log:

-  nt<streamid>.pkts - The number of packets recieved by the stream.

-  nt<streamid>.bytes - The total bytes received by the stream.

-  nt<streamid>.drop - The number of packets that were dropped from this stream due to buffer overflow conditions.

If hba is enabled the following counter will also be provided:

-  nt<streamid>.hba_drop - the number of packets dropped because the host buffer allowance high-water mark was reached.

In addition to counters, host buffer utilization is tracked and logged.  This is also useful for
debugging.  Log messages are output for both Host and On-Board buffers when reach 25, 50, 75
percent of utilization.  Corresponding messages are output when utilization decreases.

Support
-------

Contact a support engineer at: ntsupport@napatech.com

