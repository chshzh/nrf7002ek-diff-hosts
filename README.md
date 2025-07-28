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

After install Zephyr, fetch latest nrf wifi firmware.

```
% west blobs fetch nrf_wifi
Fetching blob nrf_wifi: /Users/chsh/zephyrproject/modules/lib/nrf_wifi/zephyr/blobs/wifi_fw_bins/default/nrf70.bin
Fetching blob nrf_wifi: /Users/chsh/zephyrproject/modules/lib/nrf_wifi/zephyr/blobs/wifi_fw_bins/scan_only/nrf70.bin
Fetching blob nrf_wifi: /Users/chsh/zephyrproject/modules/lib/nrf_wifi/zephyr/blobs/wifi_fw_bins/radio_test/nrf70.bin
Fetching blob nrf_wifi: /Users/chsh/zephyrproject/modules/lib/nrf_wifi/zephyr/blobs/wifi_fw_bins/system_with_raw/nrf70.bin
Fetching blob nrf_wifi: /Users/chsh/zephyrproject/modules/lib/nrf_wifi/zephyr/blobs/wifi_fw_bins/offloaded_raw_tx/nrf70.bin
```

Prepare a Zephyr_Env_Setup.sh for convenience, run source Zephyr_Env_Setup.sh before open terminal.

```
#!/bin/zsh
echo "Loading Zephyr Environment ..."
# Source Zephyr environment
source /opt/nordic/zp/v4.1.0/zephyr/zephyr-env.sh

# Activate virtual environment
source $ZEPHYR_BASE/../../.venv/bin/activate

# Export paths
export PATH=/Applications/NXP/LinkServer_25.3.31:$PATH
export PATH=/Applications/STMicroelectronics/STM32Cube/STM32CubeProgrammer/STM32CubeProgrammer.app/Contents/MacOs/bin:$PATH

# Get Zephyr version information
ZEPHYR_VERSION_MAJOR=$(cat $ZEPHYR_BASE/VERSION | grep VERSION_MAJOR | awk '{print $3}')
ZEPHYR_VERSION_MINOR=$(cat $ZEPHYR_BASE/VERSION | grep VERSION_MINOR | awk '{print $3}')
ZEPHYR_PATCHLEVEL=$(cat $ZEPHYR_BASE/VERSION | grep PATCHLEVEL | awk '{print $3}')

# Get SDK version
SDK_VERSION=$(cat $ZEPHYR_BASE/SDK_VERSION)

# Print formatted output
echo "Zephyr: V${ZEPHYR_VERSION_MAJOR}.${ZEPHYR_VERSION_MINOR}.${ZEPHYR_PATCHLEVEL}"
echo "Toolchain: zephyr-sdk-${SDK_VERSION}"

# Change directory to ZEPHYR_BASE
cd $ZEPHYR_BASE

echo "Zephyr Environment Ready!!!"

```

MacOS: Generate an app(suggest to name it ZephyrTerminal) with Script Editor to open zsh terminal and run above script to make Zephyr Envrionment Ready in Terminal.

## Blinky sample test

Install STM32CubeProgrammer to defaul path following [Nucleo H723ZG](https://docs.zephyrproject.org/latest/boards/st/stm32h750b_dk/doc/index.html).

```
cd zephyr
west build -p -b nucleo_h723zg samples/basic/blinky
west flash
```

Install LinkServer to default path following [MIMXRT1160-EVK](https://docs.zephyrproject.org/latest/boards/st/stm32h750b_dk/doc/index.html).

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

Solution 1: Add `-DCONFIG_NET_IF_MAX_IPV4_COUNT=2` in building command to increase the default value from 1 to 2.

```
west build -p -b nucleo_h723zg -- -DSHIELD=nrf7002ek -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

Solution 2: Disable ethernet port with boards/[nucleo\_h723zg.overlay](nucleo_h723zg.overlay).

```
west build -p -b nucleo_h723zg -- -DSHIELD=nrf7002ek
```

### MIMXRT1160-EVK

### Issue1: Same as Nucleo, also have ethernet on board, this will cause warning that expect two ipv4 address.

Solution 1: Add `-DCONFIG_NET_IF_MAX_IPV4_COUNT=2` to increse the default value from 1 to 2.
Solution 2: Disable ethernet port with boards/[mimxrt1160\_evk\_mimxrt1166\_cm7.overlay](mimxrt1160_evk_mimxrt1166_cm7.overlay).

### Issue2: nrf7002eks sheild use arduino pins for SPI connection, but there is no arduino pins definition in mimxrt1160_evk device tree files.

Solution 1: Create a custom device tree overlay file that provides spi connection directly: [mimxrt1160\_evk\_nrf7002ek.overlay](mimxrt1160_evk_nrf7002ek.overlay).

```
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -- -DDTC_OVERLAY_FILE=mimxrt1160_evk_nrf7002ek.overlay -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

Solution 2(better): Add arduino defination on mimxrt1160\_evk board file. zephyr/boards/nxp/mimxrt1160\_evk/[mimxrt1160\_evk.dtsi](mimxrt1160_evk.dtsi).

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
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -S wifi-ipv4 -- -DDTC_OVERLAY_FILE=mimxrt1160_evk_nrf7002ek.overlay -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

## [zephyr/samples/net/sockets/http\_get](https://docs.zephyrproject.org/latest/samples/net/sockets/http_get/README.html)

### Nucleo H723ZG

```
west build -p -b nucleo_h723zg -S wifi-ipv4 -- -DSHIELD=nrf7002ek -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

### MIMXRT1160-EVK

```
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -S wifi-ipv4 -- -DDTC_OVERLAY_FILE=mimxrt1160_evk_nrf7002ek.overlay -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

## [zephyr/samples/net/sockets/sntp\_client](https://docs.zephyrproject.org/latest/samples/net/sockets/sntp_client/README.html)

The sample uses an NTP server with static IP in the LAN by default. To make the test simpler, replace `CONFIG_NET_CONFIG_PEER_IPV4_ADDR` with a public NTP server IPv4 address. Ping pubice NTP server like `ntp.ubuntu.com` to get its ip `185.125.190.56` and replace current `CONFIG_NET_CONFIG_PEER_IPV4_ADDR` in prj.conf to do quick test.

### Nucleo H723ZG

```
west build -p -b nucleo_h723zg -S wifi-ipv4 -- -DSHIELD=nrf7002ek -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
```

### MIMXRT1160-EVK

```
west build -p -b mimxrt1160_evk/mimxrt1166/cm7 -S wifi-ipv4 -- -DDTC_OVERLAY_FILE=mimxrt1160_evk_nrf7002ek.overlay -DCONFIG_NET_IF_MAX_IPV4_COUNT=2
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

## NCS softap sample

Due to the lack of crypto lib support on Zephyr, this sample cannot work on other hosts than Nordic chips which use nRF Security library only avaliable in NCS.
