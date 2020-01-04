
Packets are human interprettable/enterable: 

`S10C0:00:cf000000\n`

The protocol is configured using a json file.

# Purpose

The RoBus protocol is designed to:
* Provide intuitive protocol configuration
* Speed up development/debugging on the comms layer

# Configuration

The json file has the following root schema:
* version: the current semver of the RoBus protocol
* category: a map of start characters used in packets
* separator: protocol separator character
* compound: protocol compound character
* end: protocol end character
* _data: the root of the applications data map

### Version

Version is a semvar version with major, minor and patch numbers. This version relates to the RoBus protocol itself. Its purpose is to determine compatability.

### Category

Category characters determine how the receiving application should interpret a packet.

The RoBus protocol defines six categories by default. However, an application may define more categories when needed.

### Separator, Compound, End characters

These configuration items define the utf-8 characters used in encoded packets. They may be changed to suit a users application if they meet the following rules:
* Must be copyable (eg, a delete character is not allowed)
* Must not be hexadecimal `[0-9a-f]`
* Must not be a category character

### Data

The `_data` field is a heirachical structure of user defined data.
 
```json
"_data" : [
  { "sensor" : { "_addr": "8000", "_data": [
    { "imu": { "_addr": "00A0", "_data": [
      { "accel": { "_data": [
        { "x": { "_type": "float" } }, 
        { "y": { "_type": "float" } }, 
        { "z": { "_type": "float" } }
      ]}},
      { "gyros": { "_data": [
        { "x": { "_type": "float" } },
        { "y": { "_type": "float" } },
        { "z": { "_type": "float" } }
      ]}},
    ]}},
    { "temperature": { "_addr": "00C0", "_type": "float" } },
    { "barometer": { "_type": "float" } }
  ]}},
  { "timestamp_ms": { "_addr": "9000", "_type": "u64" }}
] 
```

Results in the following address mapping:

| path                  | 16-bit address  | type                    |
| ---                   | ---             | ---                     |
| *sensor*              | *0x8000*        | -                       |
| *sensor/imu*          | *0x80A0*        | -                       | 
| *sensor/imu/accel*    | *0x80A1*        | -                       |
| *sensor/imu/accel/x*  | *0x80A2*        | float                   |
| *sensor/imu/accel/y*  | *0x80A3*        | float                   |
| *sensor/imu/accel/z*  | *0x80A4*        | float                   |
| *sensor/imu/gyros*    | *0x80A5*        | -                       |
| *sensor/imu/gyros/x*  | *0x80A6*        | float                   |
| *sensor/imu/gyros/y*  | *0x80A7*        | float                   |
| *sensor/imu/gyros/z*  | *0x80A8*        | float                   |
| *sensor/temperature*  | *0x80C0*        | float                   |
| *sensor/barometer*    | *0x80C1*        | float                   |
| *timestamp_ms*        | *0x9000*        | unsigned integer 64-bit |

Addresses:
* Addresses are always four character hexidecimal strings `/[0-9a-f]{4}/i`
* Items inherit the address of the parent object and then add thier own.
* If an item does not define an address, it increments from the last assigned address.
* Addresses must always increment (eg/ an address of 8000 following and address of C000 is invalid).
* Addresses never exceed 16-bits.

Items
* An item must have either a `"_data"` array or `"_type"` string
* Items are placed in arrays to *guarantee* order (its a JSON thing)
* Item names must start with a character and can only contain characters, numbers, dashes and underscores `/^[a-z][\w\-_]*$/i`

Valid types:
* *unsigned integers*: "u8", "u16", "u32", "u64"
* *signed integers*: "i8", "i16", "i32", "i64"
* float 32-bit: "float"
* float 64-bit: "double"
* string: "string"
* boolean: "bool"
* enumeration: [ "item1", "item2", ...]

For enumerations, the item names are user defined - these get encoded as integers. An enumeration can have no more than 256 entries.


# Packets

A typical packet looks like:

`S13e7:00:cf000000\n`

Where:
* `S` is the packet's category
* `13e7` is a data address
* `:` are item separation characters
* `00:cf000000` is encoded payload data
* `\n` is the packet's end character


### Category
The first character of a packet is always a category character. This specifies how the packet should be interpretted.

RoBus defines the following catergories by default:

| Category    | Default encoding | Description |
| ---         | ---              | ---         |
| *set*       | S                | Sets a data value, a response of *nak* or *ack* are recommended (though optional) |
| *ack*/*nak* | A/N              | Responds to *set* packets. <br>They should contain the same address(es) as the *set* packet, but no payload data |
| *get*       | G                | Requests a data value once, a response of *pub* with all contained data is expected.<br>   <ul><li>If the requested path has no children:<br> The response payload should contain only one item.<br> (eg: path *sensor/imu/accel/x* carries only one data item in the payload)</li><li>If the requested path has children:<br>The response payload should contain multiple items. <br> (e.g. path *sensor/imu* from above carries six items in the payload)</li></ul> |
| *sub*       | B                | Is the same as *get* but expects the data to be sent continously <br> The publishing rate is at the discretion of the sending application. |
| *pub*       | P                | Is used for sending application data values |

It is not necessary that an application implements all categories (eg. *ack* and *nak* can easily be omitted in many cases)

### Address

Addresses are encoded as a four character big-endian hexidecimal string.

### Separators

Separator characters go between addresses and data values. The default separator character is `:`

### Data

Encoding of data depends on the data type. All data gets encoded into some big endian hexidecimal string.
Data typing is inferred from the address and configuration, it is not carried in the packet itself.

| Type              | Encoding |
| ---               | ---      |
| Unsigned integers | 1 character per 4-bits, zero padded |
| Signed integers   | 2's complement, 1 character per 4-bits, zero padded |
| Float             | IEEE 754 binary 32, 8 characters |
| Double            | IEEE 754 binary 64, 16 characters |
| Bool              | Single character, only '0' or '1' is valid |
| Enumeration       | 2 characters to represent an unsigned 8-bit integer - enumerations are limited to 256 values |

### End

Packets terminate with an end character. The default end character is `\n`.

### Compound

Instead of an end charactor, packets can be combined using a compound character. The purpose of a compound character is to ensure sets of data are processed together.
The default compound character is `|`.


