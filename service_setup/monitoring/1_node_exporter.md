# phillymesh.net Monitoring - Node Exporter

Philly Mesh is running a monitoring suite composed of several pieces of software. This document covers the installation of [node-exporter](https://github.com/prometheus/node_exporter), an exporter for machine metrics. node-exporter is designed to be used along with [Prometheus](https://prometheus.io/), an open-source monitoring system. 

## Set Up node-exporter

1. The first step will be to download the **node-exporter** software. Navigate to [ https://github.com/prometheus/node_exporter/releases/]( https://github.com/prometheus/node_exporter/releases/) and download the latest release for your system's architecture. **node-exporter** seems to work well on a variety platforms, and happily plugs away on Raspberry Pis and the like, causing no noticeable slow-downs.

	```
	# wget https://github.com/prometheus/node_exporter/releases/download/v0.13.0/node_exporter-0.13.0.linux-amd64.tar.gz
	```

	Now, we can extract the contents of the archive:

	```
	# tar -xzf ../Downloads/node_exporter-0.13.0.linux-amd64.tar.gz
	```

1. Now, we can run `node_exporter` with one command:

	```
	# ./node_exporter-0.13.0.linux-amd64/node_exporter
	```

	**node-exporter** will start listening on port 9100, on all interfaces. You can view a web console here, [http://localhost:9100/metrics](http://localhost:9100/metrics). We will consider these metrics to be public, so it will be necessary to punch a hole in the firewall on any machine that will not have the **Prometheus** server also installed on it.

	**node-exporter** will terminate if we close our console session, so you may wish to run it in screen/tmux for now.

## TODO

**TODO**: See about writing an init script to have this run on machine boot. This could be useful, [https://blog.svedr.in/posts/prometheus-quick-start.html](https://blog.svedr.in/posts/prometheus-quick-start.html)
