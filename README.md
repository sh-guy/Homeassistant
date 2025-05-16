# Homeassistant Snippits
Just some Home Assistant related stuff I wanted to archive.

- BlueConnect Go
  I have a BlueConnect Go in my Hot tub to measure water temperature, pH and ORP. There are two versions of this pool sensor - Go and Plus. Both are practically the same, except the Plus has an edditional Conductivity sensor that is also used to calculate salt level. So, the Plus is only really necessary if Salt is used for water desinfection. Also, the Plus is typically sold in a bundle with the Bluetooth to WiFi Gateway and the BlueRiiot subscription.
  This BlueRiiot subscription allows for automatic measurement updates to the cloud.
  I decided to use an ESP32 developer board as a Bluetooth to WiFi gateway into HA. There is a BlueConnect integration that would capture the BLE messages sent via a BLE Proxy. But, I decided to process the BLE messages locally on the ESP32 board and just integrate it via ESPHOME into HA. 
- Viessmann ViCare Cloud App

  My implementation of the ViCare Cloud by emulating the ViCare App connectivity.

  This implementation emulates how an iPad ViCare App connects to the ViCare Cloud in tablet mode (wall-mounted always on tablet).
  My intention was primarily to get the same data in my Homeassistant that I get in the App, including Smart Thermostat and Climate Sensor information.
  I also wanted the ability to trigger the open window functionality of the TRVs when a window open via a window sensor status change. Unfortunately, the open window feature cannot be triggered by the ViCare Cloud API. So, I ended up just setting the target temperature to 8° Celsius as a manual mode change and then when the window closes to remove the deactivate the manual mode (which automatically resets the target temperature to the set schedule.
  What I have not "yet" implemented is the ability to control the TRVs as Climate Entities in HA. So, you get current states and can modify e.g. target temperature within automations (like the open window automation.

  Now, this is not really an "integration" for Homeassistant, but rather some scripts and YAML code to create sensors and update their data from the ViCare Cloud.
