# Multiple IPv6 Prefix Delegation over AT&amp;T Residential Gateway for OPNsense

#### Known Compatible Versions
* OPNsense - 21.x, 22.x, 23.7.X

#### Known working residential gateways (RGs)
* Pace 5268AC
* Arris NVG599
* Arris BGW210-700
* Motorola NVG589
* HUMAX BGW320-500

#### Known Caveats
* [Google devices do not support DHCPv6](https://issuetracker.google.com/issues/36949085)
	* This includes any phone running Android, TV's running Android TV or Google TV, or any Nest product (mini, hub, doorbell, etc). The solution outlined in this readme relies upon DHCPv6. Google devices will not pull a routable GUA (Global Unicast Address), but instead attempt to communicate over the local network via ULAs (Unique Local Address).

All credit goes to the user **ttmcmurry** from the [Netgate Forum](https://forum.netgate.com/) for his insight within the [thread](https://forum.netgate.com/topic/153288/multiple-ipv6-prefix-delegation-over-at-t-residential-gateway-for-pfsense-2-4-5) for which all of this was discussed.

> I've been working on this one for a while. This is the result of others posting their work across various forums, reading BSD docs, and plenty of testing as a result of needing something to do while being stuck at home. :) 
>
>The purpose of this is to make it easier for AT&T customers who wish to assign more than one IPv6 prefix delegation inside their pfSense firewall to more than one internal network interface. I am providing an example dhcp.conf script and explaining what's needed step-by-step. AT&T customers must have been furnished a Residential Gateway (Pace 5268AC / Arris BGW210-700, possibly others) and have configured the RG in DMZ+/IP Passthrough mode. This has been written with pfSense 2.4.5 in mind. 
>
> Why do this? In short, AT&T U-Verse & Fiber customer equipment is assigned a /60 and can only hand out eight /64 prefix delegations. It is not possible to request a larger PD, however it is possible to request multiple /64 PDs from pfSense's WAN interface. Since the pfSense UI does not expose this functionality directly, it is possible to take advantage of it by supplying a dhcp.conf to override pfSense DHCP6 behavior available from the UI.
>
>. . .
>
> Once this script is in place, if you need to reassign interfaces & prefix delegations, the script has to be updated You will need to edit the IPv6 Track Interface Prefix ID on the LAN/OPT interfaces with the IA-PD you specify in the .conf file.
>
> **-ttmcmurry**

<br/>

# OPNsense Configuration Steps

> **Note:** It is assumed that the `WAN` interface is named "WAN" throughout this guide. If it has a different name in your setup, that is ok. Substitute your `WAN` interface name where applicable throughout this guide.

## #0. Validate Initial Conditions
1. The WAN interface IPv6 Configuration type is configured for "none"
2. The LAN/OPT interfaces' DHCP6 option is set to "none" 

## #1. Create a local copy of the following config template
```
interface {YOUR_WAN_INTERFACE} {
	send ia-na 0;
	send ia-pd 0;
	send ia-pd 1;
	send ia-pd 2;
	send ia-pd 3;
	send ia-pd 4;
	send ia-pd 5;
        send ia-pd 6;
	send ia-pd 7;
	request domain-name-servers;
	request domain-name;
	script "/var/etc/dhcp6c_wan_script.sh";
};
id-assoc na 0 { };
id-assoc pd 0 {
	prefix-interface {YOUR_LAN_INTERFACE} {
		sla-id 0;
		sla-len 0;
	};
};
id-assoc pd 1 { 
	prefix-interface {YOUR_OTHER_LAN_INTERFACE} {
		sla-id 0;
		sla-len 0;
	};
};
id-assoc pd 2 { };
id-assoc pd 3 { };
id-assoc pd 4 { };
id-assoc pd 5 { };
id-assoc pd 6 { };
id-assoc pd 7 { };
``` 
> **Note:** The `script` declaration in the above configuration may have a different path depending on the setup. For example, some systems may have the script located at `/var/etc/dhcp6c_opt4_script.sh`. Ensure that the correct file is referenced via SSH

## #2. Update the "interface" block on line 1
In the config template from step #1, replace `{YOUR_WAN_INTERFACE}` with the network *port name* for the WAN interface.

The network port name can be found under `Interfaces -> Assignments`.

### Example:
<img width="669" alt="Screenshot 2023-04-30 at 3 54 24 AM" src="https://user-images.githubusercontent.com/7748434/235344603-7f636b9f-9871-4c4d-abe5-defc7d3748b4.png">
<img width="424" alt="Screenshot 2023-04-30 at 3 57 09 AM" src="https://user-images.githubusercontent.com/7748434/235344661-7362f08e-1c4b-4abd-9aff-5ce0e934f9fa.png">

This results in the following configuration segment:
```
interface igc3 {
	send ia-na 0;
	send ia-pd 0;
	. . .
```
> IA-NA Note: The IA-NA is an arbitrary number. A unique number must be chosen for each device connected to the AT&T residential gateway (RG) which will request a prefix delegation from the RG. If only one device will be requesting PDs from the RG (i.e. this OPNsense firewall), then "ia-na 0" is fine. 

## #3. Update the "ia-pd" declarations
In the config template from step #1, replace `{YOUR_LAN_INTERFACE}` with the network *port name* for the desired LAN interface.

### Example:
<img width="537" alt="Screenshot 2023-04-30 at 3 59 32 AM" src="https://user-images.githubusercontent.com/7748434/235344756-a1ffb66a-1d34-43eb-b152-61c95ce70f15.png">
<img width="331" alt="Screenshot 2023-04-30 at 4 00 36 AM" src="https://user-images.githubusercontent.com/7748434/235344784-0e934bbb-8eb8-4b44-ae47-75f1df251217.png">

This results in the following configuration segment:
```
id-assoc pd 0 {
	prefix-interface igc0 {
		sla-id 0;
		sla-len 0;
	};
};
id-assoc pd 1 { 
	. . .
```

Network ports can be arbitrarily assigned to PDs, staring with `pd 0` and working down the list. Note that formatting is specific. Each new PD declaration needs to be formatted exactly as `id-assoc pd 0` is in the above example; only with an updated network port name. 

The `sla-id` and `sla-len` declarations are always zero (`0`).

> **Note:** If a particular PD is not desired, it does not need to be declared in the config file. The `send ia-pd` and its respective `id-assoc pd` declaration only needs to be declared if it is going to be used by an interface. 

> **Note:**  Assigned PDs will result in numerically different networks, depending on the RG.
>
> * Pace 5268AC first assigns F then decrements to 8 to PD 0-7, i.e. PD0 = ::xxxF::/64 
> * Arris BGW210-700 first assigns 8 then increments to F to PD 0-7, i.e. PD0 = ::xxx8::/64 

## #4. Add the script to OPNsense
* SSH to OPNsense and edit the file with an editor like vi, or edit on your computer and SCP to OPNsense
* Create the file as /usr/local/etc/rc.d/att-rg-dhcpv6-pd.conf
	* Ensure there is no trailing space in the filename.
* Copy and paste your edited script into the text window.

## #5. Edit the WAN interface
1. Navigate to `Interfaces -> WAN`
1. Set the IPv6 Configuration Type to `DHCP6`
1. Under the `DHCP6 Client Configuration` section, set the `DHCPv6 Prefix Delegation size` to `60`
1. Ensure all other check boxes in this section and the Advanced section are unchecked.
1. Under Config File Override, enter the path of the configuration override file from earlier into the `Configuration File Override` text box. Leave `Use IPV4 connectivity` unchecked and `Use VLAN priority` as Disabled.
	* i.e., `/usr/local/etc/rc.d/att-rg-dhcpv6-pd.conf `
1. Click the `Save` button and apply the changes

## #6. Edit the LAN/OPT interface(s), one at a time 
1. Under `General Configuration`, set the `IPv6 Configuration Type` to `Track Interface`
1. Under `Track IPv6 Interface`, set the `IPv6 Interface` to the `WAN` interface name
1. Set the `IPv6 Prefix ID` to the correlated PD number configured in the configuration file from earlier
2. Check the `Allow manual adjustment of DHCPv6 and Router Advertisements` box
1. Click on the `Save` button and apply the changes

**Example:**

<img width="537" alt="Screenshot 2023-04-30 at 3 59 32 AM" src="https://user-images.githubusercontent.com/7748434/235344756-a1ffb66a-1d34-43eb-b152-61c95ce70f15.png">
<img width="539" alt="Screenshot 2023-04-30 at 4 03 29 AM" src="https://user-images.githubusercontent.com/7748434/235344877-3e100c7f-ebbb-4abb-82b3-8035c3fc9f2c.png">
<img width="804" alt="Screenshot 2023-04-30 at 4 06 49 AM" src="https://user-images.githubusercontent.com/7748434/235345072-f5723bbc-99a3-40b0-b050-3d7aae5f9dc1.png">

> **Note:** Be sure to use the `id-assoc pd` number associated with the respective network port for the `IPv6 Prefix ID`.

## #7. Enable OPNsense DHCPv6 Server
Navigate to `Services -> DHCPv6` and click on the LAN interface you want to enable it on
> **Note:** If you have Google devices in this LAN, you can use SLAAC and keep the DHCPv6 server off.

#### Perform the following actions for each interface:
* Within the `DHCPv6 Server` tab
	* Locate the `DHCPv6 Options` section
		* Check the `DHCPv6 Server`  
		* Set the desired `Range`
			* i.e., `::` to `::ffff:ffff:ffff:ffff`
		* Set the `Prefix Delegation Size` to `64`
	* Click the `Save` button
* Within the `Router Advertisements` tab
	* Locate the `Advertisements` section
		* Set the `Router Mode` to `Managed`
	* Click the `Save` button

> **Note:** After applying these settings, it may take several minutes for IPv6 addresses to start populating approprately.

#### If you are using SLAAC for Google compatibility, but want to assign your own IPv6 DNS server (ie for PiHole or AdGuard Home)

1. Keep DHCPv6 disabled for the LAN interface
2. Navigate to `Services -> Router Advertisements`
3. Click on each LAN interface
4. Set `Router Advertisements` to Unmanaged
5. Add your DNS Server IPv6 address
6. Click `Save`

> **Note:** Your DNS server can be configured as static using the prefix that belongs to the LAN interface it is connected to. Others have said you can also use the local-link fe80 address which should never change. I assigned a static IP


If all has gone well, IPv6 should now be working.

<br/>

# State Limits
AT&T Residential gateways have a state table that is far smaller than pfSense's defaults, which can result in problems once the RG begins tracking more states than available. pfSense should be set to never go above that limit. pfSense will adjust how states are managed based on its default adaptive algorithm from "Firewall Adaptive Timeouts." There is no need to adjust pfSense default Adaptive Timeout behavior, only the maximum number of states pfSesnse can use.

The values below are from known hardware & firmware capabilities. Depending on the # of devices directly plugged into the RG, like U-Verse set-top-boxes and devices NOT behind pfSense, you may need to adjust pfSense's maximum states downward. This information can be found on the RG under `Settings -> Diagnostics -> NAT`.

* **Pace 5268AC** - Firmware v11.5.1.532678-att - 15460 states max - Set pfSense to 15000 states
* **Arris NVG599** - Firmware v9.2.2h0d79 - 4096 states max - Set pfSense to 3500 states
* **Arris BGW210-700** - Firmware 1.9.16 - 8000 states max - Set pfSense to 7500 states
* **Motorola NVG589** - Firmware ? - 8192 states max - Set pfSense to 7600 states
* **HUMAX BGW320-500** - Firmware 2.10.6 - 8192 states max - Set pfSense to 7600

Set the pfSense state limit in `Advanced -> Firewall & NAT -> Firewall Maximum States`

> **Note:** If anyone has more up-to-date information about RG firmware and state capabilities, let me know and I'll update this table.
