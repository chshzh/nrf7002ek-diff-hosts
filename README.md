# nrf7002ek-diff-hosts
Evaluation to run Zephyr Wi-Fi samples using nRF7002EK with STM32/NXP chips as hosts.
## Hardware 
* [Nucleo H723ZG](https://docs.zephyrproject.org/latest/boards/st/stm32h750b_dk/doc/index.html) + [nRF7002EK](https://docs.zephyrproject.org/latest/boards/shields/nrf7002ek/doc/index.html)
* [MIMXRT1160-EVK](https://docs.zephyrproject.org/latest/boards/st/stm32h750b_dk/doc/index.html) + [nRF7002EK](https://docs.zephyrproject.org/latest/boards/shields/nrf7002ek/doc/index.html)

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
west update #Donwload other modules
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

Note: Due to the lack of crypto lib support on Zephyr, same commands like "wifi ap" may not be supported.

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

The sample uses an MQTT broker with static IP in the LAN. To make the test simpler, replace `CONFIG_NET_CONFIG_PEER_IPV4_ADDR` with a public MQTT IPv4 address. Ping pubice NTP server like `broker.hivemq.com` to get its ip `3.123.159.193` and replace current `CONFIG_NET_CONFIG_PEER_IPV4_ADDR` in prj.conf to do quick test.

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

## [zephyr/samples/net/sockets/zperf(similar to NCS throughput sample)](https://docs.zephyrproject.org/latest/samples/net/zperf/README.html)

### Nucleo H723ZG

```
west build -p -b nucleo_h723zg -S wifi-ipv4 -- -DSHIELD=nrf7002ek -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

### MIMXRT1160-EVK

```
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -S wifi-ipv4 -- -DSHIELD=nrf7002ek -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

Limitation:
1. Softap sample
Due to the lack of crypto lib support on Zephyr, this sample cannot work on other hosts than Nordic chips which use nRF Security library only avaliable in NCS.
2. QSPI
The nRF7002 Wi-Fi driver (/zephyr/modules/nrf_wifi/bus/qspi_if.c) is tightly coupled to Nordic hardware,Nordic-specific nrfx_qspi driver APIs. Both STM(OCTOSPI controller for QSPI operations) and NXP(FlexSPI controller for QSPI operations) has their own QSPI driver, not sue if they can be used for general Zephyr QSPI usage.