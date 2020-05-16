# Xojo-BinaryMessage
Serialize structured data into binary format, you can use it for communications, store data, etc.

## How use it?

First, create a file (.bmsg extension for example) to define the message, example:

```
release = 1
namespace = tutorial

import "some.bmsg"
import "other.bmsg"

# example
message PhoneNumber {
  1 = number string,
  2 = type int8,
}

message People {
  1 = name string,
  2 = id int32,
  3 = email string,
  4 = phones() PhoneNumber,
}
```

Run the compiler (bmsgc.exe in windows or use Parser class) with the file name as parameter to create the .rbbas 
(or .xojo_code) files, add BinaryMessage module with their classes, add the files created with the compiler into a 
project. Now you can use "WriteTo(...)" method of serialize class into memoryBlock/binaryStream object, then you 
can send or store the bytes of memoryBlock/binaryStream to anywhere, when you need the data, load into 
memoryBlock/binaryStream object, create the object with the class created with compiler, use "ReadFrom(...)" to 
load bytes into instance of the object..

If you have already the class(es) just add "BinaryMessageTag" and/or "BinaryMessageTextEncoding" attrib to properties.

## Message definition

Each message has one or more fields. Field is enumerated by a TAG (id number) follow by a EQUAL, the NAME 
of the field and a TYPE. The NAME dont have spaces nor special caracters, the TYPE could be: int8, int16, int32, 
int64, uint8, unit16, uint32, uint64, float, double, bool, string, dictionary, another message NAME or array of another
message NAME or array of previous scalar types.

The TAG should not be changed once your message TYPE is in use. TAG number are between 1 and 512, why? 
because BinaryMessage are designed to handle small messages, however, you can handle large amount of data in 
string or uint8 arrays or other messages fields.

Arrays are define with an open and an close parenthesis after the message name field.

Field separator is comma, including in the last field.

If you need Date datatype, use double or uint64 TYPE and set it with totalSeconds.

Dictionary keys/values types are: boolean, int32, int64, single, double, string; others types or nil raise an exception.

Strings are encoded/decoded in UTF8, so if you need other encoding, change the encoding before use the string.


## Encoded

The message is encoded in binary format as a key-value pairs:

```
+-----+-------------------------+-------+
| key | value length (optional) | value | n pairs...
+-----+-------------------------+-------+
```

The key, UInt16 number (most significant bit (MSB) first) has this info:

Length = First 3 bits (0-7), length+1 in bytes of the value length when type is string, dictionary or array (optional).  
Tag = Next 9 bits (0-511), number of field.  
Type = Last 4 bits (0-15), type of field.  

Example 1: Length= 1, Tag= 1 and Type= 11 are key= `1B20` in hexadecimal.  
Example 2: Length= 0, Tag= 2 and Type= 0 are key= `2000` in hexadecimal.  
Example 3: Length= 3, Tag= 3 and Type= 15 are key= `3F60` in hexadecimal.  

The type of field could be:

0 = Int8, UInt8  
1 = Int16, UInt16  
2 = Int32, UInt32  
3 = Int64, UInt64  
4 = In32 Enum  
5, 6, 7 = Not used  
8 = Float  
9 = Double  
10 = Bool  
11 = String  
12 = Message Object  
13 = Dictionary  
14 = Array of scalar values  
15 = Array of message objects  

When the type are 11 the bytes of length has the next bytes for store the string.
```
+-----+---------------+--------+
| key | string length | string |
+-----+---------------+--------+
```

When the type are 13 the bytes of length has the size of dictionary (1 based).
```
+-----+-----------------+=====================+
| key | dictionary size | n key-value pair(s) |
+-----+-----------------+=====================+
```

When the type are 14 the bytes of length has the size of the array (1 based), next byte has the type of the array.
```
+-----+------------+---------------+============+
| key | array size | type of array | n value(s) |
+-----+------------+---------------+============+
```

Note: the type of array use also 4 (uint8), 5 (uint16), 6 (uint32), 7 (uint64) as type of field.

When the type are 14 and the byte type is string, next two bytes (uint16) has the size of the string element, next 
[size] bytes has the string value.
```
+-----+------------+=============+========+
| key | array size | string size | string | n pairs...
+-----+------------+=============+========+
```

When the type are 15 the bytes of length has the size of the array (1 based).
```
+-----+------------+===========+
| key | array size | key-value | n pairs...
+-----+------------+===========+
```


## Examples


### Example release:

```vb
  Dim release As UInt8= &b10000000 Or 1
  release= &b01111111 And release
```
  

### Example to encode/decode the key:

```vb
  Dim key, tag, typ, lng As UInt16
  
  typ= 15
  tag= 511
  lng= 7
  
  key= Bitwise.ShiftLeft(lng, 13) Or Bitwise.ShiftLeft(tag, 4) Or typ
  
  lng= Bitwise.ShiftRight(key, 13)
  tag= Bitwise.ShiftRight(&b0001111111110000 And key, 4)
  typ= &b0000000000001111 And key
```


### json:

```json
{
    "name": "John Doe",
    "id": 5,
    "email": "john@server.com",
    "phones": [
        {"type": 1, "number": "123 4567"},
        {"type": 2, "number": "555 5555"}
    ]
}
```

187 (179 in linux/macOS) bytes against 66 bytes of BinaryMessage


## Benchmark

win7 VM, realstudio2011r4.3 debug:  
  De-serialize JSON : 1.15043ms  
  Serialize to JSON : 0.17712ms  
  Serialize to BinaryMessage : 0.459ms  
  De-serialize BinaryMessage : 0.55957ms 

