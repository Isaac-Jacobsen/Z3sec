# Z3sec

*Z3sec* is an open-source penetration testing framework comprising a set of command line tools for ZigBee security research.
Primary targets of the framework are ZigBee-certified devices that implement either the ZigBee Light Link (ZLL) or the ZigBee 3.0 standard.

## Introduction

ZigBee is a wireless low-power standard in the Internet of Things (IoT).
While ZLL is an application profile designed for connected lighting systems, ZigBee 3.0 comprises various applications in the domain of smart home networks.
In order to simplify the process of setting up and managing a ZigBee network, the ZLL standard introduced a commissioning procedure called touchlink commissioning.
This commissioning procedure is an optional feature in the ZigBee 3.0 standard.

*Z3sec* can send arbitrary touchlink commands, and thus exploit security weaknesses in the touchlink commissioning procedure.
A local attacker is able to perform denial-of-service attacks or to take control over ZigBee-certified products.
For more information about the security weaknesses of touchlink commissioning and the procedures of attacks, please see [1].

## Installation

### Dependencies

+ gnuradio
+ gnuradio-dev
+ gr-foo
+ gr-ieee802.15.4
+ KillerBee
+ scapy-radio (https://bitbucket.org/cybertools/scapy-radio)
+ Ipython
+ Wireshark (optional)
+ python-crypto
+ python-sphinx (docs)
+ python-numpydoc (docs)
+ setuptools
+ (see `install_dependencies.sh` for a complete list)

#### Install steps (Ubuntu 16.10 x86_64)

1. `git clone https://github.com/IoTsec/Z3sec.git`
2. `cd Z3sec`
3. Install all needed dependencies with:
  - `bash ./install_dependencies.sh`
   (Password for sudo is requested when needed.)
4. Install Z3sec with:
  - `sudo python setup.py install`
5. Add the leaked ZLL master key in `~/.config/z3sec/touchlink_crypt.ini`. This
   configuration file is created at the first run of a Z3sec tool.

### Hardware

*Z3sec* supports two radio platforms: Ettus USRPs and radios, which are
supported by the KillerBee framework. In order to use Ettus USRPs, it is necessary to
install GnuRadio.

All tools of *Z3sec* are tested with the Ettus USRP B200 (GnuRadio) and the MoteIV Tmote Sky
(KillerBee).

#### Ettus USRP

1. Connect the USRP to the computer.
3. Verify the successful connection via `uhd_find_devices`.
3. Use the `--sdr` parameter in the *Z3sec* tools.

#### KillerBee Radio

1. Flash the KillerBee firmware as described on https://github.com/riverloopsec/killerbee
2. Connect the device to the computer.
3. Execute `sudo zbid` to get the device name.
4. Use the `--kb` argument together with the device name (e.g. `--kb /dev/ttyUSB0`) in the *Z3sec* tools.

### Tools

*Z3sec* is a collection of tools to evaluate security features of ZigBee-certified products.
The *Z3sec* tools assume basic knowledge of IEEE 802.15.4 and the ZigBee standards.
For a detailed description of the parameters of each tool, please refer to the output generated by the `--help` argument.

#### z3sec_touchlink

This tool partially reimplements the touchlink commissioning procedure from as an
initiator. It consists of several sub-tools, which can send specific touchlink commands.
The received signal strength of the touchlink commands must be above a certain threshold on the targeted device, i.e., these command must be sent from a particular proximity, also denoted as *touchlink range*.
For an evaluation of the wireless range of these tools, please see [1].

For the description of all parameters of a sub-tool, please refer to the help message, e.g., `z3sec_touchlink scan --help`.

- `scan`: Actively searches for touchlink-enabled devices in the touchlink range and displays
  an overview of all received scan responses. Here, and for all subsequent
  sub-commands, it is possible to specify the channels on which shall be
  searched for devices. By default, only channel 11 is scanned.

        z3sec_touchlink --sdr --channels primary scan
- `anti-scan`: Suppress scan requests of other touchlink initiators by impersonating them.
  This tool monitors channel 11 until a touchlink scan request of another
  initiator is received. This scan request is cloned and transmitted on all other
  channels. Touchlink-enabled devices on these channels respond immediately to the spoofed
  scan request, and won't reply to the scan request by the legitimate initiator.

        z3sec_touchlink --sdr anti-scan
- `reset`: Reset a touchlink-enabled device in touchlink range to factory-new. The target devices leave their
  current network and search for new networks. The target devices have to be recommissioned before they
  can be controlled again.

        z3sec_touchlink --sdr reset
  Due to an implementation bug, some (old) touchlink-enabled devices can be reset, even if they are not in touchlink range.
  This bug can be exploited in the following way:

        z3sec_touchlink --sdr reset --no_scan
- `identify`: Trigger the identify action (e.g. blinking) of a touchlink-enabled device in touchlink range.
  The identify duration can be set up to a theoretical maximum duration of 0xFFFE seconds (approximately 18 hours):

        z3sec_touchlink --sdr identify --duration 65535
  This command also supports the bypassing of the proximity check with `--no_scan`.
- `update`: Update the channel of a touchlink-enabled device in touchlink range.  If the channel is changed,
  the targeted device cannot communicate with its legitimate network anymore.

        z3sec_touchlink --sdr update --channel_new 26
- `join`: Request a touchlink-enabled device in touchlink range to join a new network.
  The configuration of the new network can be explicitly set:

        z3sec_touchlink --sdr --src_pan_id 0xF001 --src_pan_id_ext DE:FE:C8:ED:DE:FE:C8:ED join --channel_new 26 --network_key BAD1CECAFEFEED8BEEFD00DEB00BFA11

##### Note (1)

For the `join` command, the specification of the pre-configured touchlink link key (also known as ZLL Master Key) in `"~/.config/z3sec/touchlink_crypt.ini` is required.
This key was leaked in March 2015 and can be found online.


##### Note (2)

The `z3sec_touchlink` tool is not able to send acknowledgements within the
required time frame. However, some devices expect an acknowledgment for their
scan response, otherwise they abort the touchlink transaction. In order to
still send touchlink commands to those devices, one can impersonate
another ZigBee device in the wireless range of the target device by setting `--src_addr`, `--src_addr_ext`,
`--src_pan_id`, and `--src_pan_id_ext` according to the values obtained by a prior scan. The impersonated device then sends the
acknowledgment upon the reception of the scan response.

#### z3sec_key_extract

With this tool, the network key transmitted during a legitimate touchlink commissioning procedure
can be extracted and decrypted.

As ZigBee traffic source, a radio device can sniff actively for touchlink commissioning procedures on a specific channel:

    z3sec_key_extract --sdr --channel 11
Alternatively, the ZigBee traffic can be read from a pre-captured pcap file:

    z3sec_key_extract --pcap /path/to/file.pcap

##### Note

For this tool, the specification of the pre-configured touchlink link key (also known as ZLL Master Key) in `~/.config/z3sec/zll_master_key.txt` is required.
This key was leaked in March 2015 and can be found online.

#### z3sec_control
This tool keeps track of all ZigBee devices and networks on a specific
channel.

If the network key of a ZigBee network is known, this tool can be used to
interactively send command frames to devices of this network. The
network key for a network is passed on startup:

    z3sec_control --sdr --channel 15 --pan_id 0xF001 --nwk_key BAD1CECAFEFEED8BEEFD00DEB00BFA11
An interactive shell is opened in which Python/Scapy commands can be entered in
order to send command packets to nodes of this network.
By entering

    show()
a list of all devices in the network is printed:

    Network: 0
    pan_id:      0xF001
    ext_pan_id:  DE:FE:C8:ED:DE:FE:C8:ED
    network_key: bad1cecafefeed8beefd00deb00bfa11
    # Devices:   2
    # |addr   |ext_addr                |mac_sqn |nwk_sqn |sec_fc  |aps_fc |zcl_sqn |zdp_sqn |
    =========================================================================================
    0 |0xBAAD |DD:FF:88:0D:EE:FF:CC:66 |148     |229     |8323398 |3298   |23      |None    |
    1 |0xFA11 |EE:EE:28:0D:55:F1:98:1E |38      |112     |7249729 |2384   |6       |None    |
Now a command packet can be send to a device in the network. Suppose the second device in
the network is a light bulb, we can impersonate the other device in the network
and send a `light off` command to it

    send(0, 0, 1, pkt_off)
where the first argument specifies the ID of the network, the second
argument is the ID of the (sending) device, which we want impersonate,
the third argument is the ID of the device to which the
packet is addressed, and the last argument is the command packet.

##### Note
The command packet is automatically encrypted using the given network key and all the
address fields and sequence numbers are overwritten with those of the
impersonated and spoofed device.

## Literature

For more information about the security of touchlink commissioning and an extensive evaluation of the presented attacks, please see:

[1] P. Morgner, S. Mattejat, Z. Benenson, C. Müller, F. Armknecht:
**Insecure to the Touch: Attacking ZigBee 3.0 via Touchlink Commissioning**. In Proceedings of the 10th ACM Conference on Security & Privacy in Wireless and Mobile Networks (WiSec'17, Boston, MA, July 18-20, 2017)

If you refer to *Z3sec* in a publication, please cite [1].
