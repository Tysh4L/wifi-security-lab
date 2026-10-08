# Wi-Fi Security Lab

## Overview

This project documents a controlled Wi-Fi security lab performed on my own wireless network and personal tablet.

The main objective was to explore basic 802.11 wireless monitoring concepts using the Aircrack-ng suite, including monitor mode, access point discovery, station identification, signal observation, and directed deauthentication testing.

During the lab, I also observed how MAC address randomization can affect client identification and how the behavior changes when the device uses its hardware MAC address.

> All testing was performed only on devices and networks under my control.

## Lab Environment

- Operating System: Garuda Linux
- Wireless toolkit: Aircrack-ng
- Wireless interface: Intel Wireless-AC adapter using the `iwlwifi` driver
- Test network: Personal Wi-Fi access point
- Test client: Personal tablet
- Wireless security: WPA2-PSK
- Test band/channel: 2.4 GHz, channel 10

## 1. Wireless Interface Preparation

Before starting the wireless analysis, I verified the current state of the Wi-Fi interface.

The interface was operating in **Managed mode**, which is the normal mode used to connect to wireless networks.

```
iwconfig
```

![Wireless interface in Managed mode](screenshots/01-managed-mode.png)

### Checking wireless radio status

I checked whether the wireless interface was blocked by software or hardware before enabling monitor mode.

```
sudo rfkill list
```

![RFKill status](screenshots/02-rfkill-status.png)
### Stopping interfering processes

Some background services can interfere with monitor mode by attempting to control the wireless interface.

I used Aircrack-ng to stop those processes before enabling monitor mode.

```
sudo airmon-ng check kill
```

![Stopping processes that may interfere with monitor mode](screenshots/03-stop-interfering-processes.png)
## 2. Enabling Monitor Mode

To capture nearby 802.11 traffic, the wireless interface must operate in **Monitor mode** instead of Managed mode.

I enabled monitor mode using:

```
sudo airmon-ng start wlp4s0
```

The interface changed from: `wlp4s0` -> `wlp4s0mon`

![Enabling Monitor mode with airmon-ng](screenshots/04-enable-monitor-mode.png)

I verified the new interface mode using:

```
iwconfig
```

![Wireless interface in Monitor mode](screenshots/05-monitor-mode-confirmation.png)
## 3. Access Point Discovery

With the wireless interface in Monitor mode, I used `airodump-ng` to scan for nearby wireless networks:

```
sudo airodump-ng wlp4s0mon
```

The scan displayed nearby access points and information such as:

- `BSSID` — MAC address of the access point
- `PWR` — received signal strength
- `CH` — wireless channel
- `ENC` — encryption type
- `CIPHER` — encryption algorithm
- `AUTH` — authentication method
- `ESSID` — network name

I let the scan run for approximately 30 seconds, which was enough to identify the access point used for this lab.

![Nearby wireless network discovery with airodump-ng](screenshots/06-network-discovery.png)
### Monitoring the test access point

Once the test network was identified, I restricted the capture to its channel and BSSID.

```
sudo airodump-ng -c 10 --bssid <AP_BSSID> wlp4s0mon
```

In this command:

- -c 10 specifies the channel used by the access point.
- --bssid <AP_BSSID> filters the capture to the selected access point.
- wlp4s0mon is the wireless interface operating in Monitor mode.

This made it easier to observe only the traffic and stations associated with the test network.

![Selected access point and observed stations](screenshots/07-target-ap-and-stations.png)

The `STATION` section shows wireless client devices observed communicating with the selected access point.

At this point, the tablet used as the test client was connected to the Wi-Fi network and actively using the connection.

![Test tablet connected to the wireless network](screenshots/08-tablet-connected.png)
## 4. Directed Deauthentication Test

After identifying the test tablet in the `STATION` list, I performed a directed deauthentication test against that client.

The test was performed only on my own access point and personal tablet.

```
sudo aireplay-ng -0 0 -a <AP_BSSID> -c <CLIENT_MAC> wlp4s0mon
```

In this command:

- -0 selects deauthentication mode.
- 0 sends deauthentication frames continuously until the command is stopped manually.
- -a <AP_BSSID> specifies the BSSID of the access point.
- -c <CLIENT_MAC> specifies the target client station.
- wlp4s0mon is the wireless interface in Monitor mode.

![Directed deauthentication command](screenshots/09-deauth-command.png)

### Observing the deauthentication traffic

While `aireplay-ng` was running, the terminal repeatedly displayed output similar to:

```
Sending 64 directed DeAuth (code 7). STMAC: [CLIENT_MAC] [17|64 ACKs]
```

The output provides information about the transmitted deauthentication frames:

- Directed DeAuth indicates that the frames are being sent to a specific client station.
- code 7 is the 802.11 reason code used in the generated deauthentication frames.
- STMAC identifies the target station MAC address.
- The ACK values indicate acknowledgements observed from the client and access point during transmission.

![Deauthentication traffic in progress](screenshots/10-deauth-in-progress.png)

At the same time, I monitored the selected access point using `airodump-ng`.

The `Lost` value associated with the target station increased while the deauthentication test was running, showing that communication between the client and access point was being disrupted.

