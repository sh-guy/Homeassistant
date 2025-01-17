# Installation Guide
Step for step guide to retrieve Vicare Cloud sensor data into Homeassistant the same way the ViCare App accesses this data in your account.

## Step 1: Add your ViCare Email address and password to your secrets.yaml file.
First off create or edit your Homeassistant secrets.yaml file in your HA configuration folder. Add the following lines to the secrets file:
```
vicare_email: "YOUR_VICARE_ACCOUNT_EMAIL"
vicare_password: "YOUR_VICARE_ACCOUNT_PASSWORD"
```
Replace the placeholders with your account specific information. Keep the quotes since my authentication script parses the data between the quotation marks. There's probably a better way to do it. ;-)
## Step 2: Create Script and Rest Commands to authenticate with the ViCare Cloud API.
Authentication to the ViCare Cloud utilizes OAUTH with PKCE. That means in order to access the API you must first generate a so-called Bearer Authentication token prior to accessing the API. I won't go into the details. But, basically this is a two-request process to generate the necessary tokens. To reduce the amount of usage of your ViCare account credential, Viessmann supplies refresh tokens with the original authentication token. This allows us to refresh the authentication token without using your credentials again and again.

Currently, the Viessmann API grants access tokens for one hour and the refresh token for 180 days. So, this means techically your credentials are only needed twice a year to get new refresh tokens. Currently my implementation only renews the refresh token when HA starts. So, as long as you never have an uptime for you HA longer than 180 days, you should be good. Otherwise, we should figured out a better way to renew refresh tokens without requiring a restart of your HA instance. In my case I don't think I ever run my HA longer than 180 days without any changes that require a restart. ;-)

Since the initial creation of access tokens including refresh tokens is a two-step process, I utilize a shell script to create the tokens and populate a sensor with attributes that can be used by other REST Commands to access the API as well as updating the access token every hour using the stored refresh token.

Create a shell script somewhere within your configuration directory or subdirectory thereof and populate it with the following code. My implementation assumes your ViCare email and password are stored as above in your secrets.yaml and that this script is under your HA configuration/scripts directory. If you use a different directory structure, you will need to modify some parts of the following steps.

Requirements: make sure you can access the following commands on the cli of your HA instance: grep, awk, sha256sum, base64, curl. If these command are named differently or you use wget instead of curl, then you will need to modify the script accordingly.

You will also need the following HA components: uptime integration

/homeassistant/configuration/scripts/vicare_request_token.sh
```
#!/bin/bash

# get ViCare Account username from secrets.yaml
VICARE_EMAIL=`grep -Ev "^#|^$" /config/secrets.yaml | awk -F':' '{if ( $1 == "vicare_email" ) {print substr($2,3,length($2)-3)}}'`
# get ViCare Account password from secrets.yaml
VICARE_PASSWORD=`grep -Ev "^#|^$" /config/secrets.yaml | awk -F':' '{if ( $1 == "vicare_password" ) {print substr($2,3,length($2)-3)}}'`
# generate code verifier for OATH PKCE using random characters
CODE_VERIFIER=`cat /dev/urandom | LC_ALL=C tr -dc 'a-zA-Z0-9' | fold -w 43 | head -n 1`
# generate code challenge using above code verifier
CODE_CHALLENGE=`echo -n "$CODE_VERIFIER" | sha256sum | cut -d " " -f 1 | xxd -r -p | base64 | tr '+/' '-_' | tr -d '='`
# request code from ViCare IAM
CODE=`curl -s -d "isiwebpasswd=$VICARE_PASSWORD&isiwebuserid=$VICARE_EMAIL" -H "Content-Type: application/x-www-form-urlencoded" -H "accept: application/json" -H "user-agent: ViCare/3.29.5 (iPadOS 17.1; de_DE)" -X POST "https://iam.viessmann.com/idp/v3/authorize?response_type=code&redirect_uri=vicare%3A%2F%2Foauth-callback%2Feverest&scope=Internal%20openid%20offline_access&client_id=d382a049bd6ebd7e9a219d45b6de72f3&code_challenge_method=S256&code_challenge=$CODE_CHALLENGE" | grep code | awk -F'=' '{print $3}' | cut -c -43`
# use ViCare IAM code to generate access and refresh tokens
curl -s -H "Content-Type: application/x-www-form-urlencoded" -H "accept: application/json" -H "user-agent: ViCare/3.29.5 (iPadOS 17.1; de_DE)" -X POST "https://iam.viessmann.com/idp/v3/token" -d "grant_type=authorization_code&redirect_uri=vicare%3A%2F%2Foauth-callback%2Feverest&client_id=d382a049bd6ebd7e9a219d45b6de72f3&code_verifier=$CODE_VERIFIER&code=$CODE"

```
If you run this command directly on the CLI you should receive a response containing an access_token, refresh_token, id_token and an expiration date.

