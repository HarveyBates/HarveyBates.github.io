---
layout: post
title:  "Tutorial: Reading array from an Arduino using PySerial"
date:   2020-12-29 19:52:35 +1100
author: harvey
tags: [technology, arduino, c++, python]
categories: Tutorial
---

## Arduino side

In this example the Python program is going to act as the master script (ask for data) and your Arduino microcontroller is going to act as the slave (provide data).

### Setup
First we are going to setup the Arduino to read some random data from an analog pin.
```c++
#define readPin 2 // Define which analog pin to read from
int data[10]; // Create an empty array to store out values
void setup(){
    Serial.begin(9600); // Enable serial communications at a specific baud rate
}
```

Now we need to create two functions to first capture some data and save it to our array (`read_data()`) and then send the data via the serial port to our Python script (`send_data()`). 

```c++
void read_data(){
    /* The number of itterations is determined by getting the size of 
    our array (in bytes) divided by the size of the data type in our array 
    (in our case its an interger) */
    for(int i = 0; i < sizeof(data) / sizeof(data[0]); i++){
        // Read data and save to a specific position in our array
        data[i] = analogRead(readPin); 
    }
}

void send_data(){
    for(int i = 0; i < sizeof(data) / sizeof(data[0]); i++){
        // Print out our data to the serial port followed by a new line
        Serial.println(data[i]);
    }
}
```

Finally we need to make our microcontroller check to see if there are requests from the Python script to either `read_data()` or `send_data()`.

```c++
void loop(){
    /* Check if there are commands (sent from the Python script) 
    in the serial port */
    if(Serial.available()){
        // Read data until a new line character is reached
        command = Serial.readStringUntil('\n');
        /* Use a switch statment to determine what task the Python 
        script has requested */
        switch(command){
            /* If the command is the String "ReadData" preform the 
            function read_data() */
            case "ReadData" : read_data(); 
            /* If the command is the String "SendData" preform the 
            function send_data() */
            case "SendData" : send_data();
        }
    }
    delay(1000); // One second delay as this process isn't time sensitive
}

```
That wraps up the Arduino firmware, you can now upload this script to your Arduino as the rest of the tutorial will be done using Python.

## Python side

### Installation
If you havent already make sure you have PySerial installed. You can do this by typing this command into your terminal, console or command prompt. 
```bash
python3 -m pip install pyserial
```

Now we need to create a new python script and preform some inital setup.
```python
import pyserial
import time

# Find this by navigating to Tools/Ports in the Arduino IDE
portAddress = "YOUR_PORT_ADDRESS" 
# Needs to match the Arduino setup baudrate
baudRate = 9600 
# Set the array size of the array we want to read from the Arduino
arraySize = 10
# Intialise PySerial with a speific port address and baudrate
ser = serial.Serial(portAddress, baudRate) 
```

Now we need to define a function to ask the microcontroller to read data. 

```python
def read_data():
    ser.flush()
    ser.write(b"ReadData")
```

Next, we need to define a function to ask the microcontroller to send the data to out Python program. 

```python
def send_data(): 
    # Create empty array to store values
    data = []
    ser.flush() # Ensure the serial port is empty
    # Send a string to the Arduino in the form of bytes
    ser.write(b"SendData") 
    # Loop over the length of the array we are reading
    for _ in range(arraySize):
        line = ser.readline() # Get the value at each line
        # Remove the return characters and decode values
        line = str(line[0:len(line) - 2].decode("utf-8")) 
        # Append the value to our array as an interger
        data.append(int(line))
    return data
```

Finally, we need to run these commands in sequence and print out our array. 

```python
if __name__ == "__main__":
    read_data() # Ask the Arduino to read the data
    delay(2000) # Wait for the Arduino to finish reading
    data = send_data() # Ask the Arduino to send the data
    for values in data:
        print(values) # Print out each value in our array
    ser.close() # Close the serial port 
```

Hopefully that helped you. 

