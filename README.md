# Fixing LiteSpeed Cache QUIC.cloud Activation Loop (OpenLiteSpeed + Zoraxy Proxy + Cloudflare)

If you are running WordPress on OpenLiteSpeed behind a local reverse proxy like [Zoraxy](https://github.com/tobychui/zoraxy) and Cloudflare, you may encounter an endless `302 Found` redirect loop or a `tool/wp_rest_echo` error when trying to link the LiteSpeed Cache plugin to QUIC.cloud.

## The Problem: NAT Hairpinning & Reverse Proxy Loopbacks
When you click "Link to QUIC.cloud" or "Request Domain Key", the QUIC.cloud servers attempt to verify your domain by pinging your WordPress REST API. Simultaneously, your WordPress server attempts a self-test by curling its own public domain. 

Because your OpenLiteSpeed backend is sitting behind a local reverse proxy (Zoraxy), it asks your router for your public domain. The router intercepts the request from inside the network, gets confused by the NAT loopback, and drops the connection. OpenLiteSpeed reports back to QUIC.cloud that it cannot reach its own API, and the handshake fails silently.

## The Solution
You need to explicitly tell OpenLiteSpeed how to route traffic to the proxy, and tell OpenLiteSpeed to trust the proxy's IP.

### Architecture Example Used in this Guide:
* **Cloudflare:** Public DNS / CDN
* **Reverse Proxy (Zoraxy):** `192.168.1.2`
* **Backend Web Server (OpenLiteSpeed):** `192.168.1.3`

---

### Step 1: Fix Local Loopback (`/etc/hosts`)
Force the backend server to resolve your domain directly to the reverse proxy, bypassing your router's NAT entirely.

1. SSH into your OpenLiteSpeed backend server (`192.168.31.3`).
2. Edit the hosts file:
   ```bash
   sudo nano /etc/hosts
   ```
3.Add your domain and point it to your Zoraxy Proxy IP:
```bash
    192.168.31.2 yourdomain.com [www.yourdomain.com](https://www.yourdomain.com)
```
4.Save and exit.

Test it by running curl -I https://yourdomain.com. It should successfully hit your proxy and return an HTTP status code.
