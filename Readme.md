# CAN-J1939 on linux

## CAN on linux

See [Wikipedia:socketcan](http://en.wikipedia.org/wiki/Socketcan)

## J1939 networking in short

* Add addressing on top of CAN (destination address & broadcast)

* Any (max 1780) length packets.
  Packets of 9 or more use **Transport Protocol** (fragmentation)
  Such packets use different CANid for the same PGN.

* only **29**bit, non-**RTR** CAN frames

* CAN id is composed of
  * 0..8: SA (source address)
  * 9..26:
    * PDU1: PGN+DA (destionation address)
    * PDU2: PGN
  * 27..29: PRIO

* SA / DA may be dynamically assigned via j1939-81  
  Fixed rules of precedence in Specification, no master necessary

## J1939 on SocketCAN

J1939 is *just another protocol* that fits
in the Berkely sockets.

	socket(AF_CAN, SOCK_DGRAM, CAN_J1939)

## differences from CAN_RAW
### addressing

SA, DA & PGN are used, not CAN id.

Berkeley socket API is used to communicate these to userspace:

  * SA+PGN is put in sockname ([getsockname](http://man7.org/linux/man-pages/man2/getsockname.2.html))
  * DA+PGN is put in peername ([getpeername](http://man7.org/linux/man-pages/man2/getpeername.2.html))  
    PGN is put in both structs

PRIO is a datalink property, and irrelevant for interpretation
Therefore, PRIO is not in *sockname* or *peername*.

The *data* that is [recv][recvfrom] or [send][sendto] is the real payload.
Unlike CAN_RAW, where addressing info is data.

### Packet size

J1939 handles packets of 8+ bytes with **Transport Protocol** fragmentation transparently.
No fixed data size is necessary.

	send(sock, data, 8, 0);

will emit a single CAN frame.

	send(sock, data, 9, 0);

will use fragementation, emitting 1+ CAN frames.

# Enable j1939

CAN has no protocol id field.
Enabling protocols must be done manually

### netlink

	ip link set can0 j1939 on

### procfs for legacy kernel (2.6.25)

This API is dropped for kernels with netlink support!

	echo can0 > /proc/net/can-j1939/net

# Using J1939

## BSD socket implementation
* socket
* bind / connect
* recvfrom / sendto
* getsockname / getpeername

## Modified *struct sockaddr_can*

	struct sockaddr_can {
		sa_family_t can_family;
		int         can_ifindex;
		union {
			struct {
				__u64 name;
				__u32 pgn;
				__u8 addr;
			} j1939;
		} can_addr;
	}

* *can_addr.j1939.pgn* is PGN

* *can_addr.j1939.addr* & *can_addr.j1939.name*  
  determine the ECU

  * receiving address information,  
    *addr* is always set,  
    *name* is set when available.

  * When providing address information,  
    *name* != 0 indicates dynamic addressing

## iproute2

### Static addressing

	ip addr add j1939 0x80 dev can0

### Dynamic addressing

	ip addr add j1939 name 0x012345678abcdef dev can0

# Kickstart guide to j1939 on linux

## Prepare using VCAN

You may skip this step entirely if you have a functional
**can0** bus on your system.

Load module, when *vcan* is not in-kernel

	modprobe vcan

Create a virtual can0 device and start the device

	ip link add can0 type vcan
	ip link set can0 up

## First steps with j1939

Use [testj1939](testj1939.c)

When *can-j1939* is compiled as module, load it.

	modprobe can-j1939

Enable the j1939 protocol stack on the CAN device

	ip link set can0 j1939 on

Most of the subsequent examples will use 2 sockets programs (in 2 terminals).
One will use CAN_J1939 sockets using *testj1939*,
and the other will use CAN_RAW sockets using cansend+candump.

testj1939 can be told to print the used API calls by adding **-v** program argument.

### receive without source address

Do in terminal 1

	./testj1939 -r can0:

Send raw CAN in terminal 2

	cansend can0 1823ff40#0123

You should have this output in terminal 1

	40 02300: 01 23

This means, from NAME 0, SA 40, PGN 02300 was received,
with 2 databytes, *01* & *02*.

now emit this CAN message:

	cansend can0 18234140#0123

In J1939, this means that ECU 0x40 sends directly to ECU 0x41
Since we did not bind to address 0x41, this traffic
is not meant for us and *testj1939* does not receive it.

### Use source address

	./testj1939 can0:0x80

will say

	./testj1939: bind(): Cannot assign requested address

Since J1939 maintains addressing, **0x80** has not yet been assigned
as an address on **can0** . This behaviour is very similar to IP
addressing: you cannot bind to an address that is not your own.

Now tell the kernel that we *own* address 0x80.
It will be available from now on.

	ip addr add j1939 0x80 dev can0
	./testj1939 can0:0x80

now succeeds.

### receive with source address

Terminal 1:

	./testj1939 -r can0:0x80

Terminal 2:

	cansend can0 18238040#0123

Will emit this output

	40 02300: 01 23

This is because the traffic had destination address __0x80__ .

### send

Open in terminal 1:

	candump -L can0

And to these test in another terminal

	./testj1939 -s can0:0x80

This produces **1BFFFF80#0123456789ABCDEF** on CAN.

	./testj1939 -s can0:

will produce exactly the same because **0x80** is the only
address currently assigned to **can0:** and is used by default.

### Multiple source addresses on 1 CAN device

	ip addr add j1939 0x90 dev can0

	./testj1939 -s can0:0x90

produces **1BFFFF90#0123456789ABCDEF** ,

	./testj1939 -s can0:

still produces **1BFFFF80#0123456789ABCDEF** , since **0x80**
is the default _source address_.
Check

	ip addr show can0

emits

	X: can0: <NOARP,UP,LOWER_UP> mtu 16 qdisc noqueue state UNKNOWN 
	    link/can 
	    can-j1939 0x80 scope link 
	    can-j1939 0x90 scope link

0x80 is the first address on can0.

### Use specific PGN

	./testj1939 -s can0:,0x12345

emits **1923FF80#0123456789ABCDEF** .

Note that the real PGN is **0x12300**, and destination address is **0xff**.

### Emit destination specific packets

The destination field may be set during sendto().
*testj1939* implements that like this

	./testj1939 -s can0:,0x12345 can0:0x40

emits **19234080#0123456789ABCDEF** .

The destination CAN iface __must__ always match the source CAN iface.
Specifing one during bind is therefore sufficient.

	./testj1939 -s can0:,0x12300 :0x40

emits the very same.

### Emit different PGNs using the same socket

The PGN is provided in both __bind( *sockname* )__ and
__sendto( *peername* )__ , and only one is used.
*peername* PGN has highest precedence.

For broadcasted transmissions

	./testj1939 -s can0:,0x12300 :,0x32100

emits **1B21FF80#0123456789ABCDEF** rather than 1923FF80#012345678ABCDEF

Desitination specific transmissions

	./testj1939 -s can0:,0x12300 :0x40,0x32100

emits **1B214080#0123456789ABCDEF** .

It makes sometimes sense to omit the PGN in __bind( *sockname* )__ .

### Larger packets

J1939 transparently switches to *Transport Protocol* when packets
do not fit into single CAN packets.

	./testj1939 -s20 can0:0x80 :,0x12300

emits:

	18ECFF80#20140003FF002301
	18EBFF80#010123456789ABCD
	18EBFF80#02EF0123456789AB
	18EBFF80#03CDEF01234567

The fragments for broadcasted *Transport Protocol* are seperated
__50ms__ from each other.  
Destination specific *Transport Protocol* applies flow control
and may emit CAN packets much faster.

	./testj1939 -s20 can0:0x80 :0x90,0x12300

emits:

	18EC9080#1014000303002301
	18EC8090#110301FFFF002301
	18EB9080#010123456789ABCD
	18EB9080#02EF0123456789AB
	18EB9080#03CDEF01234567
	18EC8090#13140003FF002301

The flow control causes a bit overhead.
This overhead scales very good for larger J1939 packets.

## Advanced topics with j1939

### Change priority of J1939 packets

	./testj1939 -s can0:0x80,0x0100
	./testj1939 -s -p3 can0:0x80,0x0200

emits
	
	1801FF80#0123456789ABCDEF	
	0C02FF80#0123456789ABCDEF

### using connect

### advanced filtering

## dynamic addressing
