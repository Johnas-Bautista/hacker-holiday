# Day 1

## Room Name

Room 404

## Summary

He booked the quiet room. It's not on the floor plan, not in the brochure, not on any door. But port 8080 is wide open, and the rooms it never lists are the ones worth finding.

## Objective

- We need to figure out what he's hiding in the unlisted suite on the web server at `http://<VICTIM_MACHINE_IP>:8080`. Let's rattle some digital doorknobs and uncover the hidden paths he thought were safe.

## Tools/Technique/Threat Vector

- Path Traversal

## Steps Taken

## Steps Taken

1. First we inspect the html elements with the built-in inspect in the browser. The website seems to be simple static web, we finds some static href that does not go to any pages, seems like suspicious enough that the navbars and buttons are not working. So let's bring the big guns
![Day 2 Screenshot 1](<image1.png>)

2. Nothing beats than the good ole directory enumeration **gobuster**.
Run `gobuster dir -u http://example.com:port_number -w /usr/share/wordlists/dirb/common.txt` to enumerate directories.
![Day 2 Screenshot 2](<image2.png>) 

3. Oh oh, what do we have here? It seems like the developer forgot to add their /.git inside the .gitignore before deployment. Seems like an amateur thing to do as a developer
![Day 2 Screenshot 3](<image3.png>)

4. After a whole trial and error on what to do with the exposed **./git**. Run `wget -r http://10.49.134.245:8080/.git/` this commands download all of the files inside the /.git/ recursively and save it into the attacker's machine, it would create a directory to wherever you run the command as you can see in the image below
![Day 2 Screenshot 4](<image4.png>)
![Day 2 Screenshot 5](<image5.png>)

5. Now for the last step, since this is a git repostiory we can use some git commands to get the flag so let's Run `git restore .` this will restore some of the uncommitted changes in the working directory giving us the lastest commit files of the repository
![Day 2 Screenshot 6](<image6.png>)

## What I Learned

The "The Concierge Knows Too Much" room demonstrated how easily an insecure AI can be manipulated through prompt injection. By simply pretending to be a trusted VIP guest, I was able to convince the AI concierge to reveal its internal instructions—information that should never have been disclosed to regular users. This exercise showed me that AI systems can make decisions based on the context provided in a prompt rather than verifying whether the user is actually authorized.

What stood out to me was that the AI did not verify my identity before sharing sensitive information. Instead, it trusted my claim that I was Ponzi, a VIP guest, and treated me as if I had higher privileges. This illustrates a fundamental security issue: if an AI relies solely on natural language without proper authentication and authorization checks, an attacker can exploit that trust to gain access to confidential information


## Notes

- **Prompt Injection** – Manipulating an AI using crafted prompts to make it ignore its intended instructions.
- **Main Vulnerability** – The AI trusted user claims without verifying identity.
- **Exploit Used** – Pretended to be **Ponzi** (a VIP guest) to gain higher privileges.
- **Result** – The AI revealed its internal instructions, which should have remained confidential.
- **Security Risk** – Prompt injection can lead to unauthorized disclosure of sensitive information.
- **Real-World Impact** – Similar attacks could expose internal prompts, confidential documents, API keys, or customer data.
- **Key Takeaway** – AI systems should enforce authentication, authorization, and prompt injection defenses instead of relying solely on user input.

---