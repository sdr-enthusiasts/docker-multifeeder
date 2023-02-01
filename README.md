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

#### `READSB_NET_CONNECTOR` syntax

This variable allows you to configure incoming and outgoing connections. The variable takes a semicolon (`;`) separated list of `host,port,protocol`, where:

* `host` is an IP address. Specify an IP/hostname/containername for incoming or outgoing connections.
* `port` is a TCP port number
* `protocol` can be one of the following:
  * `beast_out`: Beast-format output
  * `beast_in`: Beast-format input
  * `raw_out`: Raw output
  * `raw_in`: Raw input
  * `sbs_out`: SBS-format output
  * `vrs_out`: SBS-format output

See the `docker-compose.yml` example below.

#### `MLAT_CONFIG` syntax

This variable allows you to configure outgoing MLAT connections and allows you optionally to define a port where MLAT returned results are made available. The variable takes a semicolon (`;`) separated list of `mlatserver,mlatserver_port[,results_port[,extra_params]]`, where:

* `mlat_server` is a hostname or IP address of the MLAT server
* `mlat_port` is the TCP or UDP port number of the MLAT server
* `results_port` is the TCP port in which beast-format results are made available. Note- this port is internal to the container stack and may need to be mapped if access is required outside the container stack
* `extra_params` are any extra parameters you want to pass to `mlat_client`

See the `docker-compose.yml` example below.

## docker-compose.yml` example

```
  multifeeder:
    image: ghcr.io/sdr-enthusiasts/docker-multifeeder
    tty: true
    container_name: multifeeder
    hostname: multifeeder
    restart: always
    environment:
      - TZ=${FEEDER_TZ}
      - READSB_NET_CONNECTOR=readsb,30005,beast_in;dump978,30978, raw_in;feed.adsb.fi,30004,beast_out;feed.adsb.one,64004,beast_out;in.adsb.lol,30004,beast_out
      - MLAT_USER=SITE_NAME
      - MLAT_CONFIG=feed.adsb.fi,31090,39000;feed.adsb.one,64006,39001;in.adsb.lol,31090,39002
      - READSB_LAT=${FEEDER_LAT}
      - READSB_LON=${FEEDER_LONG}
      - READSB_ALT=${FEEDER_ALT_M}m
    tmpfs:
      - /run/readsb
      - /var/log
```

## Getting Help

You can log an issue on the project's GitHub. I also have a [Discord channel](https://discord.gg/sTf9uYF), feel free to join and converse. The #adsb-containers channel is appropriate for conversations about this package.

## Summary of License Terms

Copyright (C) 2023, Ramon F. Kolb (kx1t)

`readsb` and `mlat_client` are copyright by their authors and used under the terms of the GPLv3 license.

This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program. If not, see https://www.gnu.org/licenses/.
