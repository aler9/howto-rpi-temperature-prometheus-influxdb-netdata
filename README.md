Sample code and instructions on how to push the Raspberry Pi internal temperature value into respectively Prometheus, InfluxDB or Netdata.

## Prometheus

## InfluxDB

## Netdata

Save this script somewhere and launch it at system startup. Ensure that the device `/sys/class/thermal/thermal_zone0/temp` is readable and that the netdata port 8125 is accessible. No configuration of netdata is required. The value will appear in the "statsd" section. You can tune the resolution by replacing `30` with any amount of seconds.

```bash
#!/bin/sh

while true; do
    printf "rpi_temp:$(cat /sys/class/thermal/thermal_zone0/temp | sed 's/\([0-9]\{2\}\)/\1./')|g\n" \
        | nc -u -w1 localhost 8125
    sleep 30
done
```
