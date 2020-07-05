#
# Node-red image with TTN and InfluxDB nodes
#

# Official Node-RED container
FROM nodered/node-red:1.1.0-12

LABEL maintainer="Philippe Vanhaesendonck"

# Install TTN nodes
RUN npm install node-red-contrib-influxdb@0.4.0 node-red-contrib-ttn@2.0.5

# User configuration directory volume
VOLUME ["/data"]

# Expose the listening port of node-red
EXPOSE 1880
