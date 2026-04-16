# EMQX on Tencent Cloud CVM

This directory is a safe bootstrap for a single-node EMQX broker on a Tencent Cloud CVM.

It intentionally starts with these defaults:

- `18083` (Dashboard) only listens on `127.0.0.1`
- `1883` (plain MQTT) only listens on `127.0.0.1`
- `8883` (MQTTS) only listens on `127.0.0.1`

That lets you bring EMQX up first, then configure authentication and authorization before exposing any MQTT port to the Internet.

## Files

- `compose.yaml`: single-node EMQX 6.2.0 deployment
- `.env.example`: copy to `.env` and edit the values

## Assumptions

- Ubuntu 22.04 or 24.04 on Tencent Cloud CVM
- Docker Engine and Docker Compose plugin are installed
- You can SSH into the server

## 1. Install Docker on Ubuntu

Follow Docker's official Ubuntu installation guide:

- https://docs.docker.com/engine/install/ubuntu/

If you want the exact official `apt` repository method, use the commands from that page.

## 2. Copy this directory to the server

Example:

```bash
scp -r emqx ubuntu@<YOUR_SERVER_IP>:/opt/
```

Then log in:

```bash
ssh ubuntu@<YOUR_SERVER_IP>
cd /opt/emqx
cp .env.example .env
```

Edit `.env` and set a strong `EMQX_DASHBOARD_PASSWORD`.

## 3. Start EMQX

```bash
docker compose up -d
docker compose ps
docker logs --tail=100 emqx
```

You should see the container in `Up` state.

## 4. Verify local listeners on the server

```bash
ss -lntp | grep -E ':(1883|8883|18083)\b'
```

Expected during bootstrap:

- `127.0.0.1:1883`
- `127.0.0.1:8883`
- `127.0.0.1:18083`

## 5. Open the Dashboard safely with SSH tunnel

From your own computer:

```bash
ssh -L 18083:127.0.0.1:18083 ubuntu@<YOUR_SERVER_IP>
```

Then open:

- http://127.0.0.1:18083

Login:

- username: `admin`
- password: the value you set in `.env`

## 6. Configure access control before exposing 8883

Important: EMQX does not enable authentication by default. Do not expose `8883` publicly until you finish this step.

In the Dashboard:

1. Go to `Access Control -> Authentication`
2. Click `Create`
3. Choose `Password-Based`
4. Choose `Built-in Database`
5. Create at least one test device account

Then configure authorization:

1. Go to `Access Control -> Authorization`
2. Click `Create`
3. Choose `Built-in Database`
4. Add rules so each device only uses its own topics

Suggested first topic convention:

- publish: `device/<device_id>/up/#`
- subscribe: `device/<device_id>/down/#`

## 7. Expose only 8883 to the Internet

After authentication and authorization are ready:

1. Edit `.env`
2. Change:

```dotenv
EMQX_BIND_8883=0.0.0.0
```

3. Restart EMQX:

```bash
docker compose up -d
ss -lntp | grep 8883
```

At this point, keep `1883` and `18083` on `127.0.0.1`.

## 8. Add Tencent Cloud security group rules

Add inbound rules for:

- `22/tcp`: your current public IP only
- `8883/tcp`: `0.0.0.0/0` only after auth is enabled

Do not expose:

- `1883/tcp`
- `18083/tcp`

Tencent Cloud security group docs:

- https://cloud.tencent.com/document/product/213/112614

## 9. First external smoke test

Use MQTTX CLI from your own computer after `8883` is public.

Subscribe:

```bash
mqttx sub -h <YOUR_SERVER_IP_OR_DOMAIN> -p 8883 --protocol mqtts -u <USERNAME> -P <PASSWORD> -t 'device/test/down/#' --insecure
```

Publish:

```bash
mqttx pub -h <YOUR_SERVER_IP_OR_DOMAIN> -p 8883 --protocol mqtts -u <USERNAME> -P <PASSWORD> -t 'device/test/up/status' -m '{"hello":"world"}' --insecure
```

`--insecure` is only for the first smoke test if your server certificate name does not match the host you used, or if you have not replaced the default certificate yet.

## 10. Next hardening step

Before your ESP board connects over the Internet, replace the default TLS certificate on listener `8883` with your own certificate and domain.

Official references:

- EMQX Docker image overview and persistence notes:
  https://hub.docker.com/r/emqx/emqx
- EMQX getting started and default Dashboard login:
  https://docs.emqx.com/en/emqx/latest/getting-started/getting-started.html
- EMQX authentication overview:
  https://docs.emqx.com/en/emqx/latest/access-control/authn/authn.html
- EMQX built-in database authentication:
  https://docs.emqx.com/en/emqx/latest/access-control/authn/mnesia.html
- EMQX built-in database authorization:
  https://docs.emqx.com/en/emqx/latest/access-control/authz/mnesia.html
- EMQX TLS listener guide:
  https://docs.emqx.com/en/emqx/latest/network/emqx-mqtt-tls.html
