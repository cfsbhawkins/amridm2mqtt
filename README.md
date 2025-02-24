# AMRIDM2MQTT: Send AMR/ERT Power Meter Data Over MQTT

##### Copyright (c) 2018 Ben Johnson. Distributed under MIT License.

Using an [inexpensive rtl-sdr dongle](https://www.amazon.com/s/ref=nb_sb_noss?field-keywords=RTL2832U), it's possible to listen for signals from ERT compatible smart meters using rtlamr. This script runs as a daemon, launches rtl_tcp and rtlamr, and parses the output from rtlamr. If this matches your meter, it will push the data into MQTT for consumption by Home Assistant, OpenHAB, or custom scripts.

TODO: Video for Home Assistant


## Docker

If you use Docker and would rather launch this under a container see <README.Docker.md>.

## Requirements

Tested under Raspbian GNU/Linux 9.3 (stretch)

### rtl-sdr package

Install RTL-SDR package

`sudo apt-get install rtl-sdr`

Set permissions on rtl-sdr device

/etc/udev/rules.d/rtl-sdr.rules

`SUBSYSTEMS=="usb", ATTRS{idVendor}=="0bda", ATTRS{idProduct}=="2838", MODE:="0666"`

Prevent tv tuner drivers from using rtl-sdr device

/etc/modprobe.d/rtl-sdr.conf

`blacklist dvb_usb_rtl28xxu`

### git

`sudo apt-get install git`

### pip3 and paho-mqtt

Install pip for python 3

`sudo apt-get install python3-pip`

Install paho-mqtt package for python3

`sudo pip3 install paho-mqtt`

### golang & rtlamr

Install Go programming language & set gopath

`sudo apt-get install golang`

https://github.com/golang/go/wiki/SettingGOPATH

If only running go to get rtlamr, just set environment temporarily with the following command

`export GOPATH=$HOME/go`


Install rtlamr https://github.com/bemasher/rtlamr

`go get github.com/bemasher/rtlamr`

To make things convenient, I'm copying rtlamr to /usr/local/bin

`sudo cp ~/go/bin/rtlamr /usr/local/bin/rtlamr`

## Install

### Clone Repo
Clone repo into opt

`cd /opt`

`sudo git clone https://github.com/ragingcomputer/amridm2mqtt.git`

### Configure

Copy template to settings.py

`cd /opt/amridm2mqtt`

`sudo cp settings_template.py settings.py`

Edit file and replace with appropriate values for your configuration

`sudo nano /opt/amridm2mqtt/settings.py`

### TODO

TLS setup is not performed, so you can only use the non-secure TCP port which is 1883 by default.

### Install Service and Start

Copy armidm2mqtt service configuration into systemd config

`sudo cp /opt/amridm2mqtt/amridm2mqtt.systemd.service /etc/systemd/system/amridm2mqtt.service`

Refresh systemd configuration

`sudo systemctl daemon-reload`

Start amridm2mqtt service

`sudo service amridm2mqtt start`

Set amridm2mqtt to run on startup

`sudo systemctl enable amridm2mqtt.service`

### Configure Home Assistant

To use these values in Home Assistant,

```yaml
sensor:
  - platform: mqtt
    state_topic: "amr/reading/SCM/4/12345678/message"
    name: "Electric Meter"
    unique_id: electric_meter_01
    unit_of_measurement: kWh
    device_class: energy
    state_class: measurement
    availability_topic: amr/status/availability
    last_reset_topic: amr/status/last_reset
    value_template: "{{ value_json.Message.Consumption }}"
    json_attributes_template: "{{ value_json.Message | tojson }}"
    json_attributes_topic: "amr/reading/SCM/4/12345678/message"
```

Note that we are publishing status information to the amr/status topic:

**amr/status/availability**: `online`|`offline` when the service starts or stops, respectively.\
**amr/status/last_reset**: `1970-01-01T00:00:00+00:00` see [last_reset_topic](https://www.home-assistant.io/integrations/sensor.mqtt/#last_reset_topic) for more information.

## Testing

Assuming you're using mosquitto as the server, and your meter's sends SCM messages with id 12345678, you can watch for events using the command:

`mosquitto_sub -t "amr/reading/SCM/4/12345678/message"`

Or if you've password protected mosquitto

`mosquitto_sub -t "amr/reading/SCM/4/12345678/message" -u <user_name> -P <password>`

If all else fails, you can listen to the base topic to see what's actually getting posted

`mosquitto_sub -t "amr/#" -u <user_name> -P <password>`

Most electric meters are going to be type 4 or 7. For a list of meter types and ERT types, see: https://github.com/bemasher/rtlamr/blob/master/meters.csv
