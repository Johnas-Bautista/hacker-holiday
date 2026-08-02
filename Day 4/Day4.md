# Day 4 — Packed Light

## Summary

Tiny packets. Odd hours. Suspiciously regular. Someone's smuggling out the data equivalent of a hotel towel every night, folded neatly inside traffic that looks ordinary until you decode it.

## Objective

Analyze the provided packet capture to uncover a covert process running in the background — one that isn't part of any legitimate hotel service — and extract the flag hidden within its traffic.

## Tools / Techniques / Threat Vectors

- Network Traffic Analysis (Wireshark)
- Command and Control (C2) Traffic Identification
- Data Exfiltration via HTTP Cookies
- XOR Cipher / Base64 Decoding (CyberChef)

---

## Steps Taken

1. In this walkthrough, we will perform digital forensics on a packet capture (PCAP) using Wireshark. First, download the provided file from the TryHackMe room. Open Wireshark and load the .pcap file to begin the investigation. You can complete this using either the TryHackMe AttackBox or your own local machine.

2. Upon opening the capture, you might feel overwhelmed by the sheer volume of data—there are over 1,300 packets in total. Sifting through this manually to find useful information would be difficult, so we need to filter out the background noise. Let's refer back to the room for some hints and clues.
![Day 4 Screenshot 1](<image1.png>)

As you can see, we have a solid clue on where to start. The machine appears to be repeatedly reaching out to port 8080. Since port 8080 is commonly used for alternative HTTP web services, these aren't standard ICMP pings, but likely HTTP requests. Let's apply the following filter in **Wireshark** to narrow our scope: `tcp.port == 8080`
![Day 4 Screenshot 2](<image2.png>)

3. After applying the filter, the number of visible packets drops significantly to 465. Now that we've cleared the noise, we can actually begin our investigation. Looking closely, there is a stream of packets continuously establishing TCP handshakes (SYN, SYN-ACK, ACK) followed by HTTP requests to a web server at the IP address `34.41.103.191`.

![Day 4 Screenshot 3](<image3.png>)

The very first TCP stream contains an HTTP GET request downloading a Python script. Let's examine this in detail. **Right-click the GET request for /tmp/updates.py, select "Follow", and click "HTTP Stream"**. This will display the entire transaction, showing the client's request in red and the server's response in blue.
![Day 4 Screenshot 4](<image4.png>)

4. What exactly are we looking at? If we analyze the Python script, it reveals itself as a keylogger communicating with a Command and Control (C2) server. It tracks keystrokes and exfiltrates the data to the web server by embedding it within a cookie session parameter: `hotel_sess_state={value}`.
![Day 4 Screenshot 5](<image5.png>)

According to the script, each captured keystroke is encrypted using an XOR cipher, encoded in Base64, and then sent via an HTTP GET request. Essentially, a new HTTP request is generated for every single character typed.

To better understand the captured data, let's investigate the subsequent HTTP GET requests. By inspecting the Packet Details Pane under Hypertext Transfer Protocol, we can see that the cookie value changes with every request. Let's filter the traffic to show only the GET requests so we can count them and isolate the exfiltrated data. Use the following Wireshark filter: `http.request.method == "GET"`
![Day 4 Screenshot 6](<image6.png>)
![Day 4 Screenshot 7](<image7.png>)

5. Excluding the initial GET request for /tmp/updates.py, there are exactly 30 GET requests made to the root directory (/). As expected from our script analysis, each request contains a different Base64-encoded cookie value. Our next step is to extract all these session values sequentially and use CyberChef to reverse the encryption.
![Day 4 Screenshot 8](<image8.png>)

```text
hotel_sess_state={get the value of this parameter} First GET /
hotel_sess_state={get the value of this parameter} 2nd GET /
# ...and so on
```

6. Once you have extracted and concatenated all the cookie values, you will be left with a string resembling this:
`HA==AA==BQ==Mw==Hg==ew==Og==fA==Fw==eQ==Ow==Fw==Pw==fA==PA==Kw==IA==eQ==Jg==Lw==Fw==eA==Pg==LQ==Gg==Fw==MQ==eA==PQ==NQ==` 

Now, let's head over to CyberChef to decode this.

1. Paste the string into the Input pane.
2. In the Operations pane, add From Base64 (leave the settings as default).
3. Next, add the XOR operation.

But what is the **XOR** key?
If we look closely at the xor function in the Python script, we can see the full key string is H0t3lSt@ff0NlyK3epS3cr3t!. However, the script encrypts and sends the data one character at a time. Because the length of the data being encrypted is always 1, the index used in the cipher loop (i) is always 0. Therefore, the script only ever uses the first character of the combined key, which is the letter H.

Set the XOR key in CyberChef to H (with standard UTF-8 encoding). Voila! The decrypted keystrokes are revealed in the Output pane, giving us our flag.
![Day 4 Screenshot 9](<image9.png>)

---

## What I Learned

In this exercise, you learned how to perform digital forensics on a packet capture by using Wireshark filters to isolate suspicious HTTP traffic communicating with a Command and Control (C2) server. By following TCP streams, you successfully identified and analyzed a Python keylogger that exfiltrated keystrokes character-by-character via Base64-encoded and XOR-encrypted cookie values. Furthermore, you learned how to logically deduce the encryption key from the script's source code to decrypt the payload using CyberChef, while also refining your technical writing skills by mastering proper Markdown syntax to create a clear, professionally formatted security write-up.
