Linux TC Demo
=======================
This demo demonstrates how to use traffic control (tc) to set both IP traffic limits and minimum guarantees.

This demo is written for the [cldemo-vagrant](https://github.com/cumulusnetworks/cldemo-vagrant) reference topology and applies the reference BGP unnumbered configuration from [cldemo-config-routing](https://github.com/cumulusnetworks/cldemo-config-routing).

On Server02, 

Quickstart: Run the demo
------------------------
(This assumes you are running Ansible 1.9.4 and Vagrant 1.8.4 on your host.)

    git clone https://github.com/cumulusnetworks/cldemo-vagrant
    cd cldemo-vagrant
    vagrant up oob-mgmt-server oob-mgmt-switch leaf01 leaf02 spine01 spine02 server01 server02
    vagrant ssh oob-mgmt-server
    sudo su - cumulus
    sudo apt-get install software-properties-common -y
    sudo apt-add-repository ppa:ansible/ansible -y
    sudo apt-get update
    sudo apt-get install ansible -qy
    git clone https://github.com/bunchc/linux-tc-demo
    cd linux-tc-demo
    ansible-playbook run-demo.yml
    tmuxinator start iperf

The above sequence will launch all of the virtual machines and network devices needed for the lab. Additionally, it will start a pre-staged tmux session on the ```oob-mgmt-server``` that initiates the following:

- Screen 1:
    + iperf server, host server02, port 8000
    + iperf server, host server02, port 8080
    + iperf client, host server01
        * 5 minute, bi-directional
        * reporting at 5 second intervals
        * port 8000
    + iperf client, host server01
        * 5 minute, bi-directional
        * reporting at 5 second intervals
        * port 8080
- Screen 2:
    + bmon
        * breaks down the HTB queues & bandwidth within them.

Topology
--------
This demo runs on a spine-leaf topology with two single-attached hosts. Each device's management interface is connected to an out-of-band management switch and bridged with the out-of-band management server from which we run Ansible.

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


Setting up the Infrastructure
-----------------------------
This lab configures HTB or hierarchical token bucket as the preferred packet scheduler on ```server02```. It then creates two traffic classes, one for port 8000 and one for port 8080. From there it creates three filters. The first two are using the u32 syntax of tc to filter on inbound and outbound ports. The third filter identifies traffic classified by IPTABLES.

### Exploring server02
Your tmux session on server01 will have a window configured to ssh in to server02 as the user cumulus. Select this window by pressing ctrl+b 

#### Show the tc queuing type for eth2

```
sudo su -
tc qdisc show dev eth2
```

You'll see that the you have htb configured:

```
qdisc htb 1: root refcnt 2 r2q 10 default 0 direct_packets_stat 21 direct_qlen 1000
```

#### Show the tc classifications for eth1

```
tc class show dev eth1
```

The above command lists each of the classes running on eth2. The first 8080 has a floor and ceiling of 500kbps. The second, 8000, at 1Mbps:

```
class htb 1:8080 root prio 0 rate 500Kbit ceil 500Kbit burst 1600b cburst 1600b
class htb 1:8000 root prio 0 rate 1Mbit ceil 1Mbit burst 1600b cburst 1600b
```

#### Show the tc filters on eth1

```
tc filter show dev eth2
```

Here you will see three filters. Two with the u32 classifier, one set to look for packets marked by IPTables:

```
filter parent 1: protocol ip pref 1 u32
filter parent 1: protocol ip pref 1 u32 fh 800: ht divisor 1
filter parent 1: protocol ip pref 1 u32 fh 800::800 order 2048 key ht 800 bkt 0 flowid 1:80
  match 00001f40/0000ffff at 20
filter parent 1: protocol ip pref 1 u32 fh 800::801 order 2049 key ht 800 bkt 0 flowid 1:80
  match 1f400000/ffff0000 at 20
filter parent 1: protocol ip pref 2 fw
filter parent 1: protocol ip pref 2 fw handle 0x6 classid 1:1
```


Scenarios
---------

Out of the box, the pre-staged tmux environment will show off the ceiling (threshold) and burst capabilities of our tc policies. However, you may wish to explore in further detail. 

> Note: You'll need to be able to navigate tmux to proceed from this point. That is a bit beyond our scope, but there is a great cheatsheet [here.](https://gist.github.com/MohamedAlaa/2961058)

### Lab 1 - Investigating the floor (traffic guarantee)

In this scenario, we will test the ability of tc & htb to guarantee minimum traffic performance. To do this we will need to exit our first set of tmux sessions (```ctrl + b :kill-session```).

To start lab 1, as the cumulus user on oob-mgmt-server, run:

```
tmuxinator start lab1
```

This command will start:
- Tmux
    + Window 1 - Traffic Generation
        * Pane 1 - iperf client to port 8000 (guaranteed 500Kbps)
        * Pane 2 - iperf client default port (the control)
        * Pane 3 - iperf server port 8000
        * Pane 4 - iperf server default port
    + Window 2 - Bmon
    + Window 3 - ssh to server02

The first pair of iperf sessions on port 8000 are to test the guaranteed performance of 500Kbps. This should hold steady, even given the second pair of iperf connections, that will send traffic as fast as possible.

### Lab 2 - Lab 1, but with IPTables

In this scenario, we will test the ability of tc & htb along with iptables to guarantee minimum traffic performance. To do this we will need to exit any existing tmux sessions (```ctrl + b :kill-session```).

To start lab 1, as the cumulus user on oob-mgmt-server, run:

```
tmuxinator start lab2
```

This command will start:
- Tmux
    + Window 1 - Traffic Generation
        * Pane 1 - iperf client to port 8080 (guaranteed 1Mbps)
        * Pane 2 - iperf client default port (the control)
        * Pane 3 - iperf server port 8080
        * Pane 4 - iperf server default port
    + Window 2 - Bmon
    + Window 3 - ssh to server02

The first pair of iperf sessions on port 8080 test the guaranteed performance of 1Mbps. This should hold steady, even given the second pair of iperf connections, that will send traffic as fast as possible.

### Lab 3 - Configuring a filter

If you have been following along, so far you have reviewed the tc classifications and filters that are part of the lab. You will have also tested them using iperf. In this lab, we will:

- Configure eth0 to use HTB
- Define a traffic classification
- Assign a traffic filter to the classification
- Test the filter

#### Getting Started

To get started run the following:

```
tmuxinator start lab3
```

What this does is open three tmux windows to server02 and server01 in our lab.

#### Configure HTB on eth0 on server02

From the first tmux window 'htb' (or ssh to server02), we need to run the following to change the traffic policy for eth0:

First, confirm the existing policy:
```
 sudo tc qdisc show dev eth0
qdisc pfifo_fast 0: root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
```

What we're interested in the prior output is ```pfifo_fast```, which is generally the default for most Linux distributions.

Now to change it, we run the following:

```
sudo tc qdisc add \
  dev eth0 \
  root \
  handle 1:0 \
  htb

sudo tc qdisc show dev eth0
qdisc htb 1: root refcnt 2 r2q 10 default 0 direct_packets_stat 72 direct_qlen 1000
```

As the ```add``` command produces no output on success, we re-ran the show command. You'll notice where pfifo changed to htb in the output.

#### Define a traffic classification

Next up, we need to add a classifier for the traffic. For us that means setting it's floor, ceiling, and burst rates. For this example, we are going to guarantee 1Kbps of traffic with a ceiling of 10Kbps, and a burst rate of 15Kbps:

```
sudo tc class show dev eth0

sudo tc class add \
  dev eth0 \
  parent 1:1 \
  classid 1:6 \
  htb \
  rate 1kbit \
  ceil 10kbit \
  burst 15kbit

sudo tc class show dev eth0
class htb 1:6 root prio 0 rate 1Kbit ceil 10Kbit burst 1920b cburst 1600b
```

You will notice a few things in the above code block. First, is that the show command will produce no output if there is no class defined. Next you will note how we specify line rates. Finally, you'll notice that we indeed classified traffic and assigned it the 1, 10, and 15Kbit rates respectively.

#### Add a filter to match traffic and assigne it a class

Our last step before testing, is to define what manner of traffic we are looking for, build a filter, and assign it to the class we just created. For our example we are going on the hunt for multicast traffic. We'll define the filter in two steps:

- Tell TC to look for packets with a netfilter marking of 42
- Use iptables to match multicast traffic, and mark it '42'

> Note, how we mark the packets is arbitrary. Pick something meaningful to you.

First we tell TC to be on the lookout for marked packets as follows:

```
sudo tc filter show dev eth0

sudo tc filter add \
  dev eth0 \
  proto ip \
  parent 1: \
  prio 1 \
  handle 42 \
  fw \
  classid 1:6

sudo tc filter show dev eth0
filter parent 1: protocol ip pref 1 fw
filter parent 1: protocol ip pref 1 fw handle 0x2a classid 1:6
```

You'll again notice some things in the output:

- The show command produces no output unless there is a filter
- Add filter uses 'handle' clause to define which packet marking to look for.
- 'fw' specifies that the fw (iptables) will be marking said packets.
- prio specifies priorities within the class queue

Next we tell iptables to mark our packets as follows:

```
sudo iptables -t mangle \
    -A INPUT \
    -i eth0 \
    -m pkttype \
    --pkt-type multicast \
    -j MARK \
    --set-mark 42

sudo iptables -t mangle \
    -A INPUT \
    -i eth0 \
    -m pkttype \
    --pkt-type multicast \
    -j RETURN

sudo iptables -L -vt mangle
Chain OUTPUT (policy ACCEPT 7 packets, 1016 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MARK       all  --  any    any     anywhere             anywhere             PKTTYPE = multicast MARK set 0x2a
    0     0 RETURN     all  --  any    any     anywhere             anywhere             PKTTYPE = multicast
```

The two iptables commands above specify we want to grab traffic of type multicast and mark it. We then want to return to normal packet processing. The final command shows that these rules were added to the mangle table.

#### Testing our new filter

To test this filter, we have set up an iperf server running on server02 that is listening to the multicast address 226.94.1.1. However, we still need to ensure traffic will only flow out eth0 from server01. We'll also want to send traffic. To do this, switch to the server01-iperf window (ctrl + b 1) or ssh to server01 and run the following:

```
sudo ifdown eth1
sudo iperf -c 226.94.1.1 -u -T 32 -t 300 -d -i 1 -B 192.168.0.32
```

## Advanced Labs

Now that you are comfortable with classifying traffic, we should apply these lessons to some more interesting Linux networking use cases. The next labs explore OVS, Linux Bridge, as well as identifying and limiting multicast traffic.

### Lab 4 - TC with OVS

In this lab you will be working with OVS or Open vSwitch. Specifically we will:

- install and configure OVS on server02
- Apply traffic shaping to OVS interfaces
- Wrap up by testing with iperf


### Lab 5 - TC with Linux Bridge

In this lab you will be working with Linux Bridge. Specifically we will:

- configure linux bridge interfaces
- Apply traffic shaping to the newly created interfaces
- Wrap up by testing with iperf

### Lab 6 - Multicast Traffic



Resources
---------
The following links explain TC and how to use it in some depth and provided the background reading for this lab.

- [http://tldp.org/HOWTO/Traffic-Control-HOWTO/elements.html](http://tldp.org/HOWTO/Traffic-Control-HOWTO/elements.html)
- [http://stackoverflow.com/questions/9513981/rtnetlink-answers-no-such-file-or-directory-error](http://stackoverflow.com/questions/9513981/rtnetlink-answers-no-such-file-or-directory-error)
- [http://docs.openstack.org/developer/neutron/devref/quality_of_service.html#linux-bridge](http://docs.openstack.org/developer/neutron/devref/quality_of_service.html#linux-bridge)
- [https://linux.die.net/man/8/tc-htb](https://linux.die.net/man/8/tc-htb)
- [http://www.tldp.org/HOWTO/html_single/Traffic-Control-HOWTO/#sc-wondershaper](http://www.tldp.org/HOWTO/html_single/Traffic-Control-HOWTO/#sc-wondershaper)
- [http://lartc.org/howto/lartc.cookbook.fullnat.intro.html](http://lartc.org/howto/lartc.cookbook.fullnat.intro.html)
- [https://www.linux.com/blog/tc-show-manipulate-traffic-control-settings](https://www.linux.com/blog/tc-show-manipulate-traffic-control-settings)
- [http://luxik.cdi.cz/~devik/qos/htb/](http://luxik.cdi.cz/~devik/qos/htb/)
