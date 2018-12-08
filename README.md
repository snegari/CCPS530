
<!-- This is for my CCPS530 Project -->

# Web Systems Development
---
## Online Demo: [negari.org](www.negari.org)
## Remote Monitoring of Physical Signals
## Professor: Dr. G. Tofighi
## Student: Shahram Negari
## Course: CCPS530
## Ryerson University
----

# Introduction
Measuring and gathering *electircal signals* from, and *environmental parameters* at each and every nodes of the electric grid is a prerquisite for ensuring safe and reliable operation of the grid. There are thousands of electrical substations and generating stations connected to the transmission grid, and to keep this giant system up and running, grid controllers need to collect and monitor network parameters from several hundred kilometers away. 

In this project, I intend to measure one parameter and use the internet to send the information to a webpage accessible from everywhere. I've used a microcontroller and temperature sensor to measure the temperature and transmit the info using TCP/IP protocol. 

# Hardware
A readily available and low-cost microcontrller (ESP 8266) and a temperature sensor MCP 9808 are employed for this project. 

## ESP 8266, Microcontroller

ESP 8266 microcontroller benefits from a Tensilica L106, a 32-bit RISC processor. It is a very low-power consumption microcontroller with 160 MHz clock speed. This microcontroller is also equipped with a Wi-Fi stack allowing it to communicate via wireless network. 
Adafruit sells this microcontroller under the brand name Feather Huzzah.

## MCP 9808, Temperature Sensor
MCP 9808 is a digital temperature sensor with a wide measurement range of -40 C to +125 C, with a resolution of 0.0625 C and an accuracy of +/- 0.25 C. It uses I2C standard for transferring the data. 
The input DC voltage is 2.7 V to 5.5 V, and operating current is 200 micro A. 

