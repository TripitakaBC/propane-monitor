# Integrate your 'remote ready' propane tank gauge with your home automation solution via ESPhome.

This project is based on a combination of hardware and software to facilitate obtaining, reading and processing the levels of propane in tanks fitted with a 'remote ready' gauge.


# About the 'remote ready' gauge

Most domestic propane tanks come with some form of gauge to show the level of propane present. If you have one of these gauges, it will probably have a plastic tab with the words 'remote ready' on it. If you do not have that gauge, this project will not work for you.

A quick Google search for 'remote ready propane gauge' will show many images and lots of articles of these devices and it is worth learning how they operate before commencing. In brief summary, the gauge consists of two parts, and in-tank component and a separate gauge that sits on the top. The in-tank component has a float that rides on the surface of the liquid propane and via a perpendicular cam assembly, turns a vertical shaft connected to a magnet as the propane rises (on filling) and falls (on use). As the shaft turns, so does the magnet which in turn moves the gauge needle which also has a magnet attached to it.

The function of a remote gauge uses a hall sensor to 'read' the change in the magnetic field and in turn, change the voltage present at the output of the Hall sensor. We map the scale of voltage change to fill percentage for a sensor in Home Assistant.

## Required Hardware

It seems that others choose to use the ESP32 for this project but I had a stock of ESP8266 with remote antennas so chose to use that hardware.

By nature, domestic propane tanks tend to be away from buildings and power sources so I utilised a dual 18650 battery shield for power. If you choose battery power, ensure that the source does not power down when there is no draw from it. This had me perplexed for a while when the ESP8266 went to deep sleep but then would not wake up. I discovered that when the power draw dropped, the battery bank also went into sleep mode and there was no power available to wake up the  ESP8266. I overcame this by bypassing the system and drawing power directly from the batteries.

As most tanks have a large metal cap over valves and sometimes gauges (mine does), the use of an external antenna allows the challenge to wifi to be overcome. You could also run a long cable to the gauge and position the battery and ESP somewhere other than under the cap.

The other piece of hardware is a Hall Effect Sensor; be sure to get the 49E version and not the others. I initially bought a KY-024 from AliExpress but I did not realise that all hall sensors have different functionality. I ended up with 3144 sensors which are a binary ON/OFF detection and stalled for a few days as I couldn't figure out why I wasn't getting changing voltage readings. You do not need all the ancillary equipment that comes with the KY-024, you just need a basic 49E hall sensor.

## Hardware Build

The build is pretty simple; power to the ESP board at 5v + GND with 3.3v to the hall sensor VCC pin, GND to GND and the hall sensor data pin to A0 on the ESP.

I discovered that when using deep sleep function, the ESP8266 requires D0 (GPIO16) bridged to reset (RST) for the device to wake. Unfortunately, I had intermittent issues flashing updated code to the device with this in place so I 'jumpered' the connection.

The hall sensor sits at the end of a length of 3-core wire. I had a spare length from a DS18b20 temperature sensor so I used that. Be careful not to short the sensor pins and secure the hall sensor under the dome of the plastic cap from the 'remote ready' gauge. I didn't find placement to be an issue worth spending a great deal of time on. I used hot-melt glue to fix the sensor in place.

I assembled all the components into a plastic 'Tupperware'-style lunchbox from Walmart.

## How the solution works

The hall sensor reads a variation in the magnetic field of the gauge and in turn, outputs a sliding scale voltage. This voltage is converted to a percentage value using a scale and offset calculator to derive values that are then placed into the lambda under the ADC sensor code. Scale value goes in the first parentheses and offset in the second. See 'Setup & Testing' below.

The propane tank level is unlikely to change quickly with values fairly constant over weeks and maybe even months. As the device is battery-powered, it makes sense to submit a sensor value before entering deep sleep for a prolonged period. In the code here, the device wakes once per week, submits the reading and then goes back into deep sleep.

For the purposes of OTA updates, testing and so on, we have the ability to suspend the deep sleep function via an MQTT publish message but this will only become effective the next time the device wakes. See 'Setup & Testing' below for suggestions for pre-deployment. As I have several future projects that require deep sleep, I created a generic override rather than a specific one for this device. You will need an MQTT integration to be working in HA for this solution to work. The setup for that is covered in detail in many tutorials elsewhere.

The reason for the MQTT domain entry in the configuration.yaml file is due to the changes in HA from the 2022.12.0 release onwards. Without this entry, the other sensors made available will not consistently update and when the device enters deep sleep, the dashboard sensor becomes unavailable.

## Setup & Testing

For the purposes of setup and testing, I suggest the following:

 - changing the values for run_duration: to 2 mins and sleep_duration: to 3 mins and comment out the deep sleep block
 - fill in your own specific information in the areas marked by {}.
 - comment out the lambda line so that voltage values are reported instead of percentage
 - change update_interval: to 10secs

Save this file under /config/esphome
Copy the code from configuration entry.yaml and paste into your own configuration.yaml. Double check your indentation after pasting.
Add the code from the automation entries.yaml file into your automations.yaml file. You may need to change the id of each if they conflict.
Add an entities card to your chosen dashboard and copy the code in admin panel.yaml to the YAML editor for that card, overwriting the basic code you see.
Restart HA
Add the newly-discovered device to ESPhome integration
Check your admin panel is reporting the device as available.

At this point, check the logs and you should see a new voltage reading every 10 seconds. If so, we are good to move onto 'field testing'. Head over to your propane tank and take your phone or laptop with you so you can see the voltage readings in the device logs in Esphome. Unscrew the two screws that hold the plastic gauge onto the tank. DO NOT unscrew the fitting that goes into the tank.
Slide the plastic 'remote ready' tab back into the plastic gauge and place the gauge in position on the tank fitting. Wait for the voltage readings to settle and record them along with the relevant position of the needle on the gauge. My tank was pretty full so I recorded at 70% then turned the gauge a little until it read 60% and recorded that voltage and so on through 50, 40, 30, 20 and 10%.
Take your maximum voltage reading and your minimum reading along with your maximum fill % and minimum fill % and enter the values in a scale and offset calculator. I used the one [here](https://www.vboxmotorsport.co.uk/index.php/us/calculators). If your minimum fill percentage is 10%, enter the corresponding voltage for 10% fill in the 'minimum voltage' box and below it, enter '10'. Do the same for the maximum voltage and fill percentage in the opposite boxes. Press 'Calculate and copy the scale to the first lambda parentheses and offset to the second. You can now uncomment the lambda line and install it to the ESP.

You should now be getting 10-second updates of fill percentage that correspond with the actual gauge reading. If not, repeat this procedure until you do. It may be necessary to manually correct the readings with a positive or negative offset in the lambda, i.e `- lambda: return x * (31.66360585143436) + (10) + 2.3;` If all looks correct, we can uncomment the deep sleep block and test to make sure that the device is entering and exiting deep sleep correctly. I found the best way to do this was to listen to the MQTT traffic using an external MQTT client. Alternatively, add 'last updated' as a secondary field to the sensor in the admin panel and watch for updates which should occur ONLY every 3 minutes.

Assuming all goes well, change the run_duration: to 5 mins and sleep_duration: to whatever period you desire. I chose weekly which is 10075min. I also changed the sensor update_interval: to daily which is 1440min. Install the changed code to the ESP and check functionality. Again, I used an external MQTT client to monitor traffic on the topic propanemonitor/# to make sure that everything was working as it should.
