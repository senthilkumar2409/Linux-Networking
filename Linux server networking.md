
## Network Interface:

In a Linux server, a **network interface** is the component that allows the server to communicate with a network.

Think of it as the **network card/connection** through which Linux sends and receives network traffic.

### 1. Common network interfaces

You may see interfaces like:

```bash
eth0
ens33
ens160
enp0s3
lo
```

* `eth0` / `ens33` / `ens160` → physical or virtual Ethernet interface
* `lo` → **loopback interface**, used for communication within the same server

For example:

```text
Linux Server
     |
     +--- ens33
     |      IP: 192.168.1.10
     |      MAC: 00:11:22:33:44:55
     |
     +--- lo
            IP: 127.0.0.1
```

### 2. Check network interfaces

The most common command is:

```bash
ip addr
```

or:

```bash
ip a
```

Example:

```text
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.1.10/24
    inet6 fe80::1234/64
    link/ether 00:11:22:33:44:55
```

Here:

* `ens33` → interface name
* `192.168.1.10/24` → IP address
* `00:11:22:33:44:55` → MAC address
* `UP` → interface is enabled
* `LOWER_UP` → physical/virtual link is active

### 3. Useful commands

**List interfaces:**

```bash
ip link
```

**Show IP addresses:**

```bash
ip addr
```

**Show routing table:**

```bash
ip route
```

Example:

```text
default via 192.168.1.1 dev ens33
192.168.1.0/24 dev ens33
```

This means traffic going outside the local network uses the **default gateway `192.168.1.1` through `ens33`**.

**Check a specific interface:**

```bash
ip addr show ens33
```

### 4. Why is this important in DevOps?

For a server such as an AWS EC2 instance or Kubernetes node, you can think of it like:

```text
                Internet
                   |
              Internet GW
                   |
             +-----+------+
             |   Router   |
             +-----+------+
                   |
              Network Interface
                 (eth0)
                   |
             +-----+------+
             | Linux OS   |
             |            |
             | IP Address |
             +------------+
```

The **network interface (NIC)** is where the server gets its network identity and through which packets enter and leave the machine.

In AWS, for example, an EC2 instance has an **Elastic Network Interface (ENI)**. The ENI can have private IPs, security groups, MAC address, etc.

### Simple interview definition

> **A network interface in Linux is a logical or physical network endpoint used by the operating system to send and receive network packets. It has properties such as an IP address, MAC address, and link state.**

If you're learning this for **Linux/DevOps**, the next important concept is understanding **NIC → IP → subnet → gateway → routing table → DNS**, because these all work together when troubleshooting network connectivity.
