# Micropython encoder based menu


This is a simple menu system written in micropython.  It uses a switch,  a rotary encoder and an OLED display.  It was developed  on a Raspberry Pi Pico but also runs on an ESP32 and ESP8266.

The prototype used a little 128 * 64 pixel SSD1306 based OLED.  It could an be  adapted to other displays using micropython's framebuffer or even  to a very basic like a one-line display like a liquid crystal display.  The rotary encoder I used has a switch on the shaft, which is used as the click button.  The Rotaryirq esp32 library worked perfectly on Raspi Pico and the display used the library SSD1306IRQ.py

When the Pico is  started the root menu is shown and menuitems are actioned when the switch is clicked.  Any number of subitems can be shown.  Possible menu actions include running a Python funciton, entering an integer by twiddling the encoder and entering a string.

Since some functions can be slow or can block, the menu runs within an asyncio loop.


## Defining menus and Submenus

Menus are defined as a list of menu-items. Submenus are really the same as menus.
A menu item is defined as a pair of values:

	1.  A Caption  (string)
	2. An action function (a python function with NO parameters)

Example of a main menu and two submenus called trees and patterns.


  ```python
trees     = wrap_menu( [('gum',wizard),('ti-tree',info),('red-gum count',get_integer),('willow',treesize),('Back!',back)])

patterns  = wrap_menu( [('Chaser',yellow),('Red',red),('Blue',blue),('Rainbow',rainbow),('Back!',back)])

main_menu = wrap_menu( [('Patterns',patterns),('trees',trees),('Brightness',brightness)])

  ```


Here we have defined three menus, 'trees', 'patterns' and 'main_menu'.  

Note that the menu caption needs quotes (because it is a string), but the function does not because it is just the name of a function.

To transform our list into a function we pass it to another function called 'wrap_menu'.  Since our menu becomes a function,  it can be the function that is called from a menuitem.  In this way we can get sub-menus.

Menu function must be defined before they are used.  The root menu should be the last to be defined.



### Functions to get information

There are three predefined functions to get information:

1. get_integer
2. get_selection
3. wizard

#### get_integer

This allows us to set a number by twiddling the shaft of the encoder.  The number is entered by clicking the switch.  The result is stored in a global dictionary called data.  The key is set by a field parameter.

``` python
sethours   = get_integer(field = 'hour', low_v=1,high_v=24,increment=1,caption='Hours',field='hour')
```

Later the value can be retrieved from the global dictionary data  as shown in the example below:

``` python
if data['hour'] = 10:
	pass
	#Then do something

```

The value from the encoder ranges from low to high.  It goes up or down by one each time we turn the encoder by one click. Sometimes the desired value is a relatively high number, say 0-100 for percentages.

Doing 100 clicks is tedious if we do not really need that degree of precision. The encoder value is multiplied by Increment when it is stored or displayed.  In this way,  with an increment of 10 for instance,  we can go from 0 to 100 with 10 clicks.

#### get_selection

The selection function lets us get a string value from a list of values. The list is similar to a menu's list but  also has a field parameter.

```python 
colour1 = selection('colour1',['RED',('Green','GREEN'),('Blue','BLUE'),('Yellow','YELLOW'),('WHITE','white')])
```

The name displayed is the first value in the tuple and the value returned is the second element of the tuple.  There is an option to just provide a string (say "RED"). 
In this case the string value is exanded to a tuple ('RED','RED') behind the scenes.

As we turn the shaft the name of the various colours are scrolled, in the same way as a menu.  
When we click the shaft  value string is stored in the global dictionary and we return to the parent menu.  

**Default values for selection and get_integer**

A selection or get_integer is initialised to the value already in the dictionary,  if a value exists for that field, otherwise the initial value is zero or an empty .  This way we can get a default value by storing values in the dictionary before starting the program and we can revisit the selection to change values.

#### Wizard and get_integer

In small microprocessor systems we often want to enter a series of numbers.  For instance we may want to set   hours, minutes and seconds for a clock or day, month year to set the date.  

The wizard calls a series of functions in sequence, usually get_integer.  A  wizard  is defined similarly to a menu.

timewizard = wizard([("Hours",sethours),("Minutes",setminutes),("Seconds",setseconds)])

This wizard will gather hours, minutes and seconds in that order,   then return.
The list looks like the list used to define a menu but in fact the caption is ignored ( because a caption is provided to get_integer.)

 ### The info function.

 The info function just displays a screen of text which will be shown when you click its menuitem. Any click or scroll with clear the display (back to the parent menu).   You can provide the text as a simple string but you can also provide a function that returns a string.  This would allow you, for instance, to display the current time. 

Examples of these alternatives are show below:

```python
showtime = info('I dont have a clock')
showtime = info(my_gettime_function)  
# don't use brackets on the function name
```

 ##  

## How to get data out of the system.

An integer or a string  is returned by the functions get_integer and selection.  Both of these functions  have a parameter called field.  The fields is used as the key to store the value in a global dictionary.

Since the data dictionary is global it can accessed by other functions.

### Hardware abstraction functions

These functions are provided  as a form of hardware abstraction in case you wnat to us different libraries from  SSD1306_i2c,  rotary-irq library.

**set_encoder(minimum_value, maximum_value,increment)**

*setencoder* sets the rotary encoder so that value ranges between the maximum and minimum value inclusive. It increments on clockwise clicks and decrements on anti-clockwise clicks. It wraps over to minimum after click maximum and vice versa.  This is the standard behaviour of the library rotary_irq.

Calls  *rotary_irq.setencoder()*

Setencoder is provider as a function as a form of hardware abstraction in case you use a different rotary encoder library.

