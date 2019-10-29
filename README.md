# nanog77-tsdb-tutorial

The goal of hthis totorial is to help you get started with Moderm Timeseries Database and Grafana. In this repository you'll find:
- A pre-build topology of 6 devices (4xIOSXR, 1,EOs, 1xJUNOS) that you can quickly setup in [Tesuto]
- An ansible project to configure all devices and servers
- A Step by Step guide to: Collect interface and bgp statistics from the network devices using gNMI, run a prometheus and a Grafana server
- Some example of dashboards and queries for Grafana

# Tutorial

## 1- Setup a gNMI collector per device

gNMI is alreayd configured on all devices, to start collecting  data, we'll be using Telegraf. Telegraf is a very flexible collector that support many input plugins and many databases (output plugins). For this tutorial we'll be using the input plugin `cisco_telemetry_gnmi` that is shipping with telegraf.
The playbook `pb.telegraf.yaml` will start one instance of telegraf per device on the jumphost. Each instance will listen on a specific port (defined in the inventory file)
```
ansible-playbook pb.telegraf.yaml
```

To verify that everything is running as expected
```
jumphost:~$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                  NAMES
1207af06d14d        telegraf:1.12.3     "/entrypoint.sh tele…"   23 hours ago        Up About an hour    8092/udp, 8125/udp, 8094/tcp, 0.0.0.0:9003->9003/tcp   telegraf-san-antonio
ea4064b4fe26        telegraf:1.12.3     "/entrypoint.sh tele…"   23 hours ago        Up About an hour    8092/udp, 8125/udp, 8094/tcp, 0.0.0.0:9004->9004/tcp   telegraf-el-paso
0f3a0fd55465        telegraf:1.12.3     "/entrypoint.sh tele…"   23 hours ago        Up About an hour    8092/udp, 8125/udp, 8094/tcp, 0.0.0.0:9001->9001/tcp   telegraf-amarillo
1d6a7954c7d5        telegraf:1.12.3     "/entrypoint.sh tele…"   23 hours ago        Up About an hour    8092/udp, 8125/udp, 8094/tcp, 0.0.0.0:9012->9012/tcp   telegraf-houston
da5d924a1bab        telegraf:1.12.3     "/entrypoint.sh tele…"   23 hours ago        Up About an hour    8092/udp, 8125/udp, 8094/tcp, 0.0.0.0:9011->9011/tcp   telegraf-dallas
f83660068e29        telegraf:1.12.3     "/entrypoint.sh tele…"   23 hours ago        Up About an hour    8092/udp, 8125/udp, 8094/tcp, 0.0.0.0:9002->9002/tcp   telegraf-austin
```

Connect to `http://<jumphost_address>:<gnmi_port_telegraf>/metrics` for each device

## 2- Start a prometheus server on the jumphost

To start a quick prometheus server, you need to download the binary for your platform and start it locally
```
# Download & untar
wget https://github.com/prometheus/prometheus/releases/download/v2.13.1/prometheus-2.13.1.linux-amd64.tar.gz
tar -xzf prometheus-2.13.1.linux-amd64.tar.gz
cp prometheus-2.13.1.linux-amd64/prometheus .
```

Create base configuration `vi ~/prom.conf`
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
```

```
./prometheus --config.file="prom.conf"
```
At this point the server should be running and you can access its web interface at `http://<jumphost_address>:9090/targets`

To collect the information from the network devices (via telgraf) we need to update the configuration file to add a list of targets
```yaml
  - job_name: 'gnmi'
    scrape_interval: 30s
    static_configs:
      - targets: 
        - 'localhost:9001'
        - 'localhost:9002'
        - 'localhost:9003'
        - 'localhost:9004'
        - 'localhost:9011'
        - 'localhost:9012'
```

Check that all targets are working properly in the Web interface `http://<jumphost_address>:9090/targets`
In the web interface you can start to run some query

- Show ingress octets counters for all interfaces : `interface_state_counters_in_octets`
- Show ingress octets counters for one device : `interface_state_counters_in_octets{device="amarillo"}`
- Show ingress octets rate for one device : `rate(interface_state_counters_in_octets{device="amarillo"}[2m])`
- Total ingress traffic for one device : `sum(rate(interface_state_counters_in_octets{device="amarillo"}[2m])) by (device)`

### Add more tags 

Enable the flag `add_interface_role` in `group_vars/all.yaml`, it will update the telegraf configuration to add new tags per interface
```
ansible-playbook pb.telegraf.yaml
```
Check if these tags are present in the UI



## 3- Install grafana 

```
wget https://dl.grafana.com/oss/release/grafana_6.4.3_amd64.deb
sudo dpkg -i grafana_6.4.3_amd64.deb
sudo /bin/systemctl start grafana-server
```

### Install discrete panel in Grafana

```
sudo grafana-cli plugins install natel-discrete-panel
sudo service grafana-server restart
```

# Setup the environment

## Create the topology in Tesuto

- Create an account on tesuto.com
- Accept the licenses
- Upload a SSH Key
- Import the topology file `tesuto.export`

## Configure everything
```
ansible-playbook pb.config.network.yaml
ansible-playbook pb.config.linux.yaml
ansible-playbook pb.prometheus.yaml
```




# K6 

k6 run --vus 100 --no-vu-connection-reuse --duration 300s test.js






# Generate some Traffic

Iperf, iperf3, NUTTCP, NGINX and K6 are installed on both `srv-dal` and `srv-hou`

## NUTTCP 
```
# From srv-hou, start the server
nuttcp -S
```

```
# From srv-dal
nuttcp -N 128 -T 600 10.0.120.10
```

```
nuttcp -N 128 -T 600 -R 1000p 10.0.110.10
```

## K6 

from either (or both) server
```
k6 run --vus 50 --no-vu-connection-reuse --duration 600s /tmp/test.j2
```

## Iperf3 

```
Start iperf3 in Server mode
iperf3 -s -D
```

```
iperf3 -P 64 -b 50K -l 100 -M 500 -t 600 -c 10.0.110.10 -u
iperf3 -P 64 -b 1k -l 100 -M 500 -t 600 -c 10.0.110.10 -u


iperf3 -P 32 -b 1K -l 100 -M 500 -t 600 -c 10.0.120.10 -u
iperf3 -P 32 -b 100M -l 100 -M 1400 -t 600 -c 10.0.120.10 -u
```

Links 
https://discuss.aerospike.com/t/benchmarking-throughput-and-packet-count-with-iperf3/2791
https://support.cumulusnetworks.com/hc/en-us/articles/216509388-Throughput-Testing-and-Troubleshooting#client_commands


