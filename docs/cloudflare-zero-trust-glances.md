# Infrastructure Documentation: Securing Glances with Cloudflare Zero Trust & GitHub OAuth

This document provides a comprehensive, step-by-step guide on how the Glances dashboard hosted on Coolify (`glances.nmcyber.com`) was secured using Cloudflare Zero Trust (Access) and GitHub OAuth authentication.

Following this architecture ensures your backend server ports remain strictly closed to the public internet, using a secure outbound Cloudflare Tunnel (`cloudflared`) to route verified traffic.

---

## Architecture Overview

```
[ User Browser ] ---> https://glances.nmcyber.com
                            |
                            v
               [ Cloudflare Access Edge ] <---> [ GitHub OAuth Validation ]
                            |
                    (Secure Tunnel Pipe)
                            |
                            v
       [ cloudflared Docker Container (goofy_chatelet) ]
                            |
                   (Internal Docker Network)
                            |
                            v
       [ Coolify Glances Container ] : Port 61208
```

1. **User Connection**: User requests `https://glances.nmcyber.com`.
2. **Authentication Barrier**: Cloudflare Access intercepts the request and verifies the user via GitHub OAuth.
3. **Secure Pipeline**: Once authorized, traffic is passed down an outbound Cloudflare Tunnel (`cloudflared`) running as a Docker container on the host machine.
4. **Target Destination**: The tunnel container passes traffic over an isolated, custom Docker network directly to the target Coolify Glances container on port `61208`.

---

## Phase 1: GitHub OAuth Application Setup

To allow Cloudflare to authenticate users through GitHub, a dedicated Developer OAuth Application must exist.

1. Navigate to **GitHub** > **Settings** > **Developer Settings** > **OAuth Apps**.
2. Click **Register a new application** and configure the fields exactly as follows:

   | Field | Value |
   |---|---|
   | **Application Name** | `Cloudflare Zero Trust` |
   | **Homepage URL** | `https://fancy-cell-829b.cloudflareaccess.com` |
   | **Authorization callback URL** | `https://fancy-cell-829b.cloudflareaccess.com/cdn-cgi/access/callback` |

   > **Note**: Ensure there are no trailing slashes or spaces at the end of the callback URL.

3. Click **Register application**.
4. Copy and securely store the generated **Client ID**.
5. Click **Generate a new client secret**. Copy and securely store this value immediately *(it will not be shown again)*.

---

## Phase 2: Configure Cloudflare Authentication & Access Policies

### 1. Add GitHub as an Identity Provider

1. Log into your **Cloudflare Zero Trust Dashboard**.
2. Navigate to **Integrations** > **Identity providers** on the left menu sidebar.
3. Click **Add an identity provider** and select **GitHub**.
4. Input your configuration details:
   - **Name**: `GitHub`
   - **Client ID**: *(Paste the Client ID from Phase 1)*
   - **Client Secret**: *(Paste the Client Secret from Phase 1)*
5. Save the configuration. *(Note: You can verify the exact Team Name URL string under **Reusable components** > **Custom pages** if needed.)*
6. Click **Test** to ensure a successful callback correlation with GitHub.

### 2. Create the Access Application & Explicit Security Policy

> ⚠️ **CRITICAL SECURITY NOTE**: Simply adding GitHub as a provider allows *any* GitHub user to authenticate. You must create an explicit Access Policy rule to lock it down to specific authorized accounts. Without this rule, authenticated but unlisted users will be blocked with an **"Un-authorized"** message.

1. Navigate to **Access** > **Applications** and click **Add an application** > **Self-hosted**.
2. **Application Configuration**:
   - **Application Name**: `Glances Dashboard`
   - **Session Duration**: Set to your preference (e.g., `24 Hours`)
   - **Application URL**: Subdomain: `glances` | Domain: `nmcyber.com`
3. **Identity Providers**: Uncheck all providers *except* **GitHub** to enforce it as the singular login method. Click **Next**.
4. **Policy Configuration (The Access Rule)**:
   - **Rule Name**: `Allow Authorized Users`
   - **Action**: `Allow`
   - **Assignee (Include Block)**:
     - **Selector**: Select **Emails** from the dropdown menu.
     - **Value**: Input your designated primary GitHub email address (e.g., your personal email).
     - *(Optional)* To authorize additional users, click **Add include**, select **Emails**, and add their primary GitHub email addresses.
     - *(Alternative Selector)* You can also select **GitHub Organization** or **GitHub Teams** if managing access via a shared GitHub organization.
5. Advance through the remaining defaults and click **Add Application**.

---

## Phase 3: Setup the Cloudflare Tunnel on the Server

Instead of modifying firewall records or exposing host server port `61208` to the public web, traffic is tunneled through a native daemon container.

