#
# Node-red image with InfluxDB node
#

# Official Node-RED container
FROM nodered/node-red:2.1.4-1

LABEL maintainer="Philippe Vanhaesendonck"

# Install TTN nodes
RUN npm install node-red-contrib-influxdb@0.4.0

# User configuration directory volume
VOLUME ["/data"]

# Expose the listening port of node-red
EXPOSE 1880
