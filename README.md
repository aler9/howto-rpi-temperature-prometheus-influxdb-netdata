Sample code and instructions on how to push the Raspberry Pi internal temperature value into Prometheus, InfluxDB or Netdata. No external libraries are required, everything is performed through unix shell scripts.

curl and netcat must be installed for the scripts to work:
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