1. In the Zero Trust dashboard, go to **Networks** > **Tunnels** and click **Create a tunnel**.
2. Choose **Cloudflare (Recommended)**, name the tunnel (e.g., `Coolify-Server`), and save.
3. On the deployment page, select **Docker** as your environment.
4. Copy the initialization command containing your unique token block.
5. SSH into your target server and append the `-d` flag right after `docker run` to run the daemon detached in the background:

   ```bash
   sudo docker run -d cloudflare/cloudflared:latest tunnel --no-autoupdate run --token <YOUR_UNIQUE_SECRET_TOKEN>
   ```

6. Check the terminal to confirm the running instance image and name:

   ```bash
   sudo docker ps --format "table {{.Image}}\t{{.Names}}"
   ```

### A Note on Docker's Random Container Names

When you run a container without an explicit `--name` flag, Docker automatically generates a quirky two-word name so it can track the container internally. It combines a **random adjective** with the **last name of a famous scientist, engineer, hacker, or mathematician**.

In this case, `goofy_chatelet` breaks down as:

- **`goofy`** — a random adjective chosen from Docker's internal pool.
- **`chatelet`** — a reference to **Émilie du Châtelet**, a brilliant 18th-century French natural philosopher and mathematician. She translated Isaac Newton's *Principia Mathematica* into French (still the standard translation today) and accurately deduced that kinetic energy is proportional to velocity squared:

$$E_k \propto v^2$$

So when the original Cloudflare installation command was run without a `--name` flag, Docker noticed and dubbed the tunnel container `goofy_chatelet` in her honor.

#### Want to give it a cleaner name?

If you prefer a recognizable name over a randomly assigned one, you can replace the container:

1. Stop and delete the current one:

   ```bash
   sudo docker rm -f goofy_chatelet
   ```

2. Run it again with an explicit name (e.g., `cf-tunnel`):

   ```bash
   sudo docker run -d --name cf-tunnel --network k0k8gwsgw4wsowcssk0s8wo0 cloudflare/cloudflared:latest tunnel --no-autoupdate run --token <YOUR_TOKEN_HERE>
   ```

> **Note**: If you already ran the `docker network connect` command and your dashboard is working smoothly, you don't have to touch a thing — you can leave Madame du Châtelet running your network pipeline in the background.

---

## Phase 4: Configure Advanced Docker Networking & Routing

Because Coolify isolates separate multi-container applications inside native, random-hash bridge networks, your Cloudflare Tunnel container must be manually bridged into the target application network space to communicate.

### 1. Identify Target Network and Application Metrics

Run `sudo docker ps` on your server to extract the precise runtime instance names:

| Role | Name |
|---|---|
| **Glances Container** | `glances-k0k8gwsgw4wsowcssk0s8wo0` |
| **Isolated App Network** | `k0k8gwsgw4wsowcssk0s8wo0` |
| **Tunnel Container** | `goofy_chatelet` |

### 2. Connect the Tunnel to the Isolated Network

Bridge the network boundary by cross-connecting the Cloudflare client to the destination network block:

```bash
sudo docker network connect k0k8gwsgw4wsowcssk0s8wo0 goofy_chatelet
```

### 3. Update the Cloudflare Public Hostname Route

Now that the containers can communicate directly via the internal Docker DNS, configure the endpoint routing rules inside Cloudflare:

1. Go back to your tunnel settings under **Networks** > **Tunnels** > **Configure**.
2. Navigate to the **Public Hostname** tab and click **Edit** on your route.
3. Configure the specific private entrypoint:
   - **Type**: `HTTP`
   - **URL**: `glances-k0k8gwsgw4wsowcssk0s8wo0:61208`
4. Save the adjustments.

---

## Phase 5: Coolify Domain Optimization

To ensure that the internal Coolify reverse proxy (Traefik) does not loop or break SSL handshakes — since Cloudflare terminates public SSL natively at the edge — the entry domain configuration inside Coolify must be downgraded to an unencrypted internal HTTP string.

1. Open your **Coolify Panel** and navigate to your **Glances Application Settings**.
2. Under the **Domains** entry string, strip the HTTPS protocol, replacing it with:

   ```
   http://glances.nmcyber.com
   ```

3. Save the changes. Coolify will smoothly drop proxy tasks over to Cloudflare without throwing continuous redirect configurations.

---

## Operational Troubleshooting Checklist

| Symptom | Cause & Resolution |
|---|---|
| **Error Code 502 (Bad Gateway)** | The tunnel is up but cannot find the target container. Ensure you have run the `docker network connect` command and verified the spelling of the target container name in Cloudflare's Public Hostname settings. |
| **`The redirect_uri is not associated...`** | Double check your GitHub Developer Settings. Ensure your Zero Trust team subdomain precisely matches the string (`fancy-cell-829b`) saved inside the GitHub Authorization callback text field. |
| **Access Denied / Un-authorized Page** | GitHub login was successful, but the email address of that GitHub account is not listed inside your Cloudflare Access Application Include Rule. |

### Inspecting Active Containers

Run the following command on your server to check running infrastructure assets:

```bash
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```