What we now need is a way to run this script and store the provided tokens as attributes of a sensor. I am using attributes as text sensor helpers or the state of sensors only accept up to 256 characters and the provided tokens are larger than that. Therefore I create a template sensor setting the state to the date and time the request was made and store the other values as attributes. In a later step we run a REST command to refresh these attributes with new access_token using the existing refresh_token.

In your configuration.yaml create a shell command referencing the above script so we can run it within a template sensor:
```
shell_command:
  vicare_fetch_tokens: './scripts/vicare_request_token.sh'

```

While you are in your configuration.yaml create a few REST commands for us to refresh the access token on an hourly basis and also two for sending any GET or POST request to the ViCare API (these will be necessary later on):
```
rest_command
  vicare_refresh_auth_token:
    url: "https://iam.viessmann.com/idp/v3/token"
    method: POST
    headers:
      accept: "application/json"
      user-agent: "ViCare/3.29.5 (iPadOS 17.1; de_DE)"
    content_type: application/x-www-form-urlencoded
    payload: "grant_type=refresh_token&client_id=d382a049bd6ebd7e9a219d45b6de72f3&refresh_token={{state_attr('sensor.vicare_tokens','refresh_token')}}"
    verify_ssl: false
  vicare_api_post:
    url: "{{ url }}"
    method: POST
    headers:
      accept: "application/json"
      user-agent: "ViCare/3.29.5 (iPadOS 17.1; de_DE)"
      Authorization: "Bearer {{state_attr('sensor.vicare_tokens','access_token')}}"
    content_type: application/json
    payload: '{{ payload }}'
    verify_ssl: false
  vicare_api_get:
    url: "{{ url }}"
    method: GET
    headers:
      accept: "application/json"
      user-agent: "ViCare/3.29.5 (iPadOS 17.1; de_DE)"
      Authorization: "Bearer {{state_attr('sensor.vicare_tokens','access_token')}}"
    content_type: application/json
    payload: '{{ payload }}'
    verify_ssl: false

```
Now we need a template sensor to store the tokens and other information into:
```
template:
  - trigger:
      - platform: homeassistant
        event: start
        id: ha_start
      - platform: time_pattern
        minutes: /45
        id: refresh_tokens
      - platform: template
        value_template: "{{ now() - states('sensor.uptime')|as_datetime > timedelta( minutes = 1) }}"
        id: ha_started
    action:
      - choose:
          - conditions:
              - condition: trigger
                id:
                  - ha_start
            sequence:
              - action: shell_command.vicare_fetch_tokens
                data: {}
                response_variable: auth_response
          - conditions:
              - condition: trigger
                id:
                  - refresh_tokens
            sequence:
              - action: rest_command.vicare_refresh_auth_token
                data: {}
                response_variable: auth_response
          - conditions:
              - condition: trigger
                id:
                  - ha_started
            sequence:
              - action: rest_command.vicare_api_get
                data:
                  url: 'https://api.viessmann.com/iot/v1/equipment/installations/?includeGateways=true'
                response_variable: auth_response
    sensor:
      - name: "Vicare Tokens"
        unique_id: vicare_tokens
        state: "{{ now().timestamp() | timestamp_custom('%Y-%m-%d_%H:%M:%S') }}"
        attributes:
          access_token: >
            {% if trigger.platform == 'homeassistant' %}
              {{ (auth_response['stdout']|from_json)['access_token'] }}
            {% elif trigger.platform == 'time_pattern' %}
              {{ auth_response.content['access_token'] }}
            {% elif trigger.platform == 'template' %}
              {{ state_attr('sensor.vicare_tokens','access_token') }}
            {% endif %}
          refresh_token: >
            {% if trigger.platform == 'homeassistant' %}
              {{ (auth_response['stdout']|from_json)['refresh_token'] }}
            {% elif trigger.platform == 'time_pattern' %}
              {{ auth_response.content['refresh_token'] }}
            {% elif trigger.platform == 'template' %}
              {{ state_attr('sensor.vicare_tokens','refresh_token') }}
            {% endif %}
          id_token: >
            {% if trigger.platform == 'homeassistant' %}
              {{ (auth_response['stdout']|from_json)['id_token'] }}
            {% elif trigger.platform == 'time_pattern' %}
              {{ auth_response.content['id_token'] }}
            {% elif trigger.platform == 'template' %}
              {{ state_attr('sensor.vicare_tokens','id_token') }}
            {% endif %}
          install_id: >
            {% if trigger.platform == 'template' %}
              {{ (auth_response.content['data'] | first).id | int(0) }}
            {% elif trigger.platform == 'time_pattern' %}
              {{ state_attr('sensor.vicare_tokens','install_id') | int(0) }}
            {% endif %}
          gateway_id: >
            {% if trigger.platform == 'template' %}
              {{ ((auth_response.content['data'] | first).gateways | first).serial | int(0) }}
            {% elif trigger.platform == 'time_pattern' %}
              {{ state_attr('sensor.vicare_tokens','gateway_id') | int(0) }}
            {% endif %}

```
This is a template sensor that updates its attributes on three different triggers:

