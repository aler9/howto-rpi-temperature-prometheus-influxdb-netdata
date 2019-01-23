Sample code and instructions on how to push the Raspberry Pi internal temperature value into Prometheus, InfluxDB or Netdata. No external libraries are required, everything is performed through unix shell scripts.

## Prometheus

## InfluxDB

Save this script somewhere and launch it at system startup. Ensure that the device `/sys/class/thermal/thermal_zone0/temp` is readable and that the influxDB port 8086 is accessible. No configuration of InfluxDB is required. The value will be saved in the `rpitemp` database. You can tune the resolution by replacing `30` with any amount of seconds.

```bash
#!/bin/sh

while true; do
    curl -i -XPOST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE rpitemp" && break
    sleep 1
done

while true; do
    curl -i -XPOST 'http://localhost:8086/write?db=rpitemp' \
        --data-binary "rpi_temp value=$(cat /sys/class/thermal/thermal_zone0/temp | sed 's/\([0-9]\{2\}\)/\1./')"
    sleep 30
done
```

## Netdata

Save this script somewhere and launch it at system startup. Ensure that the device `/sys/class/thermal/thermal_zone0/temp` is readable and that the netdata port 8125 is accessible. No configuration of netdata is required. The value will appear in the `statsd` section. You can tune the resolution by replacing `30` with any amount of seconds.

```bash
#!/bin/sh

while true; do
    printf "rpi_temp:$(cat /sys/class/thermal/thermal_zone0/temp | sed 's/\([0-9]\{2\}\)/\1./')|g\n" \
        | nc -u -w1 localhost 8125
    sleep 30
done
```