![First Photo](https://github.com/snegari/CCPS530/blob/master/img/530_1.jpg)

# Firmware 
Python and Flask are used for this project. Therefore, I needed to wipe out the flash on the microcontroller (it was set up for Arduino IDE) and install micropython on it. The steps as explained by Peter Kazarinoff [Python for Undergraduate Engineers](pythonforundergradengineers.com) are as follows: 

> * Install the Anaconda distribution of Python
> * Create a new conda environment and pip install esptool
> * Download the latest Micropython .bin firmware file
> * Install the SiLabs driver for the Adafruit Feather Huzzah ESP8266
> * Connect the Adafruit Feather Huzzah ESP8266 board to the laptop using a microUSB cable
> * Determine which serial port the Feather Huzzah is connected to
> * Run the esptool to upload the .bin firmware file to the Feather Huzzah
> * Download and install Putty, a serial monitor
> * Use Putty to connect to the Feather Huzzah and run commands in the Micropython REPL

## Creating cond environment and installing esptool.py

```Linux
conda create -n micropython python=3.6
conda activate micropython
(micropython) pip install esptool
(micropython) conda list
(micropython) cd Documents
(micropython) mkdir micropthon
(micropython) cd micropython
```
## Downloading micropython firmware .bin file
Firmware for ESP8266 microcontroller board can be downloaded from [Micropython](micropython.org/download/#esp8266).

However, to transfer the file we also need a USB to UART Bridge as well. Silicon Labs [Silabs](silabs.com/products/development-tools/software/usb-to-uart-bridge-vcp-drivers) shall be downloaded to the laptop and installed. Then, we can go to the device manager and see the microcontroller connected to one of the COM ports. 

## Erasing microcontroller flash
Now that the microcontroller is detectable on the COM port, we can use esptool to erase the flash and install micropython: 

```
cd Documents
cd micropython
pwd
Documents/micropython
dir
conda activate micropython
(micropython) esptool --help

(micropython) esptool --port COM5 erase_flash

(micropython) esptool --port COM5 --baud 460800 write_flash --flash_size=detect 0 esp8266-20171101-v1.9.3.bin

```
Now, we can use PuTTY to connect to the microcontroller via USB cable and directly code or use the micropython interpreter. 

![Second Photo](https://github.com/snegari/CCPS530/blob/master/img/530_2.jpg)

## Transferring files to microcontroller
> 1. Install __ampy__ with __pip__
> 2. Write Python code in .py files
> 3. Upload the .py files to the board with ampy
> 4. Unplug and power up the microcontroller

## Installing __ampy__ with __pip__
```Linux
$ conda activate micropython
(micropython) $ pip install ampy-adafruit
(micropython) $ ampy --help

```
## Python code for the sensor & Microcontroller

```Python
# MCP9808.py

# Functions for the  MCP9808 temperature sensor
# https://learn.adafruit.com/micropython-hardware-i2c-devices/i2c-master

def readtemp():
    import machine
    i2c = machine.I2C(scl=machine.Pin(5), sda=machine.Pin(4))
    byte_data = bytearray(2)
    i2c.readfrom_mem_into(24, 5, byte_data)
    value = byte_data[0] << 8 | byte_data[1]
    temp = (value & 0xFFF) / 16.0
    if value & 0x1000:
        temp -= 256.0
    return temp
```

```Python
#wifitools.py

# Wifi connection and ThingSpeak.com post functions for an ESP8266 board running Micropython
#https://docs.micropython.org/en/v1.8.6/esp8266/esp8266/tutorial/network_basics.html

def connect(SSID,password):
    import network
    sta_if = network.WLAN(network.STA_IF)
    if not sta_if.isconnected():
        print('connecting to network...')
        sta_if.active(True)
        sta_if.connect(SSID, password)
        while not sta_if.isconnected():
            pass
    print('network config:', sta_if.ifconfig())

#https://docs.micropython.org/en/v1.8.6/esp8266/esp8266/tutorial/network_tcp.html
def http_get(url):
    import socket
    _, _, host, path = url.split('/', 3)
    addr = socket.getaddrinfo(host, 80)[0][-1]
    s = socket.socket()
    s.connect(addr)
    s.send(bytes('GET /%s HTTP/1.0\r\nHost: %s\r\n\r\n' % (path, host), 'utf8'))
    while True:
        data = s.recv(100)
        if data:
            print(str(data, 'utf8'), end='')
        else:
            break

def thingspeak_post(API_key,data):
    if not isinstance(data, str):
        data = str(data)
    if not isintance(API_key, str):
        API_key = str(API_key)
    base_url = 'https://api.thingspeak.com/update'
    API_key = '?api_key=' + API_key
    field = '&field1='
    url = base_url + API_key + field + data
    http_get(url)
```
```Python
# main.py
# Adafruit Feather Huzzah ESP8266 WiFi Weather Station

import wifitools
import MCP9808
import time
import config

api_key = config.API_KEY
ssid = config.SSID
password = config.WIFI_PASSWORD

wifitools.connect(ssid,password)
time.sleep(5)

for i in range(8*60):
    data = MCP9808.readtemp()
    wifitools.thingspeak_post(api_key,data)
    time.sleep(60)
```
## Uploading .py files to the board with __ampy__
```Linux
$ conda activate micropython
(micropython)$ ampy --port COM4 put MCP9808.py
(micropython)$ ampy --port COM4 put wifitools.py
(micropython)$ ampy --port COM4 put main.py
(micropython)$ ampy --port COM4 put config.py
(micropython)$ ampy --port COM4 ls
boot.py
wifitools.py
MCP9808.py
config.py
main.py
```

# Setting Up the Server and Codes
I purchased a paid hosting service at Digital Ocean; called a *droplet*. Created a non-root sudo user, and set-up an SSH key to communicate with the server by PuTTY. I also registered the domian name *negari.org* and linked the domain name to the server IP address. 
Following two references were consulted for the project: 
* Digital Ocean Community
* Python for Undergraduate Engineers

The code follows: 

```linux
adduser shahram
usermode -aG sudo shahram
ufw enable

// copying the SSH keys to the non-root sudo user

rsync --archive --chown=shahram:shahram ~/.ssh /home/shahram

// to check the status
sudo -l
User shahram may run the following commands on flask-app-server: 
(ALL : ALL) ALL

// Building the Flask App

sudo apt-get update
sudo apt-get upgrade
sudo apt-get install python3-pip
sudo apt-get install python3-dev
sudo apt-get install python3-setuptools
sudo apt-get install python3-venv
sudo apt-get install build-essential libssl-dev libffi-dev 

cd ~
mkdir flaskapp
cd flaskapp
python3.6 -m venv flaskappenv
source flaskappenv/bin/activate

(flaskappenv)$ pip install wheel
(flaskappenv)$ pip install flask
(flaskappenv)$ pip install uwsgi
(flaskappenv)$ pip install requests

(flaskappenv)$ pwd
# ~/flaskapp
(flaskappenv)$ nano flaskapp.py

```
## Building the Flask APP

```Python

# flaskapp.py

from flask import Flask
app = Flask(__name__)

@app.route("/")
def index():
    return "<h1>The temperature is 25 C at Ryerson University</h1>"

if __name__ == "__main__":
    app.run(host='0.0.0.0')

```

## Testing the Preliminary Flask App

To ensure that the flask app works properly, it was necessary to open port # 5000 on the firewall: 

```Linux
(flaskappenv)$ sudo ufw allow 5000
(flaskappenv)$ python flaskapp.py
```
Then typing my IP address followed by :5000, i.e. 104.248.187.84:5000 I could read: "The temperature is 25 C at Ryerson University".

## Responding to GET requests
The flask app needs to respond to the GET requests sent over the internet. Therefore, I had to use uWSGI and NGINX to make the server ready. That is, the requests will arrive to NGINX, then passed on to uWSGI, which will relay them to the flask app. Configuration was as follows: 

```Linux
(flaskappenv)$ pwd
# ~/flaskapp
(flaskappenv)$ nano wsgi.py
```
```Python
# wsgi.py

from flaskapp import app

if __name__ == "__main__":
    app.run()
```
```Linux
(flaskappenv)$ uwsgi --socket 0.0.0.0:5000 --protocol=http -w wsgi:app
(flaskappenv)$ deactivate
$ pwd
# ~/flaskapp
$ nano flaskapp.ini
[uwsgi]
module = wsgi:app

master = true
processes = 5

socket = flaskapp.sock
chmod-socket = 660
vacuum = true

die-on-term = true
$ sudo nano /etc/systemd/system/flaskapp.service

[Unit]
Description=uWSGI instance to serve flaskapp
After=network.target

[Service]
User=peter
Group=www-data
WorkingDirectory=/home/peter/flaskapp
Environment="PATH=/home/peter/flaskapp/flaskappenv/bin"
ExecStart=/home/peter/flaskapp/flaskappenv/bin/uwsgi --ini flaskapp.ini

[Install]
WantedBy=multi-user.target

// testing the systemctl
$ sudo systemctl daemon-reload
$ sudo systemctl start flaskapp
$ sudo systemctl status flaskapp

flaskapp.service - uWSGI instance to serve flaskapp
   Loaded: loaded (/etc/systemd/system/flaskapp.service; disabled; vendor preset
   Active: active (running) since Wed 2018-11-23 23:49:16 UTC; 11s ago

// Installing NGINX

$ sudo apt-get install nginx
$ sudo nano /etc/nginx/sites-available/flaskapp
server {
    listen 80;
    server_name negari.org wwww.negari.org;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/peter/flaskapp/flaskapp.sock;
    }
}

$ sudo ln -s /etc/nginx/sites-available/flaskapp /etc/nginx/sites-enabled
$ sudo systemctl restart nginx
$ sudo systemctl status nginx
#ctrl-c to exit

$ sudo ufw delete allow 5000
$ sudo ufw allow 'Nginx Full'

// Creating and index page and updating flask app
$ cd ~/flaskapp
$ mkdir templates
$ cd templates
$ nano index.html
$ nano ~/flaskapp/flaskapp.py
```

```Python
# flaskapp.py

from flask import Flask, render_template
app = Flask(__name__)

@app.route("/")
def index():
    return render_template("index.html")

if __name__ == "__main__":
    app.run(host='0.0.0.0')

```

Now need to restart the running app:

```Linux
$ sudo systemctl stop flaskapp
$ sudo systemctl start flaskapp
$ sudo systemctl status flaskapp
# [ctrl-c] to exit

```
And now we can modify the flask app to get the temperature info from thingspeak.com

```Python
# flaskapp.py

from flask import Flask, render_template
import requests
app = Flask(__name__)

@app.route("/")
def index():
    r = requests.get('https://api.thingspeak.com/channels/643937/fields/1/last.txt')
    temp_c_in = r.text
    temp_f = str(round(((9.0 / 5.0) * float(temp_c_in) + 32), 1)) + ' F'
    return render_template("index.html", temp=temp_f)

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```
Finally I needed to restart the app so that latest changes are implemented. 


# More About Project & More About Me

Two additional webpages are created using templates from [W3 Schools](w3schools.com), which are fully responsive. The first page explains more about the project, and the second is a personal resume.  


# Attachments
#Project Photos: 

![First Photo](https://github.com/snegari/CCPS530/blob/master/img/530_1.jpg)
![Second Photo](https://github.com/snegari/CCPS530/blob/master/img/530_2.jpg)

# Video Clip



