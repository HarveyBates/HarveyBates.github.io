---
layout: post
title:  "Reading array from an Arduino using PySerial"
date:   2020-12-29 19:52:35 +1100
categories: jekyll update
---

## Arduino side
In this example the Python script is going to act as the master script (ask for data) and your Arduino microcontroller is going to act as the slave (provide data).

First we are going to setup the Arduino to read some random data from an analog pin.
{% highlight C++ %}
#define readPin 2 // Define which analog pin to read from
int data[10]; // Create an empty array to store out values
void setup(){
    Serial.begin(9600); // Enable serial communications at a specific baud rate
}
{% endhighlight %}

Now we need to create two functions to first capture some data and save it to our array (`read_data()`) and then send the data via the serial port to our Python script (`send_data()`). 

{% highlight C++ %}
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
{% endhighlight %}

Finally we need to make our microcontroller check to see if there are requests from the Python script to either `read_data()` or `send_data()`.

{% highlight C++ %}
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
{% endhighlight %}
That wraps up the Arduino firmware, you can now upload this script to your Arduino as the rest of the tutorial will be done using Python.

## Python side
If you havent already make sure you have PySerial installed. You can do this by typing this command into your terminal, console or command prompt. 
{% highlight command %}
python3 -m pip install pyserial
{% endhighlight %}

Now we need to create a new python script and import pyserial.
{% highlight python %}
import pyserial
import time

portAddress = "YOUR_PORT_ADDRESS" # Find this by navigating to Tools/Ports in the Arduino IDE
baudRate = 9600 # Needs to match the Arduino setup baudrate
arraySize = 10

ser = serial.Serial(portAddress, baudRate)

def send_request(readOrSend):
    global data
    ser.flush()
    time.sleep(1)
    if(readOrSend == "ReadData"):
        ser.write(b"ReadData")
        return
    else if(readOrSend == "SendData"):
        ser.write(b"SendData")
    else:
        raise NameError("Unidentified request recieved: {}".format(readOrSend))
    data = []
    for _ in range(arraySize):
        line = ser.readline()
        line = str(line[0:len(line) - 2].decode("utf-8"))
        data.append(line)

if __name__ == "__main__":
    send_request("ReadData")
    delay(2000)
    send_request("SendData")
    for values in data:
        print(values)
{% endhighlight %}


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
