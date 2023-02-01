---
layout: post
title:  "LoRaWAN Packet Building in C++"
date:   2023-01-22 20:30:00 +1100
author: harvey
tags: [technology, c++, arduino, LoRaWAN]
categories: Technology
---

## Introduction
I made some errors in this [project](https://github.com/DPIclimate/ag-node) related
to the construction of a LoRaWAN packet to be sent from a device to a LoRaWAN gateway.

The main error was the use of an `int8_t` payload rather than `uint8_t`. I'm not sure
why I decided on it at the time, the method does work but its not correct.

I've since had a think about how I would go about this if I were to approach the task again.

## Original Method
The payload is designed to hold a rough maxiumum of 52 bytes to be sent from a device to a
gateway. These bytes can consist of various data types that are organised in some manner
that makes it easy to decode back into the original data on the receiving side.

### Building a payload
In the original method I would, create an `int8_t` buffer of size `PAYLOAD_SIZE`.
```c++
const uint8_t PAYLOAD_SIZE = 10;
static int8_t payload[10]
```

### `float` to `int` everywhere
I would only use integers as this makes it easier to get them into the buffer. To do this
I would multipy the `float` by 100 to get the two digits after the decimal place then
divide by 100 when decoding. For example:
```c++
float original_value = 102.82;
int16_t converted_value = (int16_t)(original_value * 100.0f);
// converted_value = 10282
```
You have to be careful as overflows can happen. For example:
```c++
float original_value = 1020.82;
int16_t converted_value = (int16_t)(original_value * 100.0f);
// converted_value = ??? (undefined)
```

### Assigning values to the payload
To assign values I individually bit shifted and assigned values into the buffer. For example:
```c++
payload[0] = converted_value;
payload[1] = converted_value >> 8;
...
```
**This continued for all 35 bytes in the payload... a total of 220 lines of code.**

## New Method
### Defining a payload
The alternative method I have come up with is to define the payload using `uint8_t` as its
type (as it should be). A `payload_index` is also defined to keep track of how many bytes
have been assigned to the payload.
```c++
const uint8_t PAYLOAD_SIZE = 18;
static uint8_t payload[PAYLOAD_SIZE];
uint8_t payload_index = 0;
```
### Generic packet
I then created a generic packet (`PKT`) which represents an individual data value of any type.
The type `T` is defined upon deceration.
`int8_t`.
```c++
template <typename T>
union PKT {
  T raw;                // Value
  uint8_t b[sizeof(T)]; // Values bytes
};
```
### Adding values to the payload
As `PKT` is templated a generic function can be added to append values of various types to the
payload.
```c++
template <typename T>
void add_to_payload(T value){
  if((payload_index + sizeof(value.raw)) > sizeof(payload)){
    std::cerr << "Error: Payload too small to accomodate value.\n";
    return;
  }

  uint8_t byte_index = 0;
  for(uint8_t i = 0; i < sizeof(value.raw); i++){
    payload[payload_index++] = value.b[byte_index++];
  }
}
```
The value size in bytes (`sizeof(value.raw)`) is checked against the remaining number of bytes
that can be assigned in the payload. If the value is too large and can not be assigned the
function returns.

If the payload has enough room for the value, it is appended to the payload as
[little-endian](https://en.wikipedia.org/wiki/Endianness).

Finally, to add values to the payload they can be defined as:
```c++
PKT<float> fvalue;      // Define PKT as type float
fvalue.raw = 10.22f;    // Assign value
add_to_payload(fvalue); // Append float to payload
```
To assign values of different types you can:
```c++
PKT<double> value;
// or
PKT<uint16_t> value;
// or
PKT<int8_t> value;
...
```

