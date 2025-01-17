# Homeassistant Snippits
Just some Home Assistant related stuff I wanted to archive.

- Viessmann ViCare

  My implementation of the ViCare Cloud by emulating the ViCare App connectivity.

  This implementation emulates how an iPad ViCare App connects to the ViCare Cloud in tablet mode (wall-mounted always on tablet).
  My intention was primarily to get the same data in my Homeassistant that I get in the App, including Smart Thermostat and Climate Sensor information.
  I also wanted the ability to trigger the open window functionality of the TRVs when a window open via a window sensor status change. Unfortunately, the open window feature cannot be triggered by the ViCare Cloud API. So, I ended up just setting the target temperature to 8° Celsius as a manual mode change and then when the window closes to remove the deactivate the manual mode (which automatically resets the target temperature to the set schedule.
  What I have not "yet" implemented is the ability to control the TRVs as Climate Entities in HA. So, you get current states and can modify e.g. target temperature within automations (like the open window automation.

  Now, this is not really an "integration" for Homeassistant, but rather some scripts and YAML code to create sensors and update their data from the ViCare Cloud.
