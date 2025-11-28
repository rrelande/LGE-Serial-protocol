# LG air conditioner serial protocol

The page contains discoveries made while trying to understand the serial protocol used by LG air conditioners. 
LG machines often include a dongle which connects to the main indoor board. 
The dongle communicates using serial communication. 

This work is in progress andI hope this can be helpful to other owners of LG units
I will be glad to receive further information other users may decode based on this crude initial draft. 

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


## Format of frames (PDU)

all values from serial line are in hexadecimal

There is a request response exchange, either dongle or AC can initiate the request. 
interestingly it seems to be full duplex, i.e. it is possible to send response to previous request and simultaneously listen to new requests. 

### preamble

There is always a 4 byte preamble 
04 00 00 00 

### command
followed by a 3 byte command  
0x65	0x02	0x01 (example for power off).  

### sequence number
1 byte sequence number  
The sequence number is incremented for every subsequent request of the same kind  
Sequence number remains at 0x00 for single time commands  

### length of payload
* 1 byte length of payload 
This apparently restricts the payload length to 255 bytes  
in my experience the largest payload has been 124 bytes.  

### variable payload 
The payload includes a set of 'type-length-value' data units.  
* First 10 bits are for the data type (T)
* Subsequent 2 bit indicate the length (L) in byte of the value, 
Known to be valid length is 0 and 1, but longer length is seen.  
Lenght of 0 means that only 4 bits are used for value (V).
It means that 2 bytes are used for the full TLV PDU, which maps conveniently into the 8 bit framed serial data stream. 
(only 16 different values area available in this most basic TLV format).  
Length of 1 means that 3 bytes are used for the full TLV data unit.  
Length of Value field is then 4+8 bits=12 bits.  
* remaining bits are for encoding the value.
As indicated above, length of value is minimal 4 bits,
frequently 12 bits,  
occasionally longer.  

### Data type dictionary 
To help identify and document the identified TLV, the full 2 bytes (16 bits) are denoted here in documentation.  
The 10 first bits define the type of data unit. 
which is followed by 2 bits for length. A type always has a fixed associated length.  

This means that type and length (12 bits)
To help documentation, below denotation assumes that value (V) of remaining 4 bits (part of value) is 0.  

Example of data type in PDU  
* indoor temperature type is 7f 50 
* setpoint type is 7f 90

### checksum CRC16

The two last bytes are CRC16 checksum of the entire frame including preamble to payload. 
The CRC polynom is CRC-XMODEM, implementations of that CRC are widely available.  

### Example

A complete switch off command has the below content: 
0x04·0x00·0x00·0x00·0x65·0x02·0x01·0x00·0x02·0x7D·0xC0·0x73·0xC2



# known commands

## Switch on from dongle

complete serial frame: 
0x04·0x00·0x00·0x00·0x65·0x02·0x01·0x00·0x09·0x7D·0xC1·0x7E·0x40·0x7E·0x86·0x7F·0x90·0x2A·0x48·0x48

### Command 
0x65·0x02·0x01
### sequence
Sequence number is alway 0
### payload 7D C0 (operation on off)  
First 12 bits is 7D C  (Type and length)
value C1 for switch on
value C0 for swtich off
### payload 7E 40 (mode cool/auto/dehum..)
First 12 bits is 7E 40 (Type and length)
Possible values for the remaining 4 bits are: 
cooling		  40
dehum		    41
fan-only		42
auto		    43  

### payload 7E (fan mode)
Fan speed | Code 
----- | ---
Auto|  88
1		|  82
2		|  83
3		|  84
4		|  85
5		|  86

### payload 7F90 (setpoint)  
For some reason either the type is on 2 bytes with value 1 byte or the value is 2 bytes with value one byte.  
I have assumed that type is 2 bytes so far. 

The value of temperature in C is hex_value / 2

Setpoint C | Encoding LG 
----- | -----
21		|2a
22		|2c
23		|2E
24		|30
25		|32
26		|34
27		|36
28		|38
29		|3A

## switch off from dongle 
0x04·0x00·0x00·0x00·0x65·0x02·0x01·0x00·0x02·0x7D·0xC0·0x73·0xC2
### command 65 02 01
### payload 7D C0 
means request power off
### response from AC
0x04·0x00·0x00·0x00·0x87·0x02·0x10·0x00·0x00·0x77·0xE0
Above payload seems to be a 'standard' ack to a request from dongle 
#### ACK 87 02 10 
#### no payload


## update from AC 
sent on regular basis
### command 
0x87·0x02·0x04
### sequence number 
increments regularly 
as the AC sends command on regular basis
### Associated payloads
only one of the below payloads are seen with the command
#### payload 7F50 (indoor temperature)
0x7f·0x50·0x3d
type 7F 50
The value of temperature in C is hex_value / 2 and result converted to decimal.
Same as for setpoint. 
it only support .5 degree resolution. 

Setpoint C | Encoding LG 
----- | -----
21		|2a
22		|2c
23		|2E
24		|30
25		|32
26		|34
27		|36
28		|38
29		|3A

#### payload BE 81
This type and value is not identified
#### 0x06·0xBF·0xA0·0x03·0x05·0xBE·0x80·
it is not known if this is one type 0x06 with value on 6 bytes or several types together. 
Complete frame: 
0x04·0x00·0x00·0x00·0x87·0x02·0x04·0x87·0x03·0xCA·0xD0·0x8F·0x1B·0xA1
### response from dongle
0x04·0x00·0x00·0x00·0x65·0x02·0x10·0x87·0x00·0x8A·0x60
#### sequence number 
is same as the initial request from AC (here 87) 
#### no payload
no payload in response 

#### Command ACK
65 02 10 
no payload 
seems to acknoledge the update

## Periodical update from AC
roughly every 60s

0x04·0x00·0x00·0x00·0x87·0x01·0x02·0x01·0x02·0xAA·0x42·0x60·0x3E

### command 
0x87·0x01·0x02

### sequence 
seems to be always 0x01

### payloads
AA 42
There is no information what t  his means

### response from dongle
0x04·0x00·0x00·0x00·0x65·0x01·0x01·0x02·0x02·0xAA·0xC1·0xCC·0xDB


## periodical update from dongle
0x04·0x00·0x00·0x00·0x65·0x01·0x02·0x01·0x02·0xAA·0xC1·0xB9·0xD5
### command 65 01 02
### payload AA C1
it is not known what the payload means
### response from AC
0x04·0x00·0x00·0x00·0x87·0x01·0x10·0x01·0x00·0xDF·0x0D
#### ACK command 87 01 10 


