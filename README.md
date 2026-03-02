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
3. Add your domain and point it to your Zoraxy Proxy IP:
```bash
    192.168.31.2 yourdomain.com [www.yourdomain.com](https://www.yourdomain.com)
```
4. Save and exit.

Test it by running curl -I https://yourdomain.com. It should successfully hit your proxy and return an HTTP status code.

### Step 2: Configure OpenLiteSpeed to Trust the Proxy
OpenLiteSpeed needs to know that the Zoraxy IP is a trusted proxy so it correctly processes the forwarded client IPs and security headers.

1. Log into your OpenLiteSpeed WebAdmin Console (usually `https://your-backend-ip:7080`).
2. Navigate to **Server Configuration** > **Security**.
3. Under **Access Control** > **Allowed List**, add your proxy IP with a trailing `T` (for Trusted):
   ```text
   192.168.31.2T
4. Navigate to Server Configuration > General.
5. Set Use Client IP in Header to Trusted IP Only.
6. Perform a Graceful Restart of OpenLiteSpeed.

## Step 3: Clear LiteSpeed Cache Overrides
If you previously attempted to force an IP address in the WordPress plugin settings, it will break this new internal routing.

1. Go to your WordPress Dashboard.

2. Navigate to LiteSpeed Cache > General.

3. Ensure the Server IP field is completely blank.

4. Click Save Changes.

## Step 4: Link to QUIC.cloud
Click the Link to QUIC.cloud button again. The server will now properly route the self-test through the proxy, complete the handshake, and generate your domain key!


***

Now that the proxy routing is locked in and your cache is linked, would you like me to walk you through double-checking the Cloudflare SSL/TLS settings to make sure they are set to "Full (Strict)"?
