# docker-multifeeder
Feed multiple ADS-B and MLAT aggregators from a single container

[![Docker Image Size (tag)](https://img.shields.io/docker/image-size/kx1t/multifeeder/latest)](https://github.com/sdr-enthusiasts/docker-multifeeder)
[![Discord](https://img.shields.io/discord/734090820684349521)](https://discord.gg/sTf9uYF)

## What is it?
This container enables the user to quickly add feeds to the "new" ADS-B and MLAT aggregators that have popped up.
It includes @wiedehopf's optimized versions of `readsb` and `mlat_client`.

## How to configure?
- For ADS-B, put your (ADS-B, UAT, JAERO, HFDL, etc) input sources and the outputs to the target services in the `READSB_NET_CONNECTOR` parameter
- For MLAT, define the MLAT server targets in the `MLAT_CONFIG` parameter
- For MLAT, `READSB_LAT`, `READSB_LON`, and `READSB_ALT` must also be defined.

| Variable | Description |
|----------|-------------|
| `READSB_NET_CONNECTOR` | `host1,port1,protocol1;host2,port2,protocol2` etc. See "`READSB_NET_CONNECTOR` syntax" below. | 
| `MLAT_CONFIG` | `mlatserver_1,mlatserver_port_1[,results_port_1[,extra_params_1]];mlatserver_2,mlatserver_port_2[,results_port_2[,extra_params_2]]` etc. See "`MLAT_CONFIG` syntax" below | 
| `READSB_LAT` | Reference/receiver surface latitude |
| `READSB_LON` | Reference/receiver surface longitude |
| `READSB_ALT` | Reference/receiver altitude above ground. Add `m` for meters or `ft` for feet |
| `UUID`       | UUID to be used with `beast_reduce_plus_out` protocol for `READSB_NET_CONNECTOR`. This will also be used for `MLAT_USER` if that variable is not defined. See below on how to generate the value of this parameter |
| `MLAT_USER` | Username to be used with MLAT. This variable is mandatory if `MLAT_CONFIG` is used and `UUID` is not set. If `MLAT_USER` is set, its value will be used; if not set, then the value of `UUID` will be used |

#### `READSB_NET_CONNECTOR` syntax

This variable allows you to configure incoming and outgoing connections. The variable takes a semicolon (`;`) separated list of `host,port,protocol[,uuid=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX]`, where:

* `host` is an IP address. Specify an IP/hostname/containername for incoming or outgoing connections.
* `port` is a TCP port number
* `protocol` can be one of the following:
  * `beast_reduce_plus_out`: Beast-format output with extra data (UUID). This is the preferred format when feeding the new aggregator services
  * `beast_out`: Beast-format output
  * `beast_in`: Beast-format input
  * `raw_out`: Raw output
  * `raw_in`: Raw input
  * `sbs_out`: SBS-format output
  * `vrs_out`: SBS-format output
* `uuid=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX` is an optional parameter that sets the UUID for this specific instance. It will override the global `UUID` parameter

See the `docker-compose.yml` example below.

#### How to generate a UUID

If you already have a `UUID` that was generated for the ADSBExchange service, feel free to reuse that one. If you don't have one, you can generate one by logging onto you Linux machine (Raspberry Pi, etc.) and giving this command:

```
cat  /proc/sys/kernel/random/uuid
```

You can use the output string of this command (in format of `00000000-0000-0000-0000-000000000000`) as your UUID

#### `MLAT_CONFIG` syntax

This variable allows you to configure outgoing MLAT connections and allows you optionally to define a port where MLAT returned results are made available. The variable takes a semicolon (`;`) separated list of `mlatserver,mlatserver_port[,results_port[,extra_params]]`, where:

* `mlat_server` is a hostname or IP address of the MLAT server
* `mlat_port` is the TCP or UDP port number of the MLAT server
* `results_port` is the TCP port in which beast-format results are made available. Note- this port is internal to the container stack and may need to be mapped if access is required outside the container stack
* `extra_params` are any extra parameters you want to pass to `mlat_client`

See the `docker-compose.yml` example below.

## `docker-compose.yml` snippet

```yaml
  multifeeder:
    image: ghcr.io/sdr-enthusiasts/docker-multifeeder
    tty: true
    container_name: multifeeder
    hostname: multifeeder
    restart: unless-stopped
    environment:
      - TZ=${FEEDER_TZ}
      - READSB_NET_CONNECTOR=
         readsb,30005,beast_in;
         dump978,37981,raw_in;
         feed.adsb.fi,30004,beast_reduce_plus_out,uuid=${UUID_ADSBFI};
         feed.adsb.one,64004,beast_reduce_plus_out,uuid=${UUID_ADSBONE};
         in.adsb.lol,30004,beast_reduce_plus_out,uuid=${UUID_ADSBLOL};
         feed.theairtraffic.com,30004,beast_reduce_plus_out,uuid=${UUID_THEAIRTRAFFIC};
         feed.planespotters.net,30004,beast_reduce_plus_out,uuid=${UUID_PLANESPOTTERS}
      - MLAT_CONFIG=
         feed.adsb.fi,31090,39000;
         feed.adsb.one,64006,39001;
         in.adsb.lol,31090,39002;
         feed.theairtraffic.com,31090,39003;
         mlat.planespotters.net,31090,39004
      - READSB_LAT=${FEEDER_LAT}
      - READSB_LON=${FEEDER_LONG}
      - READSB_ALT=${FEEDER_ALT_M}m
      - UUID=01234567-89ab-cdef-0123-4567890abcde
      - MLAT_USER=my_station_id_name

    tmpfs:
      - /run/readsb
      - /var/log
```

Note - if you split an environment variable across multiple lines, please make sure that every line except for the last line ends on `;`, and that all elements are properly aligned as shown in the example.

## Receiving MLAT results
If you run the `mlathub` container, or if you want to make the MLAT results available to your visualization software (piaware, tar1090, etc), you need to tell these containers to connect to the MLAT results port(s) of the `multifeeder` container. For example, using the settings in the sample `docker-compose.yml` (above) as a guideline:
```
      - READSB_NET_CONNECTOR=...;multifeeder,39000,beast_in;multifeeder,39001,beast_in;multifeeder,39002,beast_in;multifeeder,39003,beast_in
```

## Use of `beast-reduce` to lower the system load
In order to alleviate the load on your system and on the aggregators you are feeding, you can connect to the `readsb` "`beast-reduce`" port. This port needs to be enabled in your `readsb` or `dump1090` instance.

These are the configuration changes that you need to make (or verify that they have been made):

- Containerized setup: make sure that for the the `readsb-protobuf` or `tar1090` container, the `READSB_BEAST_REDUCE_PORT` is set as follows:
```
     - READSB_NET_BEAST_REDUCE_OUT_PORT=30006
```
- Non-containerized setup: make sure the following command line parameter is included with `readsb` or `dump1090[-fa|-mutability]` setup:
```
--net-beast-reduce-out-port=30006
```

Once you have done this, you can point your `multifeeder` instance at this port:
```
      - READSB_NET_CONNECTOR=readsb,30006,beast_in;...
```
## List of Aggregators
| **Site**          | **readsb_url**         | **readsb_port** | **mlat_url**           | **mlat_port** |
|-------------------|------------------------|-----------------|------------------------|---------------|
| [adsb.fi](https://adsb.fi/)           | feed.adsb.fi           |           30004 | feed.adsb.fi           |         31090 |
| [adsb.one](https://adsb.one/)          | feed.adsb.one          |           64004 | feed.adsb.one          |         64006 |
| [adsb.lol](https://adsb.lol/)          | feed.adsb.lol          |            1337 | feed.adsb.lol          |          1338 |
| [Planespotters](https://planespotters.net)    | feed.planespotters.net |           30004 | mlat.planespotters.net |         31090
| [The Air Traffic](https://theairtraffic.com/) | feed.theairtraffic.com |           30004 | feed.theairtraffic.com |         31090 |

## Getting Help
You can log an issue on the project's GitHub. I also have a [Discord channel](https://discord.gg/sTf9uYF), feel free to join and converse. The #adsb-containers channel is appropriate for conversations about this package.

## Summary of License Terms
Multifeeder is Copyright (C) 2023, Ramon F. Kolb (kx1t)

The `multifeeder` container incorporates `readsb` and `mlat_client`, are copyright by their authors and used under the terms of the GPLv3 license.

This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program. If not, see https://www.gnu.org/licenses/.
