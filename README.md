# S0CounterGasEnergy
read out Gas or energy meters with S0 counter

In this repo I'll explain on how to read out gas or energy meters with an D1 Wemos, that have (only) a S0 interface. The goal is to publish it via MQTT, so you can use it with your home automation system or other systems. 

First of, you'll have to find out, what kind of energy meter or gas meter you're using and if and how to access the S0 pulses. Usually the device either has an output for "GND/Ground" and "S0/Contact", which you can use. 
In my case, I use:
* elster IN-Z61 for reading out my "BK4" gas meter
* Eltako WSZ15D-65A for reading out my energy meter for my Whirlpool

The “BK4” gas meter (pretty standard in Germany), already has an predefined place for a reed switch like “elster IN-Z61”. So I plugged that in my BK4 and got the two necessary cables (“green” and “brown”). I also attached to cables to the outputs of my Eltako WSZ15D-65A

So, on the D1 Wemos, I placed the cables:

* GPIO-12 (D6) => for my gas meter
* GPIO-13 (D7) => for my energy meter
* GND => both GND cables

Now we have the S0 pulse counters on the D1 Wemos and are ready to go:

1. download (or flash directly) ESP Easy MEGA (don’t know, if Mega is needed, but sounds cool) from letscontrolit.com as described here: https://espeasy.readthedocs.io/en/latest/Reference/Flashing.html 
2. once, the ESP Easy is up, go into its web-UI and do the magic
3. add an MQTT-controller under “controller” (I used Home Assistant (openHAB) MQTT),
 you can adjust the topics to your liking in there!
4. add a new device “Gas” as follows

 * Internal PullUp: activated
 * GPIO ← Puls: GPIO-12 (D6)
 * Debounce Time (mSec): 10
 * counter Type: Delta/Total/Time
 * Mode Type: Falling
 * Data Acquistion/single event: unchecked
 * Send to Controller (1): activated (MQTT)
 * Interval: 30 sec
 * Values: leave as is (should be three: Count, Total and Time)
5. add a new device “Energy” as follows

 * Internal PullUp: activated
 * GPIO ← Puls: GPIO-13 (D7)
 * Debounce Time (mSec): 10
 * counter Type: Delta/Total/Time
 * Mode Type: Falling
 * Data Acquistion/single event: unchecked
 * Send to Controller (1): activated (MQTT)
 * Interval: 30 sec
 * Values: leave as is (should be three: Count, Total and Time)

6. now you have all you need, the D1 Wemos should now send the Pulses, the Total and Time-Delta to your MQTT topics defined in "Controller".

*Be aware! The "Total" is not persisted on the ESP, if it loses power, meaning it will reset on each power disconnect of the ESP.*

# Bonus
If you - for some reason are keen on calculating as much at the source as possible, especially if it’s crititcal to get the correct time calculations, you can let your D1 do the work for you! :wink:

So, what I did was not only send the raw pulses via MQTT (as explained in steps 1-6), but I wanted to let it calculate the kWh out of it, so I deactivated the “send to controller” part in step 4 and added a “Rule” in ESP Easy:

7. add “rules” for ESP Easy (“Tools” - “Advanced”: check the “rules” box and submit)
8. take Rule 1 (they’re numbered without the possibility to insert names
9. insert the following code for an energy meter with 2000 impulses / kWh

```
On System#Boot do    //When the ESP boots, do
  event,"Rules#Timer=1" // Force initial update
  loopTimerSet,1,30      //Set Timer 1 endless repeating for the next event every 30 seconds
endon

On Rules#Timer=1 do  //When Timer1 expires, do
  let,1,[Whirlpool#Count]*3600/2/30
  let,2,[Whirlpool#Total]/2000
  Publish openHAB/%sysname%/Whirlpool/W,[VAR#1]
  Publish openHAB/%sysname%/Whirlpool/kWh,[VAR#2]
  Publish openHAB/%sysname%/Whirlpool/Count,[Whirlpool#Count]
  Publish openHAB/%sysname%/Whirlpool/Total,[Whirlpool#Total]
endon
```

10. insert the following code in Rule2 for an gas meter with 100 impulses / m3 (1 imp for 0.01 m3)

```
On System#Boot do    //When the ESP boots, do
  event,"Rules#Timer=2" // Force initial update
  loopTimerSet,2,30      //Set Timer 1 endless repeating for the next event every 30 seconds
endon

On Rules#Timer=2 do  //When Timer1 expires, do
  let,1,[Gas#Count]*3600*1000/30*0.9122*11.271
  let,2,[Gas#Total]/100
  let,3,[Gas#Total]/100*0.9122*11.271
  Publish openHAB/%sysname%/Gas/W,[VAR#1]
  Publish openHAB/%sysname%/Gas/m3,[VAR#2]
  Publish openHAB/%sysname%/Gas/kWh,[VAR#3]
  Publish openHAB/%sysname%/Gas/Count,[Gas#Count]
  Publish openHAB/%sysname%/Gas/Total,[Gas#Total]
endon
```
11. deactivate the “Send to Controller” in the device
12. reboot the ESP

Now, it should send you the values every 30secs. I figured a shorter interval gets messy with the pulses, even if it’s as high as 3.000Watts or a bit higher, which my Whirlpool consumes, if heating is on. Luckily I have a potent PV! :wink:
As I said before, the "total" count is not reliable, as the ESP resets the Total on each power disconnect. So keep that in mind.

# Disclaimer
of course as usual: if you're not an electrician, you aren't meant to change anything in your electrical system, especially in Germany.
Do it on your onw risk!

# open
=> the gas-part is untested, because it’s summer and my solarthermic is heating up my heating storage, so I’ll have to wait for late autumn to get correct gas consumptions…