1. On HA start the initial access_token, refresh_token and id_token are stored.
2. Every 45 minutes (although I think my time_pattern here actually updates at x:45 of every hour instead of every 45 minutes) update the access_token using the refresh_token.
3. After HA has started and has an uptime of one minute, then use the stored access_token retrieved during restart to get the installation and gateway IDs for the relevant ViCare Account. (My implementation only supports one installation gateway. So if you have multiple then you will need to modify my scripts to capture the status of more than one installation. For the typical consumer, you will probably only have one installation.)

You may have noticed that the 3rd trigger utilizes the vicare_api_get command from above. This is a generic GET request where you can provde any URL and Payload of the ViCare API. We will also be using the vicare_api_post command to access the primary URL to get all installation data needed in one request. More about that in the next step.

**!! If you have not done so already, then restart HA so these components become active. This will be necessary for the following steps. !!**

After HA restarts, validate the access_token and refresh_token attributes are created in the vicare_tokens sensor. After a minute or two of uptime the attributes gateway_id and install_id should also be added to that sensor. Continue to the next step only when the vicare_tokens attributes have been set. If not, check your logs where something may be going wrong with the configuration up to this point. Without the vicare_tokens sensor and its attributes, the rest of this implementation will not work. 

## Step 3: Retrieve ViCare Cloud configuration (Gateway and Device information).
Now we need to request the configuration of your installation to locate which sensors you want to create and have automatically updated in HA. HA will poll this URL once a minute to update your defined sensors going forward. Be advised, the repsonse from this request has a lot of information in it. So, you will want to copy the contents to a text file (preferably in a text editor with JSON formatting for ease of viewing).