**get_value()**

Returns  the encoder value as an integer.

Same as rotary_irq.encoder.value()

***display(text)***

Display lines of text on the display device.  Up to 4 lines of text are allowed.



### Utility functions

***back()***

Clicking goes forward in a menu and scrolling goes up and down but we need a way to go back.  This is achieved by calling the back function as a menu action. (see menu examples).

If we had a way to provide more events than simple clicking and scrolling we could use one of these events to go back.  Possible sources for such an event would be another button or using long push or double click on a single button.  While easy to implement, this has not been done since the current system seems quite intuitive.

***make_task(coroutine)***

This simple turns a co-routine into a task and stores the task in a global variable called task.  Its main use is to hide the global.

***Stop()***

This cancels the global task above.  This way we can make and stop tasks without worrying about global declarations.  For instance if we have a rolling rainbow display on neopixels this should be run as a task, otherwise it will block the menu.  We can make the task by passing our function or more precisely our co-routine to *make_task()*.  If we want to run a different pattern we would stop the current pattern by calling *stop().*



 ##  Writing action functions.

There are several considerations in writing action functions.



**Simplest**

The simplest action function is just a normal python function with no parameters.



  ```python
  def  showblueneopixels():
    	make_strip_blue()
  ```

This will work  as a menu function.  The menu will be unchanged, which is usually fine, especially if your function is small and quick. 

**An action function with parameters.**

``` python
def showpixels(colour):
  make_strip(color)
 
#There are two ways to convert this into a function with no parameters
def showpixels(color = 'blue'):
  make_strip(color)
# calling showpixels() with no parameters will make the strip blue

def wrap_show_pixels():
  color = 'blue'
  def show_pixels(color):
    	make_strip(color)
  return show_pixels

# calling wrap_show_pixels() with no parameters will make the strip blue
  
```

**Writing a co-routine or multiasking.**

There are several tutorials about multitasking with asyncio. Peter Hinchs tutorial is a particularly good one but gets moderately advanced.

Action functions that run for a long time like endless loops should be written as co-routines otherwise they will block and stop the menu working.

However, if the function takes a long time to run it will block and the menu system will freeze.
We can get around blocking by writing an async function.  

I will assume that a  program is long running because it  has a loop, which is usually the case.

**The steps to turn a function  into a co-routine and then a task: **

1. Import uasyncio as asyncio to make the async functions available.

1. put async before def
2. Put "await asyncio.sleep(0)" into the loop so it is called frequently.  This makes our function play nicely with others, including the main menu.
3. Turn the co-routine into a task.  This also starts it running. We dont have to await it. Make_task(co-routine ) is provided as a utility.
4. cancel the task when we want our loop to stop. Stop() s is provided as a utility.

Example:

``` python
import uasyncio as aysncio
from encodermenu import make_task, stop

# Define this in the body of the program (or locally in the action function if you want)
async def loopy_function():
    do_something()
    await asyncio.sleep(0)
    
#In our action function
def my_action_function():
  stop() # stop any previously running tasks 
  make_task(loopy_function) # this will make a task and run it
  #note there are no brackets on our loopy_function

```

# How it works

* The system runs a bit like a normal event driven GUI.
* There is a event loop that polls the switch and the encoder. 
* If the switch is pressed or the value of the encoder changes then an on_click or on_scroll event is called.
* Events are handled by an object, which can be a Menu, GetInteger, Selection, Info, Wizard and so on.
* The object currently handling  events is a global variable called current.
* We change menus and entry screens etc by changing the current object.
* The convenience function back() pops the parent object off the stack and makes it current
* The main loop runs as an asyncio task so it does not block.
  Any function called within the menu system is also running within the asyncio loop so it can be a task or routine.
* A convenience function  make_task  stores the task in a global variable called task.
* A  convenience function called stop can cancels  the running task.
  (Note: This simple system will only handle one task. You can run more tasks,   but you will have to manange them yourself).


### Various issues

1. Be careful to define submenus before main parent menus.  If not you will get "variable not defined" errors.
2. Going back.  The system defines a scroll event for  going up and down the menu structure and a click event that goes forward.  
   You note that we have no event to go back.  To get around this there is a back function that can be called from a menu item. 



A menu is like a tree.  We start off on the trunk ("main_menu") and from this sub_menus branch off.  We can also branch off a submenu, so the menu can be arbitarily complex.   When we hit a leaf an action is performed. (Actually we can perform an action at any part of the tree but we will come back to that later.)

We have to write the main_menu last because it will refer to branchs above. In this case our display will show "Patterns" then "Trees" then "Brightness" as we turn the rotary encoder. If we click the "go" button (I use the switch on the shaft that comes with the encoder I used ) a function associated with the menu item will be executed. If we click on "Patterns" the patterns menu we defined in the line above will show.  Similarly for the "tees" menu. However if we click on Brightness we will excute a brightness function.  This allows us to enter an integer value for brightness. (More on this later).



An alternative would be to provide a back button.  Another alternative is to get some extra events off our simple button (like long-press or double click)  I will show how this can be done later. (The menu uses  uasyncio which  has  a good primitives libary from Peter Hinch that allows us to easily program for long presses or double clicks)

### Writing a menu (menus are functions and actions are functions)

We note that a menuitem is defined as tuple composed of a string menu item caption and an action that is performed if the menu is clicked, like so ('Caption1', action1).  A menu is a list of menu-items.  (Note - we could use either lists or tuples - it does not really matter).  

After we have provided a list of menu-items we have to wrap the list up into a function and we do this by using the wrap function.  This means that both actions and submenus are handled the same way in our system.  A submenu is simply an action that installs a new menu.