![Increase in lost frames for the target station](screenshots/11-increased-lost-frames.png)

### Client disconnection

After the test had been running for a short period, the tablet lost its connection to the monitored 2.4 GHz network.

Because another saved Wi-Fi network was available, the tablet automatically connected to the 5 GHz network instead.

![Tablet automatically connected to another Wi-Fi network](screenshots/12-client-disconnected-from-target-network.png)

I then attempted to reconnect the tablet to the original 2.4 GHz network.

![Tablet attempting to reconnect to the test network](screenshots/13-client-reconnection-attempt.png)
## 5. MAC Address Randomization Observation

After reconnecting the tablet to the original Wi-Fi network, the device appeared in `airodump-ng` with a different client MAC address.

This happened because the tablet was configured to use a **randomized MAC address** for that Wi-Fi network.

Instead of reusing the same client MAC address that had been targeted by `aireplay-ng`, the tablet appeared as a new station with a different MAC address.

As a result, the deauthentication command that was still running continued targeting the previous client address, while the tablet was now communicating with the access point using a new randomized MAC address.

![Randomized MAC address enabled on the test tablet](screenshots/14-randomized-mac-enabled.png)

The change was also visible from the monitoring system, where the tablet appeared as a new `STATION` with a different MAC address.

![Tablet appearing as a new station with a randomized MAC address](screenshots/15-new-randomized-station.png)

### Observation

This demonstrated an important limitation when identifying wireless clients only by their MAC address.

A device using MAC address randomization may appear as a different station after reconnecting, even though it is still the same physical device.

In this test, the original deauthentication process continued targeting the old station address, while the tablet had already reconnected using a different randomized MAC address.

## 6. Static MAC Address Comparison

To compare the behavior, I changed the tablet Wi-Fi privacy setting from a randomized MAC address to the device MAC address.

This allowed the tablet to connect using its hardware MAC address instead of a temporary randomized address.

![Changing the tablet from Randomized MAC to Tablet MAC](screenshots/16-switch-to-device-mac.png)

Once the tablet reconnected, `airodump-ng` showed a new station using the device MAC address.

![Tablet detected using its device MAC address](screenshots/17-device-mac-station.png)

I repeated the directed deauthentication test using the tablet's device MAC address:

```
sudo aireplay-ng -0 0 -a <AP_BSSID> -c <DEVICE_MAC> wlp4s0mon
```

While the test was running, the Lost value for the station increased and the tablet lost connectivity to the Wi-Fi network.

![Directed deauthentication test against the tablet device MAC](screenshots/18-deauth-device-mac.png)

The tablet was unable to reconnect successfully while the continuous deauthentication test was still running.

![Wi-Fi connection failure while the test was running](screenshots/19-connection-failed.png)

## 7. Ending the Test and Restoring Connectivity

After stopping the continuous deauthentication test, the tablet was able to connect to the Wi-Fi network again.

This confirmed that the connectivity problem was caused by the active deauthentication test rather than by a permanent configuration issue.

![Tablet reconnected after stopping the deauthentication test](screenshots/20-connectivity-restored.png)

For privacy, I changed the tablet back to using a randomized MAC address after completing the test.

![Randomized MAC address restored on the tablet](screenshots/21-randomized-mac-restored.png)
### Disabling Monitor Mode

After completing the wireless test, I disabled Monitor mode:

```
sudo airmon-ng stop wlp4s0mon
```

### Disabling Monitor Mode

After completing the wireless test, I disabled Monitor mode:

```
sudo airmon-ng stop wlp4s0mon
```

This removed the monitor interface and restored the wireless adapter to its normal operating state.

![Disabling Monitor mode](screenshots/22-disable-monitor-mode.png)
### Restoring NetworkManager

Finally, I restarted NetworkManager:

```
sudo systemctl restart NetworkManager
```

I then verified the interface again with:

```
iwconfig
```

The wireless interface was back in **Managed mode**, confirming that the system had returned to its normal Wi-Fi configuration.

![Wireless interface restored to Managed mode](screenshots/23-managed-mode-restored.png)

## 8. Key Findings

This lab helped me understand several practical aspects of 802.11 wireless security:

- Monitor mode allows a wireless adapter to observe nearby 802.11 traffic without being connected to an access point.
- `airodump-ng` can be used to identify access points, channels, signal strength, and observed client stations.
- A directed deauthentication test can temporarily disrupt communication between an access point and a specific client.
- The `Lost` value in `airodump-ng` can help indicate communication problems while the test is running.
- MAC address randomization can make the same physical device appear as a different station after reconnecting.
- A deauthentication process targeting an old randomized MAC address will not automatically follow the device when it reconnects using a different MAC address.
- Using the device MAC allowed me to observe the difference between a changing client identifier and a stable one.

## Disclaimer

This project was performed exclusively in a controlled environment using my own wireless network and personal devices.

The purpose of this lab was educational: to understand wireless monitoring, 802.11 client behavior, MAC address randomization, and deauthentication behavior using the Aircrack-ng suite.

No third-party networks or devices were intentionally targeted.