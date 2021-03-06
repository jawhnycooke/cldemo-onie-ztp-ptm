ONIE, ZTP, and PTM Demo
=======================
This demo demonstrates how to configure an out of band management network to automatically install and configure Cumulus Linux using [Zero Touch Provisioning](https://docs.cumulusnetworks.com/display/DOCS/Zero+Touch+Provisioning+-+ZTP), and validate the cabling of the switches using [Prescriptive Topology Manager](https://docs.cumulusnetworks.com/display/DOCS/Prescriptive+Topology+Manager+-+PTM).

This demo is written for the [cldemo-vagrant](https://github.com/cumulusnetworks/cldemo-vagrant) reference topology.


Quickstart: Run the demo
------------------------
(This assumes you are running Ansible 1.9.4 and Vagrant 1.8.4 on your host.)

    git clone https://github.com/cumulusnetworks/cldemo-vagrant
    cd cldemo-vagrant
    vagrant up oob-mgmt-server oob-mgmt-switch leaf01 leaf02 spine01 spine02 server01 server02
    vagrant ssh oob-mgmt-server
    sudo su - cumulus
    git clone https://github.com/cumulusnetworks/cldemo-onie-ztp-ptm
    cd cldemo-onie-ztp-ptm
    sudo apt-get update
    sudo apt-get install -qy apache2 isc-dhcp-server bind9
    sudo cp ./etc/network/interfaces /etc/network
    sudo cp ./etc/dhcp/* /etc/dhcp
    sudo cp ./etc/bind/zones/* /etc/bind/zones
    sudo cp ./etc/bind/named.conf.options /etc/bind
    sudo cp ./var/www/* /var/www
    cp example_private_key ~/.ssh/id_rsa
    cp example_public_key ~/.ssh/id_rsa.pub
    chmod 0700 -R ~/.ssh/
    sudo cp example_public_key /var/www/id_rsa.pub
    sudo ifdown eth1
    sudo ifup eth1
    sudo service isc-dhcp-server restart
    sudo service bind9 restart
    ssh leaf01
    sudo su
    ifdown eth0; ifup eth0
    exit
    ptmctl


Topology
--------
This demo runs on a spine-leaf topology with two single-attached hosts. Each device's management interface is connected to an out-of-band management switch and bridged with the out-of-band management server that runs our webserver, DHCP server, and DNS server.

             +------------+       +------------+
             | spine01    |       | spine02    |
             |            |       |            |
             +------------+       +------------+
             swp1 |    swp2 \   / swp1    | swp2
                  |           X           |
            swp51 |   swp52 /   \ swp51   | swp52
             +------------+       +------------+
             | leaf01     |       | leaf02     |
             |            |       |            |
             +------------+       +------------+
             swp1 |                       | swp2
                  |                       |
             eth1 |                       | eth2
             +------------+       +------------+
             | server01   |       | server02   |
             |            |       |            |
             +------------+       +------------+


What is being configured?
-------------------------
During the quick start, we install and configure three major services on the out of band management server.

 * Apache Webserver: allows the management server to deliver ONIE install images and ZTP scripts to the nodes.
 * ISC DHCP Server: sends IP addresses, hostnames, and the ONIE/ZTP DHCP options to the nodes.
 * Bind9 Nameserver: provides DNS to the nodes (not strictly necessary, but very convenient)

If you are using the cldemo-vagrant topology, you will find that many of these services are already installed and running, so these steps exist to show you how to set things up if you were provisioning on hardware.

When a node running Cumulus Linux comes online and performs DHCPDISCOVER, it will download and run the zero touch provisioning script specified in /etc/dhcp/dhcpd.pools. This script performs four common functions:

 * Enables passwordless sudo, which makes driving the switch much easier.
 * Adds a public key to the cumulus user, enabling passwordless SSH as well.
 * Licenses the switch (only needed on physical hardware)
 * Restarts switchd (for licensing to take effect)
 * Sets all ports to ADMIN_UP (for PTM)
 * Downloads the topology.dot file from the oob-mgmt-server
 * Restarts PTM (to read the topology file)
