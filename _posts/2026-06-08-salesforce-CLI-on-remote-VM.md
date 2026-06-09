---
layout: single
title: "Authenticating Salesforce CLI with SSO on Ubuntu"
date: 2026-06-08
categories: [linux]
tags: [cli, ubuntu, sso, authentication]
excerpt: "Bridging the gap between a headless Linux VM and a local Windows desktop browser to securely authenticate the Salesforce CLI through corporate Single Sign-On. Or, I should have read the manual."
toc: true
---

<figure>
    <a href="/assets/sf-cli/headless-auth-header.jpg"><img src="/assets/sf-cli/headless-auth-header.jpg"></a>
    <figcaption>SF CLI header</figcaption>
</figure>


> Dumb challenge that I decided to take on: I've got Ubuntu 22.04 VM running my `sf` CLI. The CLI authentication launches a local browser (or, requires a Salesforce admin to allow JWT auth). But my web browser lives on my local Windows desktop. I authenticate to Salesforce via our corporate workspace SSO. How can I authenticate my CLI?

It sounds like a simple remote connection issue, but the modern Salesforce CLI architecture has some strict guardrails around interactive logins. I put this write-up together to track the clever hacks that Claude and Gemini came up with that failed, and the method I could have used from the beginning.

---

## The Environment & Constraints

Our target state required meeting three distinct conditions:
*   **CLI on Ubuntu:** The Ubuntu VM has no graphical user interface (GUI) and no native web browser installed.
*   **SSO Login:** Standard username/password logins are blocked by corporate policy; authentication *must* flow through the identity provider, and that uses a browser that **must** be running on a corporate-managed device.
*   **Local Isolation:** The local Windows host cannot have the Salesforce CLI installed, preventing us from simply authenticating locally and exporting the session.

---

## What I Tried (and Why it Failed)

When you run a standard `sf org login web` command on a headless machine, the CLI hangs. It searches the Linux environment for a default browser utility to launch the SSO page, finds nothing, and gets stuck in an infinite retry loop without printing a fallback URL. 

I tried a few creative ways to cheat this validation mechanism (Claude and Gemini thought of them):

### The Fake "Firefox" Executable
I thought I could trick the CLI into thinking it had a real browser. I created a local directory, added it to the front of the system `PATH`, and created a dummy bash script named `firefox`. The script was incredibly simple — it just accepted the URL passed by the CLI and echoed it out to the terminal:

```bash
# 1. Create a local bin folder if you haven't already
mkdir -p ~/bin

# 2. Create a fake "firefox" executable that just echoes the login URL
echo -e '#!/bin/bash\necho -e "\n=== COPY THIS URL ===\n$1\n=====================\n"' > ~/bin/firefox
chmod +x ~/bin/firefox

# 3. Add our local bin folder to the FRONT of your session's execution path
export PATH=~/bin:$PATH

# 4. Force the CLI to launch what it thinks is your firefox browser
sf org login web --instance-url $INSTANCE_URL --alias f5-org --set-default --browser firefox
```

I executed the command with the `--browser firefox` flag. In theory, it should have printed the login link. Instead, the CLI threw a hard **CannotOpenBrowserError**. Under the hood, the modern Node.js core of the CLI doesn't just look for a string in your PATH; it actively probes the operating system registry to verify a functional graphical desktop application is registered. No GUI means an immediate crash.

### The Container Mode Flag
Next, I tried forcing the CLI into headless mode using environment variables: `SF_CONTAINER_MODE=true`.

This bypassed the browser validation checks, but hit a hard security gate built into the compiler, resulting in this flat error:

```bash
Error (SfError): "org login web" isn't supported when authorizing in a headless environment.
```

Salesforce CLI explicitly blocks the web login flow in container modes because opening a local listening server on port 1717 to catch browser redirects is considered an insecure pattern on headless cloud environments. 

### Port Forwarding
I really thought this would work. Here's what you do:
1. Set up a port forwarding from your Windows VM (port 1717) to your Linux VM (port 1717). In PowerShell:

```powershell
ssh -L 1717:localhost:1717 ubuntu@192.168.1.153
```
2. run `sf org login web` from the Ubuntu VM, and see if you can get the URL that it would call with a browser. In Bash:
{:start="2"}

```bash
sf org login web --instance-url $INSTANCE_URL --alias f5-org --set-default --browser firefox
```

3. A URL should appear, find a way to open it on the Windows VM, and when it redirects to the callback URL of `localhost:1717/xxx`, the Windows browser will call localhost, get forwarded to the Linux VM, and the process listening on the Linux VM would respond.
{:start="3"}

Unfortunately, this fails. In step #2, the URL that *would* be called by the CLI is not printed to the user. Trying to catch it failed (see my first trick). And in step #3, the process was not listening or expecting for any traffic on tcp/1717 because launching a browser never worked.

## The Breakthrough: Sfdx-URL Auth Flow

What is this? At a fundamental level, the difference comes down to **how the authentication secret is generated and captured**. 

`sf org login --help` shows us we have options other than a browser-based login flow:

