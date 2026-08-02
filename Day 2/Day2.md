# Day 2 — Room 404

## Summary

He booked the quiet room. It's not on the floor plan, not in the brochure, not on any door. But port 8080 is wide open, and the rooms it never lists are the ones worth finding.

## Objective

Investigate the web server at `http://<VICTIM_MACHINE_IP>:8080` to uncover hidden, unlisted paths and determine what's being concealed within them.

## Tools / Techniques / Threat Vectors

- Directory Enumeration (Gobuster)
- Exposed `.git` Repository Disclosure
- Git History Recovery (`wget`, `git restore`)

---

## Steps Taken

1. First, let's start with some basic recon using the browser's built-in inspector. The site appears to be purely static, but there's a catch: the navbars and buttons are full of dead `href` links that go nowhere. A static site with broken navigation is definitely suspicious. Time to bring out the big guns.
![Day 2 Screenshot 1](<image1.png>)

2. Nothing beats good old-fashioned directory enumeration. Let's fire up **Gobuster** to see what's hiding out of sight.

```text
gobuster dir -u http://example.com:port_number -w /usr/share/wordlists/dirb/common.txt
```

Run this command to start knocking on those digital doorknobs.
![Day 2 Screenshot 2](<image2.png>) 

3. Well, well, what do we have here? Gobuster found an exposed /.git directory! It looks like the developer deployed the entire repository to the live web server without restricting access. A classic—and fatal—amateur mistake.
![Day 2 Screenshot 3](<image3.png>)

4. After a bit of trial and error on how to properly exploit this, we can use wget to snatch the goods. We'll run it recursively to download all the files from the /.git/ folder straight to our attacker machine. This will automatically rebuild the directory structure wherever you run the command, as shown below:

```text
wget -r http://10.49.134.245:8080/.git/
```

![Day 2 Screenshot 4](<image4.png>)
![Day 2 Screenshot 5](<image5.png>)

5. Now for the grand finale. Since we successfully downloaded a valid Git repository, we can leverage native Git commands to reveal the hidden contents. Just cd into the downloaded directory and run git restore .. This restores the working tree files, pulling the hidden source code (and our flag!) right out of the commit history.
![Day 2 Screenshot 6](<image6.png>)

## What I Learned

Today's biggest takeaway is a harsh lesson in deployment security: **never leave your `/.git` directory exposed on a production server.** It might seem like a harmless hidden folder, but as we saw today, it's essentially handing over the blueprints to the entire operation. By combining directory enumeration (Gobuster) with simple recursive downloading (wget), we proved that "security by obscurity"—like keeping a room off the hotel floor plan—doesn't work against someone willing to knock on every door. Finally, using `git restore .` was a great reminder that version control remembers *everything*, and a single sloppy deployment is all it takes to expose an organization's deepest secrets.
