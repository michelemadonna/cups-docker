# CUPS-docker

Run a CUPS print server on a remote machine to share USB printers over WiFi. Built primarily to use with Raspberry Pis as a headless server, but there is no reason this wouldn't work on `amd64` machines. Tested and confirmed working on a Raspberry Pi 3B+ (`arm/v7`) and Raspberry Pi 4 (`arm64/v8`).

## Usage
Clone this Git repo
```sh
git clone https://github.com/michelemadonna/cups-docker.git .
docker build -t cups .
```
 
Quick start with default parameters
```sh
docker run -d -p 631:631 --device /dev/bus/usb -v /var/run/dbus:/var/run/dbus --net=host --name cups cups
```

Customizing your container
```sh
docker run -d --name cups \
    --restart unless-stopped \
    --net=host \
    -p 631:631 \
    --device /dev/bus/usb \
    -e CUPSADMIN=cupsadmin \
    -e CUPSPASSWORD=password \
    -e TZ="Europe/Rome" \
    -v /var/run/dbus:/var/run/dbus \
    cups
```
> Note: :P make sure you use valid TZ string. Also changing the default username and password is highly recommended.

### Parameters and defaults
- `port` -> default cups network port `631:631`. Change not recommended unless you know what you're doing
- `device` -> used to give docker access to USB printer. Default passes the whole USB bus `/dev/bus/usb`, in case you change the USB port on your device later. change to specific USB port if it will always be fixed, for eg. `/dev/bus/usb/001/005`.

#### Optional parameters
- `name` -> whatever you want to call your docker image. using `cups` in the example above.

Environment variables that can be changed to suit your needs, use the `-e` tag
| # | Parameter    | Default            | Type   | Description                       |
| - | ------------ | ------------------ | ------ | --------------------------------- |
| 1 | TZ           | "America/New_York" | string | Time zone of your server          |
| 2 | CUPSADMIN    | admin              | string | Name of the admin user for server |
| 3 | CUPSPASSWORD | password           | string | Password for server admin         |

### docker-compose
```yaml
version: "3"
services:
    cups:
        build: .
        container_name: cups
        restart: unless-stopped
        network_mode: "host"
        ports:
            - "631:631"
        devices:
            - /dev/bus/usb:/dev/bus/usb
        environment:
            - CUPSADMIN=${CUPSADMIN}
            - CUPSPASSWORD=$(CUPSPASSWORD)
            - TZ=${TZ}
        volumes:
            - /var/run/dbus:/var/run/dbus
```

## Server Administration
You should now be able to access CUPS admin server using the IP address of your headless computer/server http://192.168.xxx.xxx:631, or whatever. If your server has avahi-daemon/mdns running you can use the hostname, http://yourserverhostname.local:631. (IP and hostname will vary, these are just examples)

If you are running this on your PC, i.e. not on a headless server, you should be able to log in on http://localhost:631

## Ensuring Printer Reconnection After Power Off and On
In the event that the printer is turned off and then turned back on, the Docker container will no longer be able to connect to it. It is necessary to create a udev rule that executes the hp-firmware command for HP printers (check your printer vendor/model for an utility for the same task) within the container to reconnect the printer without having to restart the container itself.

1. **Connect your printer** to your computer.
2. **Open a terminal in the host (not in the dicker!!!!)** and run the following command to list all connected USB devices:

    ```bash
    lsusb
    ```

    This will give you a list of connected USB devices along with their vendor and product IDs.

3. **Find your printer** in the list and note down its `idVendor` and `idProduct`.

4. **Run the following command** to get detailed information about your printer, including its serial number:

    ```bash
    udevadm info --query=all --name=/dev/usb/lp0
    ```

    Replace `/dev/usb/lp0` with the appropriate device path for your printer. You can find the correct device path by running:

    ```bash
    dmesg | grep -i usb
    ```

    Look for the line that corresponds to your printer.

5. **Look for the attributes** `ID_VENDOR_ID`, `ID_MODEL_ID`, and `ID_SERIAL` in the output of the `udevadm` command. These correspond to the `idVendor`, `idProduct`, and serial number, respectively.

    Here's an example of what the output might look like:
    
    ```bash
    E: ID_VENDOR_ID=03f0
    E: ID_MODEL_ID=002a
    E: ID_SERIAL=CN12345678
    ```

    Now you can use these values to create the udev rule.

6. **Create the file hprinter.rules** in /etc/udev/rules.d with this content :
    ```
    ACTION=="add", ATTRS{idVendor}=="03f0", ATTRS{idProduct}=="3e17", ATTRS{serial}=="AC2AP8E", RUN+="/bin/sh -c '/path/to/the/script/restart-printer.sh'"
    ```

7. **Create the file restart-printer.sh** in /path/to/the/script/ with this content :
    ```bash
    #!/bin/bash
    
    # Check if the Docker container "cups" is active
    container_status=$(docker inspect -f '{{.State.Status}}' cups)
    
    if [ "$container_status" == "running" ]; then
        echo "Docker container 'cups' is active. Executing 'hp-firmware' inside the container."
        docker exec cups hp-firmware
    else
        echo "Docker container 'cups' is not active. Cannot execute 'hp-firmware'."
    fi
    ```

8. **restart the udev**
    ```bash
    udevadm control --reload-rules && systemctl restart udev
    ```

## HP Printers with auxliary firmware
Some HP printers like the HP LaserJet P1006 require auxiliary non-open source firmware. 
The hp-setup utility is unable to download such files due to a connection error. However, it is possible to download the necessary files from the site https://www.openprinting.org/download/printdriver/auxfiles/HP/plugins/ by selecting the file with the corresponding version of hplip that we have installed.

Directly from the container, execute: 
```
wget https://www.openprinting.org/download/printdriver/auxfiles/HP/plugins/hplip-3.22.10-plugin.run && sh hplip-3.22.10-plugin.run --accept --quiet -- -i 
```

Once the firmware is installed, it will be necessary to configure the printer with the hp-setup utility.
```
hp-setup -i
```
Let me know if you need any further assistance!

## Thanks
Based on the work done by **RagingTiger**: [https://github.com/RagingTiger/cups-airprint](https://github.com/RagingTiger/cups-airprint) and **anujdatar**: [https://github.com/anujdatar/cups-docker](https://github.com/anujdatar/cups-docker)
