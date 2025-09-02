# nrf7002ek-diff-hosts
Evaluation to run Zephyr Wi-Fi samples using nRF7002EK with STM32/NXP chips as hosts.
## Hardware 
* [Nucleo H723ZG](https://docs.zephyrproject.org/latest/boards/st/stm32h750b_dk/doc/index.html) + [nRF7002EK](https://docs.zephyrproject.org/latest/boards/shields/nrf7002ek/doc/index.html)
* [MIMXRT1160-EVK](https://docs.zephyrproject.org/latest/boards/nxp/mimxrt1160_evk/doc/index.html) + [nRF7002EK](https://docs.zephyrproject.org/latest/boards/shields/nrf7002ek/doc/index.html)

## Firmware
* [zephyr/samples/net/wifi/shell](https://docs.zephyrproject.org/latest/samples/net/wifi/shell/README.html)
* [zephyr/samples/net/mqtt\_publisher](https://docs.zephyrproject.org/latest/samples/net/mqtt_publisher/README.html)
* [zephyr/samples/net/sockets/http\_get](https://docs.zephyrproject.org/latest/samples/net/sockets/http_get/README.html)
* [zephyr/samples/net/sockets/sntp\_client](https://docs.zephyrproject.org/latest/samples/net/sockets/sntp_client/README.html)
* [zephyr/samples/net/sockets/zperf(for throughput test)](https://docs.zephyrproject.org/latest/samples/net/zperf/README.html)

# Vanilla Zephyr v4.1.0 Setup

## Install Zephyr and Toolchain

* [Getting Started Guide — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/develop/getting_started/index.html#getting-started-guide)
* [Zephyr SDK — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/develop/toolchains/zephyr_sdk.html#toolchain-zephyr-sdk-install)

Zephyr installation on MacOS
```
brew install cmake ninja gperf python3 python-tk ccache qemu dtc libmagic wget openocd

mkdir /opt/nordic/zp/v4.1.0

cd /opt/nordic/zp/v4.1.0
python3 -m venv .venv

source .venv/bin/activate

pip install west

west init . #Download zephyr
west update #Download other modules
west zephyr-export
west packages pip --install

cd zephyr
west sdk install -b /opt/nordic/zp/toolchains
west blobs fetch nrf_wifi
```

## VS Code Workspace Configuration

Create a VS Code workspace file (e.g., `zephyr410.code-workspace`) with the following configuration for easy environment setup:

```json
{
	"folders": [
		{
			"path": "."
		}
	],
	"settings": {
		"terminal.integrated.profiles.osx": {
			"Zephyr410SH": {
				"path": "zsh",
				"args": [
					"-c",
					"source /opt/nordic/zp/v4.1.0/.venv/bin/activate && source /opt/nordic/zp/v4.1.0/zephyr/zephyr-env.sh && EXPECTED_SDK_VERSION=$(cat $ZEPHYR_BASE/SDK_VERSION | tr -d '\\n') && export ZEPHYR_SDK_INSTALL_DIR=/opt/nordic/zp/toolchains/zephyr-sdk-$EXPECTED_SDK_VERSION && export PATH=/Applications/NXP/LinkServer:$PATH && export PATH=/Applications/STMicroelectronics/STM32Cube/STM32CubeProgrammer/STM32CubeProgrammer.app/Contents/MacOs/bin:$PATH && cd $ZEPHYR_BASE && ZEPHYR_VERSION=$(grep 'VERSION_MAJOR' $ZEPHYR_BASE/VERSION | cut -d' ' -f3).$(grep 'VERSION_MINOR' $ZEPHYR_BASE/VERSION | cut -d' ' -f3).$(grep 'PATCHLEVEL' $ZEPHYR_BASE/VERSION | cut -d' ' -f3) && echo 'Zephyr Environment Ready!!!' && echo '========================' && echo 'Zephyr Version:' && echo $ZEPHYR_VERSION && echo 'Expected SDK Version:' && cat $ZEPHYR_BASE/SDK_VERSION && echo 'Using Toolchain:' && echo $ZEPHYR_SDK_INSTALL_DIR && if [ -d \"$ZEPHYR_SDK_INSTALL_DIR\" ]; then echo 'Toolchain Found: ✓'; else echo 'Toolchain Missing: ✗'; fi && echo '========================' && exec /bin/zsh"
				]
			}
		}
	}
}
```

When you open the `Zephyr410SH` terminal profile, you should see the following output:

**If toolchain is installed:**
```
Zephyr Environment Ready!!!
========================
Zephyr Version:
4.1.0
Expected SDK Version:
0.17.0
Toolchain Found: ✓
Using Toolchain:
/opt/nordic/zp/toolchains/zephyr-sdk-0.17.0
========================
```

**If toolchain is missing:**
```
Zephyr Environment Ready!!!
========================
Zephyr Version:
4.1.0
Expected SDK Version:
0.17.0
Expected Toolchain Missing: ✗
Please install Toolchain:
/opt/nordic/zp/toolchains/zephyr-sdk-0.17.0
========================
```

This configuration automatically:
- Activates the Python virtual environment
- Sources the Zephyr environment
- Dynamically detects and sets the correct toolchain version
- Adds external tool paths (LinkServer, STM32CubeProgrammer)
- Validates that the expected toolchain is available
- Displays version information for verification

## Blinky sample test

Install STM32CubeProgrammer to defaul path following [Nucleo H723ZG](https://docs.zephyrproject.org/latest/boards/st/stm32h750b_dk/doc/index.html).

```
cd zephyr
west build -p -b nucleo_h723zg samples/basic/blinky
west flash
```

Install LinkServer to default path following [MIMXRT1160-EVK](https://docs.zephyrproject.org/latest/boards/nxp/mimxrt1160_evk/doc/index.html).

Note: The LinkServer on board seems not very stable so the writer used a Jlink debugger as programmer following [Using J-Link with MIMXRT1160-EVK or MIMXRT1170-EVK](https://community.nxp.com/t5/i-MX-RT-Crossover-MCUs-Knowledge/Using-J-Link-with-MIMXRT1160-EVK-or-MIMXRT1170-EVK/ta-p/1529760).

```
cd zephyr
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 samples/basic/blinky
west flash -r jlink
```

# Zephyr Wi-Fi Samples Adaptation

## [zephyr/samples/net/wifi/shell](https://docs.zephyrproject.org/latest/samples/net/wifi/shell/README.html)

Note: Due to the lack of crypto lib support on Zephyr, some commands like "wifi ap" may not be supported when use host chips from other venders than Nordic.

### Nucleo H723ZG

### Issue: Nucleo also have ethernet on board, this will cause warning that expect two ipv4 address.

Solution: Add `-DCONFIG_NET_IF_MAX_IPV4_COUNT=2` in building command to increase the default maximum network interface from 1 to 2.

```
west build -p -b nucleo_h723zg -- -DSHIELD=nrf7002ek -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

### MIMXRT1160-EVK

### Issue1: Same as Nucleo, also have ethernet on board, this will cause warning that expect two ipv4 address.

Solution: Add `-DCONFIG_NET_IF_MAX_IPV4_COUNT=2` to increse the default value from 1 to 2.

### Issue2: nrf7002EK sheild use arduino pins for SPI connection, but there is no arduino pins definition in mimxrt1160_evk device tree files.

Solution 1: Create a custom device tree overlay file that provides spi connection directly: [mimxrt1160\_evk\_spi\_nrf7002ek.overlay](mimxrt1160_evk_spi_nrf7002ek.overlay).

```
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -- -DDTC_OVERLAY_FILE=mimxrt1160_evk_spi_nrf7002ek.overlay -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

Solution 2: Add arduino defination on mimxrt1160\_evk board file. zephyr/boards/nxp/mimxrt1160\_evk/[mimxrt1160\_evk.dtsi](mimxrt1160_evk.dtsi).

[https://github.com/zephyrproject-rtos/zephyr/issues/92540](https://github.com/zephyrproject-rtos/zephyr/issues/92540)

```
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -- -DSHIELD=nrf7002ek  -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

## [zephyr/samples/net/mqtt\_publisher](https://docs.zephyrproject.org/latest/samples/net/mqtt_publisher/README.html) 

The sample uses an MQTT broker with static IP in the LAN. To make the test simpler, replace `CONFIG_NET_CONFIG_PEER_IPV4_ADDR` with a public MQTT IPv4 address. Ping pubice MQTT server like `broker.hivemq.com` to get its ip `3.123.159.193` and replace current `CONFIG_NET_CONFIG_PEER_IPV4_ADDR` in prj.conf to do quick test.

### Nucleo H723ZG

```
west build -p -b nucleo_h723zg -S wifi-ipv4 -- -DSHIELD=nrf7002ek -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

### MIMXRT1160-EVK

```
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -S wifi-ipv4 -- -DDTC_OVERLAY_FILE=mimxrt1160_evk_spi_nrf7002ek.overlay -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

## [zephyr/samples/net/sockets/http\_get](https://docs.zephyrproject.org/latest/samples/net/sockets/http_get/README.html)

### Nucleo H723ZG

```
west build -p -b nucleo_h723zg -S wifi-ipv4 -- -DSHIELD=nrf7002ek -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

### MIMXRT1160-EVK

```
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -S wifi-ipv4 -- -DDTC_OVERLAY_FILE=mimxrt1160_evk_spi_nrf7002ek.overlay -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

## [zephyr/samples/net/sockets/sntp\_client](https://docs.zephyrproject.org/latest/samples/net/sockets/sntp_client/README.html)

The sample uses an NTP server with static IP in the LAN by default. To make the test simpler, replace `CONFIG_NET_CONFIG_PEER_IPV4_ADDR` with a public NTP server IPv4 address. Ping pubice NTP server like `ntp.ubuntu.com` to get its ip `185.125.190.56` and replace current `CONFIG_NET_CONFIG_PEER_IPV4_ADDR` in prj.conf to do quick test.

### Nucleo H723ZG

```
west build -p -b nucleo_h723zg -S wifi-ipv4 -- -DSHIELD=nrf7002ek -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

### MIMXRT1160-EVK

```
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -S wifi-ipv4 -- -DDTC_OVERLAY_FILE=mimxrt1160_evk_spi_nrf7002ek.overlay -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

## [zephyr/samples/net/zperf(similar to NCS throughput sample)](https://docs.zephyrproject.org/latest/samples/net/zperf/README.html)

### Nucleo H723ZG

```
west build -p -b nucleo_h723zg -S wifi-ipv4 -- -DSHIELD=nrf7002ek -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

### MIMXRT1160-EVK

```
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -S wifi-ipv4 -- -DDTC_OVERLAY_FILE=mimxrt1160_evk_spi_nrf7002ek.overlay -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

# Log Extracts: Successful Integration

Following is an Shell/Zperf logs showing successful implementation.

```
[00:00:00.051,000] <inf> phy_mii: PHY (0) ID 7C131
[00:00:00.053,000] <inf> wifi_nrf_bus: SPIM spi@40013000: freq = 8 MHz
[00:00:00.053,000] <inf> wifi_nrf_bus: SPIM spi@40013000: latency = 0
[00:00:00.310,000] <inf> wifi_nrf: Management buffer offload enabled

*** Booting Zephyr OS build v4.1.0-2-g08b9cc57c223 ***
[00:00:00.438,000] <inf> net_config: Initializing network
[00:00:00.438,000] <inf> net_config: Waiting interface 1 (0x24001908) to be up...
[00:00:00.438,000] <inf> net_config: Running dhcpv4 client...
[00:00:00.438,000] <inf> wifi_supplicant: wpa_supplicant initializedßß
uart:~$ help
Please press the <Tab> button to see all available commands.
You can also use the <Tab> button to prompt or auto-complete all commands or its subcommands.
You can try to call commands with <-h> or <--help> parameter for more information.

Shell supports following meta-keys:
  Ctrl + (a key from: abcdefklnpuw)
  Alt  + (a key from: bf)
Please refer to shell documentation for more details.

Available commands:
  clear    : Clear screen.
  date     : Date commands
  device   : Device commands
  devmem   : Read/write physical memory
             Usage:
             Read memory at address with optional width:
             devmem <address> [<width>]
             Write memory at address with mandatory width and value:
             devmem <address> <width> <value>
  help     : Prints the help message.
  history  : Command history.
  kernel   : Kernel commands
  net      : Networking commands
  rem      : Ignore lines beginning with 'rem '
  retval   : Print return value of most recent command
  shell    : Useful, not Unix-like shell commands.
  wifi     : Wi-Fi commands
  zperf    : Zperf commands
uart:~$ wifi
wifi - Wi-Fi commands
Subcommands:
  11k                   : Configure 11k or get 11k status.
                          [enable/disable]

  11k_neighbor_request  : Send Neighbor Report Request frame.
                          [ssid <ssid>]

  11v_btm_query         : <query_reason: The reason code for a BSS transition
                          management query>.

  ap                    : Access Point mode commands.
  channel               : wifi channel setting
                          This command is used to set the channel when
                          monitor or TX-Injection mode is enabled
                          Currently 20 MHz is only supported and no BW parameter
                          is provided
                          [-i, --if-index <idx>] : Interface index
                          [-c, --channel <chan>] : Set a specific channel number
                          to the lower layer
                          [-g, --get] : Get current set channel number from the
                          lower layer
                          [-h, --help] : Help
                          Usage: Get operation example for interface index 1
                          wifi channel -g -i1
                          Set operation example for interface index 1 (setting
                          channel 5)
                          wifi -i1 -c5.

  connect               : Connect to a Wi-Fi AP
                          <-s --ssid "<SSID>">: SSID.
                          [-c --channel]: Channel that needs to be scanned for
                          connection. 0:any channel.
                          [-b, --band] 0: any band (2:2.4GHz, 5:5GHz, 6:6GHz]
                          [-p, --psk]: Passphrase (valid only for secure SSIDs)
                          [-k, --key-mgmt]: Key Management type (valid only for
                          secure SSIDs)
                          0:None, 1:WPA2-PSK, 2:WPA2-PSK-256, 3:SAE-HNP,
                          4:SAE-H2E, 5:SAE-AUTO, 6:WAPI,7:EAP-TLS, 8:WEP, 9:
                          WPA-PSK, 10: WPA-Auto-Personal, 11: DPP
                          12: EAP-PEAP-MSCHAPv2, 13: EAP-PEAP-GTC, 14:
                          EAP-TTLS-MSCHAPv2,
                          15: EAP-PEAP-TLS, 20: SAE-EXT-KEY
                          [-w, --ieee-80211w]: MFP (optional: needs security
                          type to be specified)
                          : 0:Disable, 1:Optional, 2:Required.
                          [-m, --bssid]: MAC address of the AP (BSSID).
                          [-t, --timeout]: Timeout for the connection attempt
                          (in seconds).
                          [-a, --anon-id]: Anonymous identity for enterprise
                          mode.
                          [-K, --key1-pwd for eap phase1 or --key2-pwd for eap
                          phase2]:
                          Private key passwd for enterprise mode. Default no
                          password for private key.
                          [-S, --wpa3-enterprise]: WPA3 enterprise mode:
                          Default 0: Not WPA3 enterprise mode.
                          1:Suite-b mode, 2:Suite-b-192-bit mode,
                          3:WPA3-enterprise-only mode.
                          [-T, --TLS-cipher]: 0:TLS-NONE, 1:TLS-ECC-P384,
                          2:TLS-RSA-3K.
                          [-A, --verify-peer-cert]: apply for EAP-PEAP-MSCHAPv2
                          and EAP-TTLS-MSCHAPv2
                          Default 0. 0:not use CA to verify peer, 1:use CA to
                          verify peer.
                          [-V, --eap-version]: 0 or 1. Default 1: eap version 1.
                          [-I, --eap-id1]: Client Identity. Default no eap
                          identity.
                          [-P, --eap-pwd1]: Client Password.
                          Default no password for eap user.
                          [-R, --ieee-80211r]: Use IEEE80211R fast BSS
                          transition connect.[-h, --help]: Print out the help
                          for the connect command.

  disconnect            : Disconnect from the Wi-Fi AP.

  mode                  : mode operational setting
                          This command may be used to set the Wi-Fi device into
                          a specific mode of operation
                          [-i, --if-index <idx>] : Interface index
                          [-s, --sta] : Station mode
                          [-m, --monitor] : Monitor mode
                          [-a, --ap] : AP mode
                          [-k, --softap] : Softap mode
                          [-h, --help] : Help
                          [-g, --get] : Get current mode for a specific
                          interface index
                          Usage: Get operation example for interface index 1
                          wifi mode -g -i1
                          Set operation example for interface index 1 - set
                          station+promiscuous
                          wifi mode -i1 -sp.

  packet_filter         : mode filter setting
                          This command is used to set packet filter setting when
                          monitor, TX-Injection and promiscuous mode is enabled
                          The different packet filter modes are control,
                          management, data and enable all filters
                          [-i, --if-index <idx>] : Interface index
                          [-a, --all] : Enable all packet filter modes
                          [-m, --mgmt] : Enable management packets to allowed up
                          the stack
                          [-c, --ctrl] : Enable control packets to be allowed up
                          the stack
                          [-d, --data] : Enable Data packets to be allowed up
                          the stack
                          [-g, --get] : Get current filter settings for a
                          specific interface index
                          [-b, --capture-len <len>] : Capture length buffer size
                          for each packet to be captured
                          [-h, --help] : Help
                          Usage: Get operation example for interface index 1
                          wifi packet_filter -g -i1
                          Set operation example for interface index 1 - set
                          data+management frame filter
                          wifi packet_filter -i1 -md.

  pmksa_flush           : Flush PMKSA cache entries.

  ps                    : Configure or display Wi-Fi power save state.
                          [on/off]

  ps_exit_strategy      : <tim> : Set PS exit strategy to Every TIM
                          <custom> : Set PS exit strategy to Custom
  ps_listen_interval    : <val> - Listen interval in the range of <0-65535>.

  ps_mode               : <mode: legacy/WMM>.

  ps_timeout            : <val> - PS inactivity timer(in ms).

  ps_wakeup_mode        : <wakeup_mode: DTIM/Listen Interval>.

  reg_domain            : Set or Get Wi-Fi regulatory domain
                          [ISO/IEC 3166-1 alpha2]: Regulatory domain
                          [-f]: Force to use this regulatory hint over any other
                          regulatory hints
                          Note1: The behavior of this command is dependent on
                          the Wi-Fi driver/chipset implementation
                          Note2: This may cause regulatory compliance issues,
                          use it at your own risk.
                          [-v]: Verbose, display the per-channel regulatory
                          information

  rts_threshold         : <rts_threshold: rts threshold/off>.

  scan                  : Scan for Wi-Fi APs
                          [-t, --type <active/passive>] : Preferred mode of
                          scan. The actual mode of scan can depend on factors
                          such as the Wi-Fi chip implementation, regulatory
                          domain restrictions. Default type is active
                          [-b, --bands <Comma separated list of band values
                          (2/5/6)>] : Bands to be scanned where 2: 2.4 GHz, 5: 5
                          GHz, 6: 6 GHz
                          [-a, --dwell_time_active <val_in_ms>] : Active scan
                          dwell time (in ms) on a channel. Range 5 ms to 1000 ms
                          [-p, --dwell_time_passive <val_in_ms>] : Passive scan
                          dwell time (in ms) on a channel. Range 10 ms to 1000
                          ms
                          [-s, --ssid] : SSID to scan for. Can be provided
                          multiple times
                          [-m, --max_bss <val>] : Maximum BSSes to scan for.
                          Range 1 - 65535
                          [-c, --chans <Comma separated list of channel ranges>]
                          : Channels to be scanned. The channels must be
                          specified in the form
                          band1:chan1,chan2_band2:chan3,..etc. band1, band2 must
                          be valid band values and chan1, chan2, chan3 must be
                          specified as a list of comma separated values where
                          each value is either a single channel or a channel
                          range specified as chan_start-chan_end. Each band
                          channel set has to be separated by a _. For example, a
                          valid channel specification can be 2:1,6_5:36 or
                          2:1,6-11,14_5:36,163-177,52. Care should be taken to
                          ensure that configured channels don't exceed
                          CONFIG_WIFI_MGMT_SCAN_CHAN_MAX_MANUAL
                          [-h, --help] : Print out the help for the scan
                          command.

  statistics            : Wi-Fi interface statistics.
                          [reset] : Reset Wi-Fi interface statistics
                          [help] :  Print out the help for the statistics
                          command.
  status                : Status of the Wi-Fi interface.

  twt                   : Manage TWT flows.
  version               : Print Wi-Fi Driver and Firmware versions

  wps_pbc               : Start a WPS PBC connection.

  wps_pin               : Set and get WPS pin.
                          [pin] Only applicable for set.

uart:~$ zperf
zperf - Zperf commands
Subcommands:
  connectap  : Connect to AP
  setip      : Set IP address
               <my ip> <prefix len>
               Example setip 2001:db8::2 64
               Example setip 192.0.2.2

  tcp        : Upload/Download TCP data
  udp        : Upload/Download UDP data
  version    : Zperf version
uart:~$ 
```

# Limitation
1. Softap sample
Due to the lack of crypto lib support on Zephyr, this sample cannot work on other hosts than Nordic chips which use nRF Security library only avaliable in NCS.
2. QSPI
The nRF7002 Wi-Fi driver (/zephyr/modules/nrf_wifi/bus/qspi_if.c) is tightly coupled to Nordic hardware,Nordic-specific nrfx_qspi driver APIs. Both STM(OCTOSPI controller for QSPI operations) and NXP(FlexSPI controller for QSPI operations) has their own QSPI driver, not sue if they can be used for general Zephyr QSPI usage.
