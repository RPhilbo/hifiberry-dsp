# Reverse engineering LG Sound Sync

LG TVs have a feature called "Sound Sync (optical)" that allows the TV to control the volume of a sound bar or speaker that is connected via SPDIF.
As this might be useful, let's see if we can find out how this works.

## Basics

SPDIF is a one-way protocol. There is no feedback from the receiver to the sender.
Therefore no negotiation between sender and receiver is possible. SPDIF sends a left/right data stream. In addition to the PCM data, additional status and user bits can be set. Unfortunately there is no common standard to encode control information into these bits.

I would expected that LG uses some of these bits to send additional control information.
Let's have a look.

## Status of the SPDIF registers with no active SPDIF

```
0xf61: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 02 02 00 8C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 02 00 8C 04
```

## Status of the SPDIF registers with active SPDIF

```
0xf61: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 02 02 00 8C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 02 00 8C 04
```

no change yet

## Status of the SPDIF registers with LG Sound Sync optical

```
0xf61: 00 00 00 00 00 00 00 00 00 00 01 E0 00 00 00 10 1F 04 8A 62 02 00 8C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 01 E0 00 00 00 10 1F 04 8A 60 02 00 8C 04
```

Now, we see additional channel status bits set

## Changing the volume

```
0xf61: 00 00 00 00 00 00 00 00 00 00 00 60 00 00 00 11 9F 04 8A 62 02 00 8C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 00 60 00 00 00 11 9F 04 8A 60 02 00 8C 04
```

Looks like this changes some status bits - cool :-)

## Setting volume to 0

```
0xf61: 00 00 00 00 00 00 00 00 00 00 01 F0 00 00 00 10 0F 04 8A 62 02 00 8C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 01 F0 00 00 00 10 0F 04 8A 60 02 00 8C 04
```

## Setting volume to max

```
0xf61: 00 00 00 00 00 00 00 00 00 00 07 B0 00 00 00 16 4F 04 8A 62 02 00 8C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 07 B0 00 00 00 16 4F 04 8A 60 02 00 8C 04
```

## Setting volume to 50%

```
0xf61: 00 00 00 00 00 00 00 00 00 00 02 D0 00 00 00 13 2F 04 8A 62 02 00 8C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 02 D0 00 00 00 13 2F 04 8A 60 02 00 8C 04
```

## Setting volume to 51%

```
0xf61: 00 00 00 00 00 00 00 00 00 00 02 C0 00 00 00 13 3F 04 8A 62 02 00 8C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 02 C0 00 00 00 13 3F 04 8A 60 02 00 8C 04
```

## Setting volume to 52%

```
0xf61: 00 00 00 00 00 00 00 00 00 00 02 B0 00 00 00 13 4F 04 8A 62 02 00 8C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 02 B0 00 00 00 13 4F 04 8A 60 02 00 8C 04
```

## Setting volume to 53%

```
0xf61: 00 00 00 00 00 00 00 00 00 00 02 A0 00 00 00 13 5F 04 8A 62 02 00 8C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 02 A0 00 00 00 13 5F 04 8A 60 02 00 8C 04
```

### Corresponding status when connected to a LG OLED55C9

```
0xf61: 00 00 00 00 00 00 00 00 00 00 02 B0 00 00 00 03 5F 04 8A 60 02 00 0C 04
0xf62: 00 00 00 00 00 00 00 00 00 00 02 B0 00 00 00 03 5F 04 8A 60 02 00 0C 04
```

## Muting

Pressing the mute button on the remote toggles two bits:

```
0xf61: 00 00 00 00 00 00 00 00 00 00 04 E0 00 00 00 05 0F 04 8A 60 02 00 0C 04
                                      |              |
0xf61: 00 00 00 00 00 00 00 00 00 00 0C E0 00 00 00 0D 0F 04 8A 60 02 00 0C 04
```

More visible at the bit level:

```
                                      4: 0100        5: 0101
                                      |  |           |  |
                                      C: 1100        D: 1101
```

(Data collected using a LG OLED55C9 with volume set to 80%.)

## Conclusions

It seems the volume information is encoded multiple times.

It might be the easiest way to use byte 16.5 (half of byte 16 and 17) that gives the volume as 0-100 when unmuted.
When muted, the first bit in that byte is set.

Checking bytes 17.5/19 for 0xF048A seems to indicate that Sound Sync is active.

## Other

- More about channel status bits 
https://www.av-iq.com/avcat/images/documents/pdfs/digaudiochannelstatusbits.pdf
