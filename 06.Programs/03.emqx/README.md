# EMQX — MQTT broker

The MQTT broker between the MES / SCADA and the Bay 11 [Node-RED gateway](../nodered/README.md). Node-RED
connects to it at `172.16.50.69:1883` (MQTT 3.1.1, no username/password, no TLS), using the client IDs
`loadingbay_prod_oder` and `weighbridge_nodered`.

| | |
|---|---|
| Image | `emqx/emqx-enterprise:6.1.1` |
| Node name | `vflow-emqx@vc-mqtt` |
| Dashboard | `http://<host>:18083` |
| Data / logs | `emqx/emqx_data/` → `/opt/emqx/data`, `emqx/emqx_log/` → `/opt/emqx/log` (not in git) |

## Ports

| Port | Use |
|---|---|
| 1883 | MQTT (used by Node-RED) |
| 8883 | MQTT over TLS |
| 8083 | MQTT over WebSocket |
| 8084 | MQTT over secure WebSocket |
| 18083 | Dashboard / REST API |

## Running

The container runs as user `emqx` (uid 1000) and needs to write to both folders. On a host where the repo
is owned by root (the gateway), hand the folders over once before the first start:

```bash
mkdir -p emqx/emqx_data emqx/emqx_log
sudo chown -R 1000:1000 emqx/emqx_data emqx/emqx_log
docker compose -f emqx/docker-compose.yml up -d
docker compose -f emqx/docker-compose.yml ps        # wait for "healthy"
```

If the container keeps restarting with `Permission denied` under `/opt/emqx/data`, the ownership step was
skipped.

The first dashboard login is `admin` / `public`. Change the password straight away.

Out of the box EMQX accepts anonymous clients, which is what the Node-RED flow expects. If you add
authentication in the dashboard, add the same credentials to both `mqtt-broker` nodes in the flow.

## Moving an existing broker onto this compose

The old setup mounted `/root/data` and `/root/log`. To keep its retained messages, sessions and dashboard
config, stop it and copy the data over before starting this one:

```bash
docker stop emqx && docker rm emqx
sudo cp -a /root/data/. emqx/emqx_data/
sudo chown -R 1000:1000 emqx/emqx_data emqx/emqx_log
docker compose -f emqx/docker-compose.yml up -d
```

This only works because the node name (`EMQX_NAME`, `EMQX_HOST`) is unchanged. Don't edit those in
`docker-compose.yml` on a broker that already has data.
