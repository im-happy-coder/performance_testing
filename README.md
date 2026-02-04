
# 웹사이트 성능테스트 및 시각화 시스템 구축

## Diagram

![performance-diagram](https://github.com/im-happy-coder/performance_testing/blob/main/img/jmeterAndPerformance.png?raw=true)

### 목적

- K8S HPA 설정, EC2/RDS 타입 선정, 병목 구간 분석을 위해 정확한 성능을 측정하여 판단할 필요가 있다고 생각했습니다
이에 단순 부하 발생이 아닌, 애플리케이션 도메인별 부하 특성 파악하여 시스템 자원 사용률과 응답 지표의 상관관계 분석을 목표로 성능 테스트 환경을 직접 구성했습니다.

---

### 구성 개요

- JMeter: 도메인별 부하 시나리오 실행
- Telegraf: CPU, Memory, I/O, Network 자원 수집
- InfluxDB: 테스트 결과 및 시스템 메트릭 저장
- Grafana: 응답 시간과 자원 사용률의 상관관계 시각화
- Sitespeedio: 프론트엔드 로딩 및 리소스 성능 측정

---

### 학습을 통해 얻은 점

- 요청 수 증가에 따라 CPU 사용률보다 DB 응답 지연이 먼저 병목이 되는 구간 확인
- 단순 CPU 기준 HPA 설정은 실제 사용자 응답 지연을 반영하지 못할 수 있음을 체감
- 성능 테스트 결과를 통해 인프라 리소스 증설보다 쿼리/캐시/아키텍처 개선이 우선일 수 있음을 인지

---

### 실무 관점

- JMeter 기반 부하 테스트 설계
- RDS 병목 원인 분석
- Auto Scaling 조건 재정의 등을 판단하는 데 참고 기준으로 활용했습니다.

### ResultView

- Jmeter Script
  
![jmeter1](https://github.com/im-happy-coder/performance_testing/blob/main/img/jmeter-sample1.PNG?raw=true)


- Grafana Load Test Dashboard

![grafana-result](https://github.com/im-happy-coder/performance_testing/blob/main/img/grafana-dashboard2.PNG?raw=true)


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

![grafana1](https://github.com/im-happy-coder/performance_testing/blob/main/img/grafana-influxsetup.PNG?raw=true)


- Grafana DataSource 설정
 - Plugin > influxDB 등록


### influxDB 설정

![influxdb-bucket](https://github.com/im-happy-coder/performance_testing/blob/main/img/influxdb-Bucket1.PNG?raw=true)

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

![grafana-dashboard](https://github.com/im-happy-coder/performance_testing/blob/main/img/grafana-dashboard1.PNG?raw=true)

- https://grafana.com/grafana/dashboards/17472-jmeter-test-results-influxdb2-standart-backend-listener/

- https://grafana.com/grafana/dashboards/13644-jmeter-load-test-org-md-jmeter-influxdb2-visualizer-influxdb-v2-0-flux/
  - java 11이상

- https://grafana.com/grafana/dashboards/5496-apache-jmeter-dashboard-by-ubikloadpack/


### sitespeedio grafana 대쉬보드 생성

docker run --rm -v "/home/docker/sitespeedio/result:/sitespeed.io" --network host sitespeedio/sitespeed.io:26.1.0 --graphite.host=192.168.151.11 https://www.naver.com --slug test001 --graphite.addSlugToKey true


## jmeter Settings

Backend Listener 추가 > InfluxDB 정보 입력

![jmeter1.png](https://github.com/im-happy-coder/performance_testing/blob/main/img/jmeter1.png?raw=true)