```bash
ubuntu@ubuntu-Virtual-Machine:~$ sf org login --help
Authorize an org for use with Salesforce CLI.

USAGE
  $ sf org login COMMAND

COMMANDS
  org login access-token  Authorize an org using an existing Salesforce access token.
  org login jwt           Log in to a Salesforce org using a JSON web token (JWT).
  org login sfdx-url      Authorize an org using a Salesforce DX authorization URL stored in a file or through standard input (stdin).
  org login web           Log in to a Salesforce org using the web server flow.
```

I didn't try the first method (access-token) because even if I found my access token via a web login, access tokens expire quickly. This would have been shortlived. The second method (jwt) requires a Salesforce admin to set up an external app. But this third option (sfdx-url) was the key.

**web** is an interactive protocol designed for a human sitting at a desktop; **sfdx-url** is a static file ingestion engine designed for automation, scripts, and headless servers.

### 1. Web Login (`sf org login web`)

When you execute this command, the CLI assumes it has full access to a local desktop environment. 

1. **The Listener:** The CLI spins up a temporary, lightweight HTTP web server on your local machine, listening out on `http://localhost:1717/OauthRedirect`.
2. **The Handshake:** It communicates with your OS to launch your default browser, pointing it to your Salesforce login or SSO landing page.
3. **The Interception:** After you successfully pass SSO and click "Allow", Salesforce redirects the browser to `localhost:1717`. Your local CLI server catches that incoming HTTP request, extracts the authorization code, exchanges it under the hood for an active refresh token, and kills the temporary web server.

> 🛑 **Why it breaks headlessly:** If there is no desktop GUI registry to open a browser, the command blocks indefinitely waiting for a browser application process to return a success status. If you try to force it into container mode, the compiler explicitly blocks it because exposing port 1717 on an automated runner is an unnecessary security risk.

---

### 2. SFDX-URL Login (`sf org login sfdx-url`)

This flow drops the entire interactive browser dance and the local listening web server. It treats authentication exactly like importing an SSH key or an API token.

Instead of *generating* a session, it *restores* one. It reads a specialized URL string from a file using the custom `force://` protocol handler scheme:

```text
force://<clientId>:<clientSecret>:<refreshToken>@<instanceUrl>
```

When you pass this file to the CLI:

1. The CLI bypasses all browser or OS validation steps entirely.
2. It reads the string, grabs the refreshToken, and shoots a direct, headless backend API call out to the <instanceUrl> to verify the token is valid.
3. Once Salesforce verifies it, the CLI saves the connection to its local secure storage (~/.sfdx/ or ~/.local/share/sf/).

This is the standard enterprise method for CI/CD pipelines. You generate a token once, inject that raw force://... string into your pipeline's protected environment variables (like GitHub Encrypted Secrets), echo it to a temporary file during a build run, and authenticate the runner instantly with zero human eyes or clicks required.

### Summary of web vs sfdx-url login

| Feature | `web login` | `sfdx-url` |
| :--- | :--- | :--- |
| **Primary Use Case** | Local day-to-day development on a laptop/desktop. | CI/CD pipelines (GitHub Actions, Jenkins) and headless VMs. |
| **Mechanism** | Spawns a local web server socket and triggers an OS browser. | Reads a pre-constructed token string straight from a text file. |
| **Human Needed?** | **Yes.** Requires real-time UI interaction to input credentials. | **No.** Fully automated and scriptable. |
| **Network Dependency** | Requires a local port (`1717`) to receive a callback loop. | Needs only standard outbound HTTPS to validate the token against Salesforce. |
| **Token Type** | Generates a brand-new Refresh Token dynamic session. | Imports an *existing* Refresh Token generated elsewhere. |

## Conclusion: How to use sfdx-url auth flow
Manually request the token via the local Windows browser with a specially constructed URL. The redirect to a callback URL will fail, but grab the refresh token from the URL. Create an auth file for CLI auth. 

| Step | Action | Description |
| :--- | :--- | :--- |
| **1** | Construct OAuth URL | For me, this was the URL: https://<instance_url>/services/oauth2/authorize?response_type=token&client_id=PlatformCLI&redirect_uri=http://localhost:1717/OauthRedirect|
| **2** | Browser SSO | Pasting that URL into the Windows desktop browser allowed the user to complete their corporate workspace SSO and hit Allow. |
| **3** | Capture Redirect | The browser redirected to an expected "Site Cannot Be Reached" error page on localhost:1717. We copied the URL straight out of the address bar, which now contained the secret **refresh_token**. |
| **4** | Format Token File | On the Ubuntu VM, create a custom text file (sfdx_auth.txt) matching the exact URI schema the CLI expects: `force://PlatformCLI::<refreshToken>@<instance_url>` |
| **5** | Import to CLI | Execute `sf org login sfdx-url --sfdx-url-file sfdx_auth.txt` to authenticate with this file, then remove the file. |

---

### Disclosure and The Post-Logout Surprise

One interesting takeaway happened at the very end of testing. I ran `sf org logout` to test cleaning up the environment, and then ran the `sfdx-url` login command again using the exact same text file. It worked. I expected that the refresh token would be revoked at client logout and I'd be forced to login via browser, but I was wrong.

So I think that `sf org logout` does not revoke your refresh token on the server. **Remember: guard this refresh token like a password and don't persist it on disk**.

So, disclosure: I'm not sure if this is a blessed approach by corporate IT, so check with your policies. But it's a documented method for [authenticating to Salesforce for continuous integration (CI) environments](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_url.htm). 
