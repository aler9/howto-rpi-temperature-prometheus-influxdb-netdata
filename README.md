Sample code and instructions on how to push the Raspberry Pi internal temperature value into Prometheus, InfluxDB or Netdata. No external libraries are required, everything is performed through unix shell scripts.

## Reasons

Performance monitoring refers to the measurement and storage of system parameters over a certain window of time. It is a mandatory practice when dealing with professional servers, and it is also spreading in the IoT world, as it allows a fast evaluation of a system reliability over time. The gathering is not limited to performance-related parameters and can be applied to a wide variety of fields: the same monitoring tools are often used to save measurements from weather stations, vehicles, stock markets. The Raspberry Pi is currently the most popular single board computer available, and it offers a ready-to-use temperature sensor that can be used to evaluate both CPU stress and weather conditions.

Prometheus and InfluxDB are two common time-series database used for storaging data gathered by external programs, while Netdata is both a database and a data gatherer. A good amount of tools already exists to provide the RPI temperature to these systems, for instance:
* https://github.com/lukasmalkmus/rpi_exporter
* https://github.com/fgrosse/pi-temp
* https://github.com/turmoni/temp-probe-exporter

But since we're dealing with a computer with limited capabilities, and at the same time the quantity we're interested in is trivial to measure, it is possible to avoid the installation of additional softwares and achieve the objective by using only shell scripts, with significant advantanges in terms of memory consumption and setup time.

## Dependencies

The command-line utilities `curl` and `netcat` are required for the scripts to work, and can be installed through:
* Debian/Ubuntu: `sudo apt-get install -y curl netcat-traditional`
* Alpine Linux: `apk add curl`

## Prometheus

Save this script somewhere and launch it at system startup: it creates a minimal Prometheus exporter which provides the temperature. Make sure that the device `/sys/class/thermal/thermal_zone0/temp` is readable and that the port `7028` can be accessed by Prometheus.

```bash
#!/bin/sh

while true; do
    nc -l -p 7028 -e sh -c '\
        C="rpitemp $(cat /sys/class/thermal/thermal_zone0/temp | sed "s/\([0-9]\{2\}\)/\1./")"; \
        printf "HTTP/1.0 200 OK\nContent-Length: ${#C}\n\n$C"'
done
```

Then add the following lines in the `scrape_configs` section of `prometheus.yml`. The value will be saved in the `rpitemp` metric. You can tune the resolution by replacing `30` with any amount of seconds.

```yaml
- job_name: rpitemp
  scrape_interval: 30s
  static_configs:
  - targets: ['127.0.0.1:7028']
```

## InfluxDB

Save this script somewhere and launch it at system startup: it will periodically push the temperature. Make sure that the device `/sys/class/thermal/thermal_zone0/temp` is readable and that the influxDB port `8086` is accessible. No configuration of InfluxDB is required. The value will be saved in the `rpitemp` database and in the `rpitemp` dimension. You can tune the resolution by replacing `30` with any amount of seconds.

```bash
#!/bin/sh

while true; do
    curl -i -XPOST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE rpitemp" && break
    sleep 1
done

while true; do
    curl -i -XPOST http://localhost:8086/write?db=rpitemp \
        --data-binary "rpitemp value=$(cat /sys/class/thermal/thermal_zone0/temp | sed 's/\([0-9]\{2\}\)/\1./')"
    sleep 30
done
```

## Netdata

Save this script somewhere and launch it at system startup: it will periodically push the temperature. Make sure that the device `/sys/class/thermal/thermal_zone0/temp` is readable and that the netdata port `8125` is accessible. No configuration of netdata is required. The value will appear in the `statsd` section. You can tune the resolution by replacing `30` with any amount of seconds.

```bash
#!/bin/sh

while true; do
    printf "rpitemp:$(cat /sys/class/thermal/thermal_zone0/temp | sed 's/\([0-9]\{2\}\)/\1./')|g\n" \
        | nc -u -w1 localhost 8125
    sleep 30
done
```