Go to the HA Developer Tools under "ACTIONS" and select the previously created RESTful command: vicare_api_get as your action and switch to YAML mode (since we are sending variables to the REST command which is not supported in UI mode. Now paste the following in the YAML window:
```
action: rest_command.vicare_api_get
data:
  url: "https://api.viessmann.com/iot/v2/features/installations/{{ state_attr('sensor.vicare_tokens','install_id') | int(0) }}/gateways/{{ state_attr('sensor.vicare_tokens','gateway_id') | int(0) }}/features?includeDevicesFeatures=true"

```
Copy the response of this request to a text file (use the Copy to clipboard button at the bottom of the response screen).

## Step 4: Create relevant sensors in Homeassistant based on the information retrieved in Step 3.
If you've gotten this far, then you are ready to setup the sensors you want to monitor in HA. Depending on your installation this step could be a big task. But if you only want to monitor your heater and a few TRVs then it shouldn't take too much time to set this up.

We will now create a RESTful Sensor that populates all the sensor data you would like to capture from the above Step 3. Since I had a lot of sensors to create, I decided to create a separate rest.yaml file and referenced it in my configuration.yaml with:
```
rest: ./rest.yaml
```
But you can just add the contents of the rest.yaml file directly under the rest: declaration directly in your configuration.yaml if you like.

Create a rest.yaml file with the following example contents (you will need to modify parts (IDENTIFIED_IN_CAPS) and copy and paste parts to create your specific sensor configuration). In all cases yu should modify the "name" and "unique_id" to how you prefer things to show up in HA. The Boiler Sensors would probably work for most without change (except for naming). It's when you get to the Room Sensors where you will need to modify things to apply to your installation.

You will also notice that value based "sensor" and "binary_sensor entities are created. For Room Sensors I will only provide a few examples for you to duplicate and modify device ID information for your specific installation.

/homeassistant/configuration/rest.yaml
```
#rest: # Uncomment this line if using directly in configuration.yaml
## Request Vicare Cloud configuration and current status
  - resource_template: "https://api.viessmann.com/iot/v2/features/installations/{{ state_attr('sensor.vicare_tokens','install_id') | int(0) }}/gateways/{{ state_attr('sensor.vicare_tokens','gateway_id') | int(0) }}/features?includeDevicesFeatures=true"
    method: GET
    headers:
      accept: "application/json"
      user-agent: "ViCare/3.29.5 (iPadOS 17.2; de_DE)"
      Authorization: "Bearer {{state_attr('sensor.vicare_tokens','access_token')}}"
      content-type: "application/json"
    verify_ssl: false
    scan_interval: 60 # Interval to update sensor data
    sensor:
## Boiler Sensors
### Typical Boiler values
      - name: "Viessmann Boiler Temperature"
        unique_id: vicare_custom_viessmann_boiler_temp
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.boiler.temperature') | first).properties.value.value }}"
        device_class: temperature
        state_class: measurement
        unit_of_measurement: "°C"
        icon: mdi:thermometer
      - name: "Viessmann Abgastemperatur"
        unique_id: vicare_custom_viessmann_abgas_temp
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.flue.sensors.temperature.main') | first).properties.value.value }}"
        device_class: temperature
        state_class: measurement
        unit_of_measurement: "°C"
        icon: mdi:thermometer
      - name: "Viessmann Aussentemperatur"
        unique_id: vicare_custom_viessmann_aussen_temp
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.sensors.temperature.outside') | first).properties.value.value }}"
        device_class: temperature
        unit_of_measurement: "°C"
        state_class: measurement
        icon: mdi:thermometer
      - name: "Viessmann Vorlauftemperatur"
        unique_id: vicare_custom_viessmann_vorlauf_temp
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.circuits.0.sensors.temperature.supply') | first).properties.value.value }}"
        device_class: temperature
        state_class: measurement
        unit_of_measurement: "°C"
        icon: mdi:thermometer
      - name: "Viessmann Burner Modulation"
        unique_id: vicare_custom_viessmann_burner_modulation
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.burners.0.modulation') | first).properties.value.value }}"
        unit_of_measurement: "%"
        icon: mdi:percent
        state_class: measurement
      - name: "Viessmann Burner Hours"
        unique_id: vicare_custom_viessmann_burner_hours
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.burners.0.statistics') | first).properties.hours.value }}"
        unit_of_measurement: "h"
        state_class: total_increasing
      - name: "Viessmann Burner Starts"
        unique_id: vicare_custom_viessmann_burner_starts
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.burners.0.statistics') | first).properties.starts.value }}"
        state_class: total_increasing
      - name: "Viessmann Active Mode"
        unique_id: vicare_custom_vviessmann_active_mode
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.circuits.0.operating.modes.active') | first).properties.value.value }}"
      - name: "Viessmann Active Program"
        unique_id: vicare_custom_viessmann_active_program
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.circuits.0.operating.programs.active') | first).properties.value.value }}"
### Domestic Hot Water Temperature - only necessary with existing hot water storage
      - name: "Viessmann WW Temperatur"
        unique_id: vicare_custom_viessmann_ww_temp
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.dhw.sensors.temperature.dhwCylinder') | first).properties.value.value }}"
        device_class: temperature
        state_class: measurement
        unit_of_measurement: "°C"
        icon: mdi:thermometer
## Room Sensors
### Example ViCare Climate Sensor values. Replace ROOM_NUMBER with the relevant room number from your configuration for Humidity and Temperature. Replace ZIGBEE_ID with your zigbee IDs for the Battery and Signal Strength values.
#### Climate Sensor Current Humidity
      - name: "ViCare Luftfeuchtigkeit Room 1"
        unique_id: vicare_custom_luftfeuchtigkeit_room_1
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'rooms.ROOM_NUMBER.sensors.humidity') | first).properties.value.value }}"
        unit_of_measurement: "%"
        icon: mdi:water-percent
        state_class: measurement
        device_class: humidity
#### Climate Sensor Current Temperature
      - name: "ViCare Temperatur Room 1"
        unique_id: vicare_custom_temperature_room_1
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'rooms.ROOM_NUMBER.sensors.temperature') | first).properties.value.value }}"
        device_class: temperature
        state_class: measurement
        unit_of_measurement: "°C"
        icon: mdi:thermometer
#### Climate Sensor Battery level
      - name: "ViCare Batterie Klimasensor Room 1"
        unique_id: vicare_custom_batterie_klimasensor_room_1
        value_template: "{{ (value_json.data | selectattr('feature', 'match', 'device.power.battery') | selectattr('deviceId', 'equalto', 'zigbee-ZIGBEE_ID') | first).properties.level.value }}"
        device_class: battery
        state_class: measurement
        unit_of_measurement: "%"
        icon: mdi:battery
#### Climate Sensor Signal Strength
      - name: "ViCare Signal Klimasensor Room 1"
        unique_id: vicare_custom_signal_klimasensor_room_1
        value_template: "{{ (value_json.data | selectattr('feature', 'match', 'device.zigbee.lqi') | selectattr('deviceId', 'equalto', 'zigbee-ZIGBEE_ID') | first).properties.strength.value }}"
        device_class: signal_strength
        state_class: measurement
        unit_of_measurement: "dBm"
        icon: mdi:zigbee
### Example ViCare TRV values. Replace ZIGBEE_ID with the relevant zigbee IDs for your installation
#### TRV Current Temperatur. For current temperature you can either use the above room-based configuration for Climate Sensors (since TRVs are associated with a room typically.) Or get the devices measured temperature as in this example. When there is only one TRV in a room and no Climate Sensor is in use, then this is probably what you want. When more than one TRV is in a room or when a Climate Sensor is in use, the room-based configuration is how the TRV will be controlled. And if you ever plan adding Climate Sensors, using the room-based temperature would require no changes after adding them.
      - name: "ViCare TRV Temperatur Room 1"
        unique_id: vicare_custom_trvtemperature_room_1
        value_template: "{{ (value_json.data | selectattr('feature', 'match', 'trv.sensors.temperature') | selectattr('deviceId', 'equalto', 'zigbee-ZIGBEE_ID') | first).properties.value.value }}"
        device_class: temperature
        state_class: measurement
        unit_of_measurement: "°C"
        icon: mdi:thermometer
#### TRV Target Temperatur
      - name: "ViCare Solltemperatur Room 1"
        unique_id: vicare_custom_solltemperature_room_1
        value_template: "{{ (value_json.data | selectattr('feature', 'match', 'trv.temperature') | selectattr('deviceId', 'equalto', 'zigbee-ZIGBEE_ID') | first).properties.value.value }}"
        device_class: temperature
        state_class: measurement
        unit_of_measurement: "°C"
        icon: mdi:thermometer
#### TRV Valve Position
      - name: "ViCare Ventilposition Room 1"
        unique_id: vicare_custom_ventil_room_1
        value_template: "{{ (value_json.data | selectattr('feature', 'match', 'trv.valve.position') | selectattr('deviceId', 'equalto', 'zigbee-ZIGBEE_ID') | first).properties.position.value }}"
        state_class: measurement
        unit_of_measurement: "%"
        icon: mdi:percent-circle
#### TRV Battery Level
      - name: "ViCare Batterie Room 1"
        unique_id: vicare_custom_batterie_room_1
        value_template: "{{ (value_json.data | selectattr('feature', 'match', 'device.power.battery') | selectattr('deviceId', 'equalto', 'zigbee-ZIGBEE_ID') | first).properties.level.value }}"
        device_class: battery
        state_class: measurement
        unit_of_measurement: "%"
        icon: mdi:battery
#### TRV Signal Strength
      - name: "ViCare Signal Room 1"
        unique_id: vicare_custom_signal_room_1
        value_template: "{{ (value_json.data | selectattr('feature', 'match', 'device.zigbee.lqi') | selectattr('deviceId', 'equalto', 'zigbee-ZIGBEE_ID') | first).properties.strength.value }}"
        device_class: signal_strength
        state_class: measurement
        unit_of_measurement: "dBm"
        icon: mdi:zigbee
## Boiler Binary Sensors
    binary_sensor:
### Cloud Connection Status
      - name: "Viessmann Status"
        unique_id: vicare_custom_vitoconnect_status
        value_template: >-
          {% set mapper =  {
              'online' : true,
              'offline' : false } %}
          {% set state = ((value_json.data | selectattr('feature', 'match', 'gateway.devices') | first).properties.devices.value | selectattr('id', 'equalto', 'gateway') | first).status %}
          {{ mapper[state] if state in mapper else 'Unknown' }}
        icon: mdi:wan
        device_class: connectivity
### Burner Status
      - name: "Viessmann Burner"
        unique_id: vicare_custom_viessmann_burner
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.burners.0') | first).properties.active.value }}"
        icon: mdi:gas-burner
        device_class: running
### Circulation Pump Status
      - name: "Viessmann Heizkreispumpe"
        unique_id: vicare_custom_viessmann_heizkreispumpe
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.circuits.0.circulation.pump') | first).properties.status.value }}"
        icon: mdi:pump
        device_class: running
### Frost Protection Status
      - name: "Viessmann Heizkreis Frostschutz"
        unique_id: vicare_custom_viessmann_heizkreis_frostschutz
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.circuits.0.frostprotection') | first).properties.status.value }}"
        icon: mdi:snowflake
### Hot Water Charge
      - name: "Viessmann WW Aufladen"
        unique_id: vicare_custom_viessmann_ww_aufladen
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.dhw.charging') | first).properties.active.value }}"
        device_class: running
### Hot Water Circulation Pump
      - name: "Viessmann WW Zirkulationspumpe"
        unique_id: vicare_custom_viessmann_ww_zirkulationspumpe
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.dhw.pumps.circulation') | first).properties.status.value }}"
        icon: mdi:pump
        device_class: running
### Hot Water Pump
      - name: "Viessmann WW Pumpe"
        unique_id: vicare_custom_viessmann_ww_pumpe
        value_template: "{{ (value_json.data | selectattr('feature', 'equalto', 'heating.dhw.pumps.primary') | first).properties.status.value }}"
        icon: mdi:pump
        device_class: running


```
In order to capture your room numbers and zigbee IDs you have two options. Either use the above mentioned REST command and scan through the response. Or you can technically also just use the ViCare App to capture this. For the latter you can go into settings the app. The Serial Number of the TRVs and Climate Sensors is identical to the Zigbee ID. Just remove all "-" (dashes) and lower case the letters. For the room numbers it will be a little trial and error. Unfortunately, the display order in the app is not the same as the numbering from the API. So, if you have 3 rooms, you will have the room numbers 0, 1 and 2. You could create your sensors, picking one of the available rooms and just change the room numbers in your configuration after you figured out what room number applies to which devices. Using the cloud request results you would search for your room names and find the correlating "rooms.X" value.

## Step 5: Creating automation actions.
Here is an example of changing the target temperature of a room when a window sensor triggers:

Set target temperature to 8° for 30 minutes (default is 2 hours, I wanted it to go back to normal after 30 minutes in case HA missed the window closing again) when window opens for 5 seconds

Change ROOM_NUMBER to match your configuration.
```
alias: ViCare - Window Open Room 1
description: ""
triggers:
  - type: opened
    device_id: WINDOW_SENSOR_DEVICE
    entity_id: WINDOW_OPENED_SENSOR
    domain: binary_sensor
    trigger: device
    for:
      hours: 0
      minutes: 0
      seconds: 5
conditions: []
actions:
  - alias: Activate Manual Quickmode
    action: rest_command.vicare_api_post
    data:
      url: >-
        https://api.viessmann.com/iot/v2/features/installations/{{
        state_attr('sensor.vicare_tokens','install_id') | int(0) }}/gateways/{{
        state_attr('sensor.vicare_tokens','gateway_id') | int(0)
        }}/devices/RoomControl-1/features/rooms.ROOM_NUMBER.quickmodes.manual/commands/activate
      payload: "{\"duration\":30,\"temperature\":8}"
    response_variable: response
mode: single

```
Remove the manual mode setting from the window open automation when the window is closed again:
```
alias: ViCare - Window Open Room 1
description: ""
triggers:
  - type: not_opened
    device_id: WINDOW_SENSOR_DEVICE
    entity_id: WINDOW_OPENED_SENSOR
    domain: binary_sensor
    trigger: device
    for:
      hours: 0
      minutes: 0
      seconds: 5
conditions: []
actions:
  - alias: Deactivate Manual Quickmode
    action: rest_command.vicare_api_post
    data:
      url: >-
        https://api.viessmann.com/iot/v2/features/installations/{{
        state_attr('sensor.vicare_tokens','install_id') | int(0) }}/gateways/{{
        state_attr('sensor.vicare_tokens','gateway_id') | int(0)
        }}/devices/RoomControl-1/features/rooms.ROOM_NUMBER.quickmodes.manual/commands/deactivate
      payload: "{}"
    response_variable: response
mode: single

```
