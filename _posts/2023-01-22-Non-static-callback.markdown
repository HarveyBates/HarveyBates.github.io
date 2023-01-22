---
layout: post
title:  "C++ Callback Rabbit Hole"
date:   2023-01-22 20:30:00 +1100
author: harvey
tags: [technology, c++, arduino]
categories: Fun
---

## Introduction
I've had a play with making a CLI for the Arduino ecosystem 
([Arduino-CLAP](https://github.com/HarveyBates/arduino-clap)). I just want a 
quick and easy way to add commands and then pass values to functions for 
future projects.

Ideally the CLI should handle the parsing of arguments into their respective 
types. For example, if I pass and argument `port 8080`, it should recognise 
that the `port` argument accepts unsigned integers and parse the value `8080` 
into an unsigned int. This should result in less code duplication due to 
constantly converting from `const char*` to the datatype I want.

## Original Method
My command class known as `CL_Command` was designed to take a keyword (the 
	argument) and a value. It looked something like this:
```c++
class CL_Command{
  void (*callback)(){};	// Callback function ptr

  // Buffers for command information
  char name[8];
  char help[100];
  char takes_value = false;

  public:
  CL_Command(
	  const char* _name, 		// Argument name
	  const char* _help, 		// Argument help text
	  void (*_callback)(), 		// Callback function that accepts no value
	  bool _takes_value = false); 	// Does the callback function take a value
};
```

The issue with this method is if I want to add a callback function that accepts
a value, I need to generate a new constructor. For example, to accept a `const 
char *` I would need to add this constructor to my `CL_Command` class.
```c++
CL_Command(
	const char* _name,
	const char* _help,
	void (*_callback)(const char*), // <--- Add const char* here
	bool _takes_value = false); 	
```

Then to hold onto a pointer to that function I would need to add a function 
pointer for the new function type. E.g. `void (*callback)(const char*){};`.

It is pretty clear what is going to happen if I continued to add new function 
pointers that accept various data types.

## Templated Method
This seems like a good place to use templating. For example, if I use the 
original example and do some tweaking it becomes:
```c++
template <typename T, typename... U>
class CL_Command{
  // Buffers for command information
  char name[8];
  char help[100];
  char takes_value = false;

  public:
  T callback; // Templated callback function pointer

  CL_Command(
	  const char* _name, 		// Argument name
	  const char* _help, 		// Argument help text
	  T cb, 			// Callback function that accepts data type T 
	  bool _takes_value = false); 	// Does the callback function take a value
};
```

Now with templating we can use whatever data type we want to create our command
line argument. For example, using the same as above:

```c++
// Function to set a port number
void set_port(uint16_t port_number){
  port = port_number;
}

// Add a command line argument to set the port number
CL_Command<decltype(set_port)*, uint16_t> set_port_cmd("port", "Set port.", 
	set_port, true);
```

The use of `decltype` here gets the type of whatever `set_port` is. As it 
returns `void` and takes a `uint16_t` value, the line could be re-written as:
```c++
CL_Command<void(*)(uint16_t), uint16_t> set_port_cmd("port", "Set port.", 
	set_port, true);
```

## Evoking the Callback
Now to evoke the callback we can reference the `CL_Command` class instance 
`set_port_cmd` and pass or value to the callback.
```c++
// I've made the callback public here but it will be private within the CLI
set_port_cmd.callback(8080);
```

## Non-Static Member Functions
The examples above all reference static-member functions. This is useful about
60% of the time when using the Arduino ecosystem as you typically only have 
one class instance. But, for those cases when you need non-static member 
callbacks I'd like to support that as well.

For this I'll create an example class with a single setter for a non-static
member (`value`).
```c++
// Non-static class
class NS_Class{
  int value = 0; // Non-static member

  public:
  NS_Class(){}; // Basic constructor

  // Setter for value
  void set_value(int v){
	value = v;
  }
};
```

### Wrong Way
If like me, you went down the path of trying to add this function the same 
way as the previous example you would get an error. This would look like this:
```c++
NS_Class ns;
CL_Command<decltype(ns.set_value)*, int> sv_cmd("set_value", 
	"Set an int value.", ns.set_value, true);
// Uh oh error....
```

### The Right Way
If using something other than Arduino there is a `<functional>` header that 
can be imported to give this feature. However, for this project I am using 
an Arduino so the other method is to use a lambda expression.
```c++
NS_Class ns;
auto set_cb = [&](int v){ ns.set_value(v); };
CL_Command<decltype(set_cb)*, int> sv_cmd("set_value", 
	"Set an int value.", set_cb, true);

sv_cmd.callback(101); // Same as calling ns.set_value(101);
```

Here `[&]` captures all variables within the `ns` instance by reference. The 
function accepts an `int` which is passed to the instance of `NS_Class` (`ns`) 
and the `set_value` function is evoked. 

I do not know what this implys in terms of memory allocation so that will
be my next rabbit hole.


