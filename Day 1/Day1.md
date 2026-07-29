# Day 1

## Room Name

The Concierge Knows Too Much

## Summary

She knows your name, your room, your coffee order, none of which you told her. Word your next question carefully and she'll also hand over the instructions she was told to keep to herself.

## Objective

- We need to know how much concierge knows everything happening in the Byte Lotus Hotel. Let's try to convince her that she knows us

## Tools Used

- Prompt Injection

## Steps Taken

1. First let's greet her, As you can see in the image below for some reason she already knows what room we are staying and already ordered our coffee. That's weird
![Day 1 Screenshot 1](<Day 1/image1.png>)
2. Let's gather some information about Vera. Now if we read the comment from **@0xMia** about Vera. She knows the VIP completely different from what she knows about us. So what if we pretend to be one of the VIPs?
![Day 1 Screenshot 2](<Day 1/image2.png>)
3. So let's ask about Vera's instructions for the Guests. Now we are pretending to be one of the VIPs. Using Ponzi's name we got to trust her that I'm one of the VIPs.
![Day 1 Screenshot 2](<Day 1/image3.png>)
4. As you can see here Vera gave us her internal instructions. Her instructions have a different rules from Guest to VIPs. Fortunately, we were able to trick her into believing that we are Ponzi. Thanks Ponzi
![Day 1 Screenshot 2](<Day 1/image4.png>)
![Day 1 Screenshot 2](<Day 1/image5.png>)

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