---
layout: post
title:  "Testing Prometheus for monitoring and a bit of routing"
date:   2016-12-14 23:48:31 +0100
categories: docker prometheus debian bird grafana
---
Later, I want to have a complete solution with monitoring. So today I investigated some monitoring solutions, reading lengthy threads of text. To summarize, admins are favoring cloud solutions like DataDog and the only solution on premise that has no opposition is Prometheus. So I decided to start a container with it to try it myself. 
Except monitoring, I modified the configuration of bird routing daemon. After my migration from libvirt routing network, I noticed the local network is not announced. The solution was to hardcode the value (not very impacting as I have only 2 nodes).

## Prometheus
Central prometheus poll agents installed on nodes. Agents present metrics to central and all metrics are collected in 1 connection. By default, polling is done every 15 seconds.

- The Prometheus container is hosted on quay.io, as usual with containers, starting it is trivial
{% highlight shell %}
[root@db-sc1 ~]# docker run -d \
        -v /data/prometheus:/prometheus \
        -v /data/pro_config/prometheus.yml:/etc/prometheus/prometheus.yml \
        -p 9090:9090 \
        --name prometheus \
        quay.io/prometheus/prometheus
{% endhighlight %}

The port 9090 serve a local website, used to view and query the time series database. The query language is quite powerful, it can be used to apply mathematical functions to series (sum, nth percentile,...).

Once the container is running, I need to setup agents. On Debian, the agent is only available on backports.

- Install backports
{% highlight shell %}
[root@db-sc1 ~]# echo 'deb http://mirrors.online.net/debian jessie-backports main' >> /etc/apt/sources.list.d/debian_backports.list
[root@db-sc1 ~]# apt -t jessie-backports install prometheus-node-exporter
{% endhighlight %}

- Add the iptables rule
{% highlight shell %}
[root@db-sc1 ~]# echo '-A INPUT -i br10 -p tcp -m tcp --dport 9100 -j ACCEPT' >> /etc/iptables.up.rules
[root@db-sc1 ~]# iptables-restore < /etc/iptables.up.rules
{% endhighlight %}

- On the server part, I need to indicate what nodes the server should poll. This is done in the prometheus.yml 

{% highlight yaml %}
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  - job_name:       'dediboxes'

    # Override the global default and scrape targets from this job every 5 seconds.
    #scrape_interval: 5s

    static_configs:
      - targets: ['172.16.4.1:9100', '172.16.3.1:9100']
        labels:
          group: 'production'
{% endhighlight %}

## Grafana

For visualisation support, one can use Grafana. Again, I used a container to test the app.

{% highlight shell %}
[root@ubuntu-docker ~]# docker run -d \
        --name grafana \
        -p 3000:3000 \
        -e "GF_SECURITY_ADMIN_PASSWORD=mysecretpass" \
        -v /data/grafana_db:/var/lib/grafana \
        grafana/grafana
{% endhighlight %}

Once logged into grafana, add prometheus as data source and enjoy viewing metrics

## Bird
- Force the local connected subnet for the daemon
{% highlight conf %}
protocol device {
        scan time 60;
        primary 172.16.4.0/24;
}
{% endhighlight %}

- Restart bird

{% highlight shell %}
[root@db-xc1 29 ~]# service bird restart
{% endhighlight %}
