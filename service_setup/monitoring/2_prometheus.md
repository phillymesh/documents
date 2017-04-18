# phillymesh.net Monitoring - Prometheus

Philly Mesh is running a monitoring suite composed of several pieces of software. This document covers the installation of [Prometheus](https://prometheus.io/), an open-source monitoring system. Prometheus (or Prom as I may call it) works by connecting to machines running various Exporters, which constantly report metrics or monitoring information for specific services or system info.


## Set Up Prometheus

1. The first step will be to download **Prometheus**. Navigate to [  https://github.com/prometheus/prometheus/releases/](  https://github.com/prometheus/prometheus/releases/) and download the latest release for your system's architecture.

	```
	# wget wget https://github.com/prometheus/prometheus/releases/download/v1.5.2/prometheus-1.5.2.linux-amd64.tar.gz
	```

	Now, we can extract the contents of the archive:

	```
	tar -xzf prometheus-1.5.2.linux-amd64.tar.gz
	```

1. Now we will modify the `prometheus.yml` configuration file:

	```
	# nano prometheus-1.5.2.linux-amd64/prometheus.yml
	```

	Replace the contents with the following. Note the `targets`, consisting of machines running **node-exporter.** Valid targets can be inserted using FQDNs, localhost, or IPv4/IPv6 addresses as needed (though you MUST specify port). Prometheus has no issue connecting to nodes over Hyperboria.

	```
	scrape_configs:
 	- job_name: "node"
	  scrape_interval : "15s"
	  static_configs:
	  - targets: ['h.peer0.famicoman.phillymesh.net:9100', 'h.peer1.famicoman.phillymesh.net:9100', 'h.peer2.famicoman.phillymesh.net:9100', 'h.peer3.famicoman.phillymesh.net:9100', 'h.peer4.famicoman.phillymesh.net:9100']
	```


1. Now, we can start **Prometheus** in the background.

	```
	# cd prometheus-1.5.2.linux-amd64
	# nohup ./prometheus > prometheus.log 2>&1 &
	```

	**node-exporter** will start listening on port 9090, on all interfaces. 

	Your Prometheus server provides a basic web interface. In your browser, you can navigate to [http://localhost:9090/targets](http://localhost:9090/targets) to view the status of the nodes you are tracking.

## TODO

**TODO**: See about writing an init script to have this run on machine boot. This looks helpful, [https://blog.svedr.in/posts/prometheus-quick-start.html](https://blog.svedr.in/posts/prometheus-quick-start.html)

**TODO**: Some sort of job that creates a new .yml file with an updated targets list. Could be done in a script or something. Also, you can restart prom remotely, `curl -s -XPOST localhost:9090/-/reload`.
