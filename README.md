# LG air conditioner serial protocol

The page contains discoveries made while trying to understand the serial protocol used by LG air conditioners. 
LG machines often include a dongle which connects to the main indoor board. 
The dongle communicates using serial. 

# Connector

The connector is labeled as CN_LINK 
in my unit it is a 8 position female connector with male terminals. 
only 4 positions are in use with below clarifications: 

1. GND
2. NC
3. 12V
4. NC
5. LG TX - 5V
6. LG RX - 5V
7. NC
8. NC

The physical connector is a rare type originating from South Korea
SMh200-8H
it uses a 2.0mm pitch 

It may be possible to use the more widely known type JST HY which has the same pitch and somehow similar form factor. 


# Protocol

The dongle and protocol seems to be common to a large variety of LG appliances, including dishwashers, fridges, kitchen hoods. 
in this page, i have only resarched about air conditioner. 

## Serial parameters

The dongle uses a quite traditional 8N1 setting: 
* 8 data bits
* no parity bit
* 1 stop bit
* 9600 bps


## Format of frames

all values from serial line are in hexadecimal

There is a request response exchange, either dongle or AC can initiate the request. 
interestingly it seems to be full duplex, i.e. it is possible to send response to previous request and simultaneously listen to new requests. 

### preamble

There is always a 4 byte preamble 
04 00 00 00 

### command
followed by a 3 byte command
0x65	0x02	0x01 (example for power off)

### sequence number
1 byte sequence number
The sequence number is incremented for every subsequent request of the same kind 
remains at 0x00 for single time commands

### length of payload
1 byte length of payload 
which apparently restricts the payload length to 255
in my expereince the largest payload has been 124

### variable payload 
The payload includes a set of 'type-value' pairs. 
interestingly the length of type value is variable with no obvious clear rule to determine the length of the value or type. 

for example 
fan speed seems to use 7e as type
indoor temperature is ca d0 as type

### checksum CRC16

The two last bytes are CRC16 checksum of the entire frame including preamble to payload. 
The CRC polynom is CRC-XMODEM


# known commands





