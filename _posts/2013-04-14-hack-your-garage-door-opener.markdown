---
layout: post
title: How to hack your garage door opener
---

It’s well documented that the INSTEON [I/O Linc](http://www.smarthome.com/2450/IOLinc-INSTEON-Low-Voltage-Contact-Closure-Interface-1-In-1-Out-/p.aspx) can be used to control a garage door and monitor whether the door is open. It can even be accessed from a mobile device, making this a highly convenient home automation tool for less than $100. What was far less clear (to me, at least) was exactly how to connect the I/O Linc to the garage door opener. So, what follows is equal parts tutorial and lessons learned. The details will almost certainly vary, so be sure to consult the user manuals for your equipment.

## Equipment List

- [SmartLinc 2412n controller](http://www.smarthome.com/2412N/SmartLinc-INSTEON-Central-Controller/p.aspx)
- [I/O Linc](http://www.smarthome.com/2450/IOLinc-INSTEON-Low-Voltage-Contact-Closure-Interface-1-In-1-Out-/p.aspx)
- 20-28 gauge wire
- Garage door opener
- Magnetic sensor (optional)
- Soldering iron, wire nut, or some other means of joining wires (optional)


## Wiring the Garage Door Opener

A garage door opener (GDO) typically has several terminals on its rear panel:

![Terminals on garage door opener rear panel](http://content.screencast.com/users/a.jensen/folders/Snagit/media/bb483a31-b2d9-4613-bc0a-05670568a2a7/2013-04-13_21-03-40.png)

Some GDOs will have three terminals; others have four. For three-terminal openers, there are two “hot” terminals (one for devices that control the door, one for safety sensors) and a shared “common” terminal that’s normally in the center. A four-terminal GDO will normally have two terminals for controlling devices and two for sensors. The specific function of these terminals can vary from one GDO to the next, so be sure to track down the manual for your unit. (I hope you have better luck than I did.)

On my GDO—a Craftsman from circa 1997 with part number 41A5021-3B on the rear panel—there are three terminals:

![Craftsman garage door opener rear panel with terminal callouts](http://content.screencast.com/users/a.jensen/folders/Snagit/media/fe473009-3cfd-4713-b908-7dd089cfb501/2013-04-13_20-55-50.png)

To connect my GDO to the I/O Linc, I used a length of old (read: ugly) speaker wire that I had lying around. (I recommend using bell wire or one pair of wires from an RJ-45 cable as they will be a better fit for the I/O Linc terminals.)  One wire connects to the Remote/Opener Terminal (#1). The second wire connects to the common terminal (#2) in the center. The button on the wall will also be wired to these terminals.

Next, we’ll connect the other end of the new wires to the I/O Linc. The instruction sheet that ships with the I/O Linc is helpful and full of dense information, but it left me scratching my head. There is one very important wrinkle that must be dealt with before we can wire the I/O Linc to the GDO. The I/O Linc ships with its relay in *latch mode*, which means that when you press the button on the wall, the I/O Linc will close the relay and *keep it closed*. Most GDOs assume that the relay will close for a brief period (say, two seconds) and then open again. We need to configure the I/O Linc to respect this assumption.


## Configuring the relay  mode

1. Press and hold the “Set” button (on the side of the I/O Linc) until it beeps. Release the button. The I/O Linc is now in **linking mode**.
2. Press and hold the “Set” button (on the side of the I/O Linc) until it beeps. Release the button. The I/O Linc is now in **unlinking mode**.
3. Press and hold the “Set” button (on the side of the I/O Linc) until it beeps. Release the button. The I/O Linc is now in **output relay programming mode**. The unit automatically rotates to the next mode; if you were starting with a stock I/O Linc in “Latch” mode, it is now in “Momentary A” mode. This means that an “On” command will operate the door, and an “Off” command will be ignored. (If you prefer that “Off”—or even both commands—will trigger the door, then feel free to explore the “Momentary B” and “Momentary C” modes.)


## I/O Linc Wiring

We’re ready to connect the GDO to the I/O Linc. The other ends of our wires need to be connected to the Normally Open (N/O) and Common (COM) terminals on the I/O Linc:

![I/O Linc Wiring](http://content.screencast.com/users/a.jensen/folders/Snagit/media/861375c2-ba58-42e0-bcd3-293f8cc1e3e0/2013-04-13_23-02-53.png)

The other two wires in the photo are for a magnetic garage door sensor. More on that in a moment.

When the I/O Linc receives a signal from its controller (a [SmartLinc 2412n](http://www.smarthome.com/2412N/SmartLinc-INSTEON-Central-Controller/p.aspx) in my case), its relay closes, and the normally open circuit is closed for a brief moment. This is the very same event that occurs when you press the button on the wall. The GDO responds by opening, closing, or halting the garage door.


## Monitoring the door state

If you want to monitor whether the door is open or closed, then you’ll need a [magnetic door sensor](https://www.google.com/search?q=magnetic+door+sensor). These are inexpensive (< $5) and easy to install. Once the sensor is mounted, connect one wire to the I/O Linc’s "Sense" (S) terminal and the other to "Ground" (GND). When the door is closed, the green Sense light on the I/O Linc will be illuminated. When the door opens, the light will go out.

**N.B.**: If you plan to use the INSTEON mobile app, be advised that the “Status” for the I/O Linc is the status of the *relay*, not the door. When you open or close the door, the status will briefly change from Off to On and then back. If you want to monitor the open/closed state of the door, you will need a more sophisticated app. For Android devices, [MobiLinc](https://play.google.com/store/apps/details?id=com.mobileintegratedsolutions.mobilinc.lite) is a good option.


## Closing thoughts

Controlling and monitoring your garage door opener with INSTEON devices is not complicated, but pulling together the relevant instructions can be frustrating. Makers of garage door openers generally don’t explain how to do one-off installations, and the dense instruction manuals for the other components can be offputting. Hopefully I’ve found a comfortable middle ground with this tutorial. If you follow these steps and find that the steps are incorrect or that your equipment differs from what I’ve described, please leave a comment below. Good luck, be safe, and have fun.
