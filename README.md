# sphero-ble
This is documentation for the Bluetooth Low Energy messaging format
used by the Sphero Mini, and probably other Sphero robots. It is
probably incomplete. You should already have a basic understanding of
BLE advertising, services, and characteristics.

This repository includes some sample code for the ESP-IDF (for the
ESP32) that lets the ESP32 control the Sphero.

# References
Thanks to `bbraun` on synack.net, whose blogs I stumbled across when I
was trying to make a [Mac Plus keyboard
interface](https://github.com/liamhays/mac-plus-serkey). One of them
was [this documentation](http://www.synack.net/~bbraun/spherodroid/),
which linked to an [old Sphero
page](https://web.archive.org/web/20170618112259/http://sdk.sphero.com/api-reference/api-packet-format/),
which has since been taken down but is on the Internet Archive. I was
trying to do this mostly independently, and the Sphero docs provided
me with a packet format an explanation of the checksum algorithm,
which I was unable to figure out.

I captured BLE packets using a micro:bit v1 and
[btlejack](https://github.com/virtualabs/btlejack), which captured the
packets sent between the Sphero Edu app on my phone and my Sphero
Mini.

# Advertising and connection
The Sphero Mini advertises a human-readable Bluetooth name that starts
with `SM-` and ends with 4 hex digits. `SM` stands for Sphero Mini,
and the 4 digits are to help differentiate Spheros if there are more
than one in the vicinity.

The Sphero advertises the *Sphero service*, a service with UUID
`00010001-574f-4f20-5370-6865726f2121`. The first block (`00010001`)
is a unique ID, while `574f-4f20-5370-6865726f2121` is the string "`WOO
Sphero!!`" encoded in ASCII.


## Services and characteristics
On connection, three services are available to the Bluetooth master:

| UUID                                   | Service                                                   |
|----------------------------------------|-----------------------------------------------------------|
| `00010001-574f-4f20-5370-6865726f2121` | Sphero service                                            |
| `00020001-574f-4f20-5370-6865726f2121` | auxiliary Sphero service (contains extra characteristics) |
| `0x180f`                               | BLE standard Battery Service                              |

Both the Sphero service and aux Sphero service offer several
characteristics, but only three total are needed:

| UUID                                   | Characteristic             |
|----------------------------------------|----------------------------|
| `00020005-574f-4f20-5370-6865726f2121` | wake characteristic        |
| `00010002-574f-4f20-5370-6865726f2121` | UART characteristic        |
| `0x2a19`                               | BLE standard Battery Level |

## Maintaining connection
While the Sphero will accept any connection request, it will
disconnect after a few seconds if it is not attached and woken up. To
attach the Sphero, the connecting device must write certain packets to
the wake characteristic.

First, the connecting device must send the attach packet, which tells
the Sphero not to disconnect. Write "`usetheforce...band`" (18 bytes)
to the wake characteristic with no response requested.

Once the attach packet has been sent, the Sphero must be woken. Write
the wake packet below (see "Data transfer" below for packet
information) to the wake characteristic with a response requested:

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ`  | `CHK`  | `EOP`  |
|--------|--------|--------|--------|--------|--------|--------|
| `0x8d` | `0x0a` | `0x13` | `0x0d` | `0x00` | `0xd5` | `0xd8` |

The wake packet will turn on the Sphero's RGB LED and enable the
stabilization system.

## Disconnecting
Once the Sphero is connected, attached, and woken, it will stay
connected until it reaches some inactivity timeout. I don't know how
long that timeout is.

After the timeout is reached, the Sphero will disconnect and remain
on, then enter a state where the color of the RGB LED depends on the
rotation of the Sphero. According to Sphero docs (which are still
available on the Sphero site), the Sphero Mini will go to sleep after
entering this state.

(need to include link to those docs).

If the Bluetooth master disconnects from the Sphero without sleeping
the Sphero, the Sphero will enter this same rotation-dependent color
mode. To sleep the Sphero, send these packets to the UART
characteristic (again, for more info, see "Data transfer" below):

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|-------|--------|
| `0x8d` | `0x0a` | `0x13` | `0x04` | `seq` | `chk` | `0xd8` |

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|-------|--------|
| `0x8d` | `0x0a` | `0x13` | `0x03` | `seq` | `chk` | `0xd8` |

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|-------|--------|
| `0x8d` | `0x0a` | `0x13` | `0x01` | `seq` | `chk` | `0xd8` |

I think the Sphero ignores the value of the `SEQ` field for these
commands.

Once these commands have been issued, the connection will linger for
about 30 seconds until the Sphero terminates it. The host can also
terminate the connection, but it must do so with reason `0x13`,
"Remote User Terminated Connection."

# Data transfer
The Sphero uses a BLE UART, where one characteristic is both the send
and receive characteristic. The BLE master writes a command packet to
the UART characteristic, usually requesting a response. The response
is then received via BLE notifications. Each byte sent by the Sphero
is a separate value notification on the UART characteristic.

I have not documented any data transfers from the Sphero to the
master. Most Sphero responses seem to be merely acknowledgments, in
the form of a command packet. There may be commands that cause the
Sphero to issue responses that include useful data.

## Packet format
A packet can be any length, but follows a common format. I refer to
byte as a *field*.

| `SOP1` | `SOP2` | `DID` | `CID` | `SEQ` | `packet data (variable length)` | `CHK` | `EOP` |

| Field  | Meaning                                  |
|--------|------------------------------------------|
| `SOP1` | Start of Packet 1 (always `0x8d`)        |
| `SOP2` | Start of Packet 2 (always 1 of 2 values) |
| `DID`  | Virtual Device ID                        |
| `CID`  | Command ID                               |
| `SEQ`  | Command sequence number                  |
| `...`  | packet data                              |
| `CHK`  | Packet checksum                          |
| `EOP`  | End of packet (always `0xd8`)            |

`SOP2` is always `0x0a` for a command being sent to the Sphero, and
`0x09` for a command sent to the Bluetooth master from the
Sphero. `DID` and `CID` vary and I have not documented them.

`SEQ` is a sequence number, which starts at 0 with the attach
packet. For almost every command, the Sphero requires the sequence
number to be one more than the `SEQ` field of the previous
command. When `SEQ` reaches 255, it resets to 0. In this
documentation, `seq` is used in the `SEQ` field to indicate a generic
sequence number.

The packet data can be any length.

`CHK` is a rudimentary checksum. It is calculated by summing every
byte from `SOP2` to the final byte of the packet data, then ANDing
that with `0xff` to get the low byte. That low byte is then
bit-inverted to get the final checksum. In this documentation, `chk`
is used in the `CHK` field to indicate a generic checksum that must be
calculated for every packet.

## Sending packets
Unless otherwise noted, **all packets should be sent with a response
requested.** The response does not have to be interpreted by the
Bluetooth master.

# Sphero features
## RGB LED
The Sphero Mini has one RGB LED, in the center of the top PCB. Each
base color (red, green, and blue) can be controlled with 8-bit precision.

To set the color, issue this command. Example values have been inserted as placeholders:

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` | ?      | ?      | `R`    | `G`    | `B`    | `R`    | `G`    | `B`    | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|--------|--------|--------|--------|--------|--------|--------|--------|-------|--------|
| `0x8d` | `0x0a` | `0x1a` | `0x0e` | `seq` | `0x00` | `0x7e` | `0x68` | `0x71` | `0xff` | `0x68` | `0x71` | `0xff` | `chk` | `0xd8` |

I have no idea what the question mark values indicate. However,
leaving them at the values indicated in the table appears to work for
any color value. Note that the RGB data appears twice in the packet.

The brightness slider in the Sphero Edu app just changes the intensity
of each color in the packet. It does not use a separate command.

## Aiming
To enter aiming mode, send these two packets in order:

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` |        |        |        |        |        |        | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|--------|--------|--------|--------|--------|--------|-------|--------|
| `0x8d` | `0x0a` | `0x1a` | `0x0e` | `seq` | `0x00` | `0x0f` | `0x00` | `0x00` | `0x00` | `0x00` | `chk` | `0xd8` |


| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` |        |        |        | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|--------|--------|--------|-------|--------|
| `0x8d` | `0x0a` | `0x1a` | `0x0e` | `seq` | `0x00` | `0x01` | `0xff` | `chk` | `0xd8` |

To orient the Sphero in a certain direction, use this packet:

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` | ?      | `deg1` | `deg0` | ?      | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|--------|--------|--------|--------|-------|--------|
| `0x8d` | `0x0a` | `0x16` | `0x07` | `seq` | `0x00` | `0x00` | `0xdd` | `0x00` | `chk` | `0xd8` |

`deg1` and `deg0` are the high and low bytes, respectively, of a
16-bit integer. This integer is just the degree value (0-360°) to
orient the Sphero to. In the Sphero Edu app, the aiming dial always
starts with the dial set to about 7 o'clock. The degree value is
relative to that point.

I don't know what the question mark packets are.

To exit aiming mode, send these three packets in order:

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` |        |        |        |        |        |        | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|--------|--------|--------|--------|--------|--------|-------|--------|
| `0x8d` | `0x0a` | `0x1a` | `0x0e` | `seq` | `0x00` | `0x0f` | `0x00` | `0x00` | `0x00` | `0x00` | `chk` | `0xd8` |

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|-------|--------|
| `0x8d` | `0x0a` | `0x16` | `0x06` | `seq` | `chk` | `0xd8` |

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|-------|--------|
| `0x8d` | `0x0a` | `0x1a` | `0x19` | `seq` | `chk` | `0xd8` |


## Motors
The Sphero is not a traditional robot. This packet only tells the
Sphero what relative direction to go in (in units of degrees, 0-360°),
and what speed to go (from 0 to 255).

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` | `speed` | `head1` | `head0` | ?      | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|---------|---------|---------|--------|-------|--------|
| `0x8d` | `0x0a` | `0x16` | `0x07` | `seq` | `0xbc`  | `0x01`  | `0x66`  | `0x00` | `chk` | `0xd8` |

`head1` and `head0` are just like `deg1` and `deg0`, see "Aiming"
above. I do not know what the question mark value is for.

`speed` is the speed at which the Sphero will move. In the Sphero Edu
app, this is some kind of combination between the little speed slider
and the position of the virtual joystick.

## Keep-Alive Packets
Counter-intuitively, the Sphero requires keep-alive packets to keep
from going to sleep---even if the BLE connection is still active. To
indicate that the Bluetooth master is still awake, two keep-alive
packets must be sent every 10 seconds.

The first packet is:

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|-------|--------|
| `0x8d` | `0x0a` | `0x13` | `0x04` | `seq` | `chk` | `0xd8` |

Now, I think that the second keep-alive packet must be send after the
response from the Sphero has been received. This only requires that
the Bluetooth master wait until it receives via notification `EOP`
(`0xd8`) from the Sphero, after sending the first keep-alive packet.

Once the first packet has been sent and the response acknowledged, the
second packet can be sent:

| `SOP1` | `SOP2` | `DID`  | `CID`  | `SEQ` | `CHK` | `EOP`  |
|--------|--------|--------|--------|-------|-------|--------|
| `0x8d` | `0x0a` | `0x13` | `0x03` | `seq` | `chk` | `0xd8` |

This packet does not require the host to monitor the Sphero's
response.

# Final word
The Sphero Mini is a comparatively simple Sphero robot (look at it
versus the Bolt, for example), and even considering that, I don't
think I have completely documented all the commands it supports. I
also suspect that the Sphero's responses contain plenty of useful
information.
