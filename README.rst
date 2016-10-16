###################
YaST remote logging
###################
At the moment the most feasible option is to use linuxrc.
Depending on the time available it might make sense to consider other
options as well (see `OTHER`_).


The table of contents is branched in front of every relevant design
choice, please use it that was found.

.. contents:: Table of Contents
    :depth: 7

LINUXRC
~~~~~~~
Linuxrc has a Loghost flag (see `linuxrc docs`_) to handle remote
addresses to send logs to. Unfortunately this argument has not been
implemented in the main and is simply passed over to YaST.

.. _linuxrc docs: https://en.opensuse.org/SDB:Linuxrc

OBVIOUS SOLUTION
****************

The most obvious solution is implement Loghost() to do the following things:

- create a config file for a defined daemon (see `CONSIDERED DAEMONS`_)
- make sure that connection is available
- open firewall ports
- start the daemon


DESIGN
^^^^^^

CONFIG FILE CREATION
====================
problem: initrd is mounted as read only file system

possible solutions:

- ask the daemon to read an external file (TODO research)
- circumvent the limitations of the initrd
- other

MAKE SURE CONNECTION IS AVAILABLE
=================================
At the moment the connection is automatically setup.

TODO:
investigate corner cases where this doesn't happen and eventually make
sure that it happens anyway

FIREWALL PORT
=============
Shouldn't be necessary to fiddle with it because we're studying the
behaviour of the client, not the server (default settings might be enough)


TODO:
integration test on initrd ground

START THE DAEMON
================
Possible solutions to divert logging:

- develop ad hoc solution:
  This is an expensive option, and it doesn't seem to make sense at
  the moment because there's a variety of good candidates out there.

  The good is that we could call it Yet another System Logger (YaSL)

- use existing packages like rsyslog, syslog-ng or systemd logger (find how they work at `CONSIDERED DAEMONS`_)

TESTS (getting hands dirty)
^^^^^^^^^^^^^^^^^^^^^^^^^^^

PREPARE THE SYSTEM
==================
The virtualisation service used here is libvirt (install with zypper in -t patterns kvm_server)


.. code-block:: bash

  #get mksusecd
  zypper in mksusecd

  #get openSUSE
  wget http://download.suse.de/install/openSUSE-Leap-42.2-Beta3/openSUSE-Leap-42.2-NET-x86_64-Build0215-Media.iso

  #get linuxrc
  git clone git@github.com:openSUSE/linuxrc.git

  #make a custom image (see linuxrc README)
  sudo mksusecd --create ~/foo.iso --initrd /tmp/initrd/ ~/openSUSE-Leap-42.2-NET-x86_64-Build0215-Media.iso

  #run a virtual machine with it
  VIR_NAME=linuxrc_test
  virt-install --name $VIR_NAME --ram 1024 --vcpus 1 --cdrom foo.iso --disk /home/kvm_images/log_client.qcow2

  #delete it for a fresh start
  virsh destroy $VIR_NAME
  virsh undefine $VIR_NAME

.. note:: using Leap 42.2 because of systemd's version

CONSIDERED DAEMONS
==================

SYSTEMD LOGGER (what's shipped)
###############################

PROs:
it's included and the SLES user is expected to know about journalctl

CONs:
makes systemd a bit more difficult to replace

REQUIREMENTS:
systemd v >= 216, so Leap 42.1 is ruled out

TODO demo

EXTERNAL ALTERNATIVES
#####################
rsyslog, syslog-ng are not included in the initrd at the moment


RSYSLOG
+++++++
It's as easy as adding the following options and restarting the daemon.

Follow the steps:

- prepare the CLIENT

.. code-block:: bash

  #make sure the following lines are present in /etc/rsyslog.conf
  dhcp224:~ # cat /etc/rsyslog.conf

  ...

  #log system
  $ModLoad imuxsock
  #log kernel
  $ModLoad imklog


  #UDP FORWARD LOGS TO REMOTE SERVER (DHCP198)
  #this means that every log will be fowarded to the given IP address
  #on port 514. @ means that the protocol is UDP, to use TCP use @@

  *.*@10.100.51.198:514

- SERVER CONF

.. code-block:: bash

  dhcp198:~ # cat /etc/rsyslog.conf

  ...
  #log system
  $ModLoad imuxsock
  #log kernel
  $ModLoad imklog


  #load UPD reception, for TCP use imtcp
  $ModLoad imudp
  $UDPServerRun 514

  # This one is the template to generate the log filename dynamically,
  # depending on the client's IP address.
  $template FILENAME,"/var/log/%fromhost-ip%/syslog.log"

  #Log all messages to the dynamically formed file. Now each clients log
  (192.168.1.2, 192.168.1.3,etc...), will be under a separate directory
  which is formed by the template FILENAME.
  *.* ?FILENAME


- restart the daemons

.. code-block:: bash

  dhcp198:~ # rcsyslog restart

- make sure that the port you selected is not blocked by a firewall
  on the log server. In this test environment the firewall is simply off.

.. code-block:: bash

  dhcp198:~ # /sbin/SuSEfirewall2 status
  SuSEfirewall2: SuSEfirewall2 not active

- verify that logging works

.. code-block:: bash

  #the log files will be generated in /var/log/$IP_ADDRESS
  dhcp198:~ # ls /var/log/
  10.100.51.224     firewall              NetworkManager
  ...

  #to see the logs
  dhcp198:~ # tail -n 3 /var/log/10.100.51.224/syslog.log
  2016-10-16T13:02:32+02:00 dhcp224 systemd: pam_unix(systemd-user:session): session opened for user root by (uid=0)
  2016-10-16T13:15:01+02:00 dhcp224 cron[15122]: pam_unix(crond:session): session opened for user root by (uid=0)
  2016-10-16T13:15:01+02:00 dhcp224 CRON[15122]: pam_unix(crond:session): session closed for user root

CREDITS: http://www.thegeekstuff.com/2012/01/rsyslog-remote-logging/

HOW TO ADD PACKAGE TO INITRD
============================

Either include the package with mksusecd or download the content when
initrd is running.

This whole section hasn't been tested because rsyslog source package
couldn't be found in the obs and binary packages are not practical
because zypper is not installed in the initrd

MKSUSECD
########

.. code-block:: bash

  #get source
  oosc co openSUSE:Leap:42.1 rsyslog

  #with
  oosc: aliased to osc --apiurl https://api.opensuse.org

  #then include the files through mksusecd
  sudo mksusecd --create ~/foo.iso --initrd /tmp/initrd --initrd ../rsyslog ~/openSUSE-Leap-42.1-NET-x86_64.iso


DOWNLOAD at runtime using curl or wget
######################################
PROs:

- possible selection of backend from user

CONs:

- too many moving parts (at least) three points of failure (download
  server, client, connection between the two)

- possible exploits

- consistency is difficult to maintain

TODO
find other solutions

OTHER
~~~~~
To come

