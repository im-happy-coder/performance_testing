
# 성능 테스트 환경 구성 및 스크립트 개발

## Performance Diagram

![performance-diagram](image/performance-dashboard.png)

### Description

Jmeter를 이용하여 임계테스트, 스파이크 테스트, 스트레스 테스트 등의 각종 테스트 결과와 테스트 동안 발생한 서버 I/O, 네트워크 사용률, 메모리 사용률 등의 모니터링 정보를 InfluxDB에 저장한다.

InfluxDB와 Grafana를 연결하여 InfluxDB에 저장된 데이터를 Grafana DashBoard를 이용하여 시각화 할 수있다.

추가로 Sitespeedio를 이용하여 사이트의 성능을 측정하여 테스트하려는 사이트의 js, static, 로딩시간 등 성능을 sitespeedio가 자체적으로 검증하여 해당 결과를 Graphite를 통해 데이터를 수집하고 Grafana Dashboard에서 시각화 할 수있다.


### Result

- Jmeter Script
  
![jmeter1](image/jmeter-sample1.PNG)


- Grafana Load Test Dashboard

![grafana-result](image/grafana-dashboard2.PNG)

## Enviroment

Linux - CentOS Linux release 7.9.2009 (Core)

Docker - Docker version 20.10.22

DockerCompose - Docker Compose version v2.15.1

Grafana - v9.2.0

Sitespeedio - v1.1.10-3

InfluxDB - v2.0.9

Telegraf - v1.19.3


## Start Docker-compose And Create .ini

### Directory 

Using my Directory 

``` tree
/app/performance
.
├── docker-compose.yml
├── grafana
│   └── etc
│       └── grafana.ini
├── sitespeedio
│   └── result
└── telegraf
    └── etc
        └── telegraf.conf
```


### telegraf conf 생성

docker run --rm telegraf:1.19.3 telegraf config > /app/performance/telegraf/etc/telegraf.conf

### grafana ini 생성
docker run --rm --entrypoint /bin/sh grafana/grafana:9.2.0 -c "cat /etc/grafana/grafana.ini" > /app/performance/grafana/etc/grafana.ini

### yaml Create 

``` yml
version: '3'
services:
    telegraf:
      image: telegraf:1.19.3
      container_name: telegraf
      depends_on:
        - influxdb
      volumes:
        - /app/performance/telegraf/etc/telegraf.conf:/etc/telegraf/telegraf.conf:ro
        - /var/run/docker.sock:/var/run/docker.sock
        - /:/host:ro
      environment:
        - HOST_PROC=/host/proc
        - HOST_MOUNT=/host
      privileged: true
      restart: always
    influxdb:
      image: influxdb:2.0.9
      container_name: influxdb2
      ports:
        - "8086:8086"
      volumes:
        - influxdb:/var/lib/influxdb2
      network_mode: host
      restart: always
    grafana:
      image: grafana/grafana:9.2.0
      hostname: grafana
      depends_on:
        - influxdb
        - graphite
      links:
        - influxdb
        - graphite
      ports:
        - "3000:3000"
      environment:
        - GF_SECURITY_ADMIN_PASSWORD=hdeAga76VG6ga7plZ1
        - GF_SECURITY_ADMIN_USER=sitespeedio
        - GF_AUTH_ANONYMOUS_ENABLED=true
        - GF_USERS_ALLOW_SIGN_UP=false
        - GF_USERS_ALLOW_ORG_CREATE=false
        - GF_INSTALL_PLUGINS=grafana-piechart-panel
      volumes:
        - /app/performance/grafana/etc/grafana.ini:/etc/grafana/grafana.ini
        - grafana:/var/lib/grafana
      restart: always
    graphite:
      image: sitespeedio/graphite:1.1.10-3
      hostname: graphite
      ports:
        - "2003:2003"
        - "8080:8080"
      restart: always
      volumes:
        # In production you should configure/map these to your container
        # Make sure whisper and graphite.db/grafana.db lives outside your containerr
        # https://www.sitespeed.io/documentation/sitespeed.io/graphite/#graphite-for-production-important
        - whisper:/opt/graphite/storage/whisper
        # Download an empty graphite.db from https://github.com/sitespeedio/sitespeed.io/tree/main/docker/graphite
        # - /absolute/path/to/graphite/graphite.db:/opt/graphite/storage/graphite.db
        #
        # And put the configuration files on your server, configure them as you need
        # Download from https://github.com/sitespeedio/docker-graphite-statsd/tree/main/conf/graphite
        # - /absolute/path/to/graphite/conf/storage-schemas.conf:/opt/graphite/conf/storage-schemas.conf
        # - /absolute/path/to/graphite/conf/storage-aggregation.conf:/opt/graphite/conf/storage-aggregation.conf
        # - /absolute/path/to/graphite/conf/carbon.conf:/opt/graphite/conf/carbon.conf
    grafana-setup:
      image: sitespeedio/grafana-bootstrap:24.5.0
      links:
        - grafana
      environment:
        - GF_PASSWORD=hdeAga76VG6ga7plZ1
        - GF_USER=sitespeedio
volumes:
    influxdb:
    grafana:
    whisper:

```

``` bash
$ docker-compose up -d
```

### Telegraf 설정

- Telegraf 토큰 발급
 - Telegraf > Setup Instructions > Generate NewToken 

``` bash
$ docker container exec -it teleraf bash

$ export INFLUX_TOKEN=발급받은토큰
```

### Grafana 설정

![grafana1](image/grafana-influxsetup.PNG)


- Grafana DataSource 설정
 - Plugin > influxDB 등록


### influxDB 설정

![influxdb-bucket](image/influxdb-Bucket1.PNG)

Bucket 생성

생성한 Bucket 토큰 발급



### Telegraf 시스템 대시보드 템플릿

- 윈도우 용 템플릿
  - https://github.com/influxdata/community-templates/tree/master/windows_system

- 리눅스 용 템플릿
  - https://github.com/influxdata/community-templates/tree/master/linux_system
  - https://raw.githubusercontent.com/influxdata/community-templates/master/linux_system/linux_system.yml

**주의** grafana에 telegraf 대시보드를 등록할 때 해당 대시보드가 influxdb 2버전 이상의 대시보드를 등록 해야한다.

### influxDB 템플릿
  
- https://github.com/influxdata/community-templates/tree/master/apache_jmeter

### grafana 대시보드

![grafana-dashboard](image/grafana-dashboard1.PNG)

- https://grafana.com/grafana/dashboards/17472-jmeter-test-results-influxdb2-standart-backend-listener/

- https://grafana.com/grafana/dashboards/13644-jmeter-load-test-org-md-jmeter-influxdb2-visualizer-influxdb-v2-0-flux/
  - java 11이상

- https://grafana.com/grafana/dashboards/5496-apache-jmeter-dashboard-by-ubikloadpack/


### sitespeedio grafana 대쉬보드 생성

docker run --rm -v "/home/docker/sitespeedio/result:/sitespeed.io" --network host sitespeedio/sitespeed.io:26.1.0 --graphite.host=192.168.151.11 https://www.naver.com --slug test001 --graphite.addSlugToKey true


## jmeter Settings

Backend Listener 추가 > InfluxDB 정보 입력

![jmeter1.png](image/jmeter1.png)
