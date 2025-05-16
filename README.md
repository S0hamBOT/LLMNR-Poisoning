# LLMNR Poisoning

I set out on a mission to understand and stop a network attack called LLMNR Poisoning, which can trick computers into giving up their passwords. I wanted to test it out, capture some data, and then secure the network so it wouldn’t happen again. Here’s how it all went down, along with what I learned about defending against both LLMNR and NBT-NS attacks.

## Step 1: Setting Up My Workspace
I began my journey in my Kali Linux system, working from a directory I called `/home/kali/Desktop/active_directory/hacks/LLMNR_Poisoning`. This was my hub for the entire operation, where I stored all my findings.

## Step 2: Capturing Data with Responder
I used a tool called Responder to listen for network traffic, running this command:
```bash
sudo responder -I eth0 -dPv
```
This set Responder to monitor the network on the `eth0` interface at the IP `192.168.31.18`, ready to catch any credentials that came its way.

![Responder Listening](screenshots/step1.png)
![Responder Listening](screenshots/step2.png)

## Step 3: Triggering a Response from a Windows Machine
I switched to a Windows machine and tried accessing a fake resource by typing `\\192.168.31.18` into the file explorer. Since it didn’t exist, the machine sent an LLMNR query, which Responder intercepted, prompting the machine to authenticate.
![Windows Authentication Prompt](screenshots/step3.png)
![Windows Authentication Prompt](screenshots/step4.png)

After attempting to access the fake resource, a dialog box popped up on the Windows machine asking for my login credentials. I entered my username and password, and upon submitting, the machine sent an NTLMv2 hash to the Kali system, which Responder captured.
![Windows Authentication Prompt](screenshots/step5.png)
![Windows Authentication Prompt](screenshots/step6.png)

Responder captured an NTLMv2 hash from the machine, which I could see in its logs—a long string of encrypted data holding the user’s password.

![Captured NTLMv2 Hash](screenshots/step7.png)

## Step 4: Saving the Hash for Analysis
I copied the NTLMv2 hash from Responder and saved it into a file named `hash.txt`. This was the key I needed to unlock the password hidden inside.

![Hash Saved to File](screenshots/step8.png)
![Hash Saved to File](screenshots/step9.png)

## Step 5: Cracking the Hash with Hashcat
Next, I turned to Hashcat to crack the hash using a wordlist called `rockyou.txt`. I ran this command:
```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```
The `-m 5600` flag told Hashcat I was dealing with an NTLMv2 hash. It didn’t take long for Hashcat to crack it, revealing the password as `Password1`.

![Hashcat Command](screenshots/step11.png)

![Hash Cracked](screenshots/step12.png)

## Step 6: Securing the Network by Disabling LLMNR
After seeing how easy it was to capture and crack the hash, I knew I had to stop this attack from happening again. I decided to disable LLMNR using Group Policy.

I opened the Group Policy Management Console on a Windows server and created a new Group Policy Object (GPO) named `LLMNR-Poisoning-Removal`. I linked it to the domain `sohamjadhav.in` to apply it network-wide.

![New GPO Creation](screenshots/step13.png)
![New GPO Creation](screenshots/step14.png)

I edited the GPO, navigating to Computer Configuration > Policies > Administrative Templates > Network > DNS Client, and set the “Turn off multicast name resolution” policy to `Enabled` to disable LLMNR.

![Disable LLMNR Policy](screenshots/step15.png)

![Policy Settings](screenshots/step16.png)

I ensured the GPO was enforced to take effect properly.

![GPO Enforced](screenshots/step17.png)
![GPO Enforced](screenshots/step18.png)

## Step 7: Verifying the Fix on a Client Machine
To confirm the fix, I went to a Windows client and used PowerShell to check if LLMNR was disabled. I ran:
```powershell
$(Get-ItemProperty -Path "HKLM:\Software\Policies\Microsoft\Windows NT\DNSClient" -name EnableMulticast).EnableMulticast
```
The result was `0`, confirming LLMNR was turned off.

![PowerShell Verification](screenshots/step19.png)

## Step 8: Learning the Best Defenses Against LLMNR and NBT-NS
With LLMNR disabled, I wanted to dig deeper into the best ways to protect against both LLMNR and NBT-NS attacks. I found some key strategies that I didn’t capture in screenshots but are critical to know.

The best defense is to disable both LLMNR and NBT-NS where possible:
- **For LLMNR**, I already disabled it via Group Policy by setting “Turn off Multicast Name Resolution” under Computer Configuration > Administrative Templates > Network > DNS Client, as you saw earlier.
- **For NBT-NS**, I learned you can disable it by going to Network Connections > Network Adapter Properties > TCP/IPv4 Properties > Advanced tab > WINS tab, and selecting “Disable NetBIOS over TCP/IP”. Alternatively, you can write a logon script to automate this. I didn’t capture this step in a screenshot, but you can find more details on how attackers exploit NBT-NS here: [MITRE ATT&CK: T1557.001](https://attack.mitre.org/techniques/T1557/001/)

If disabling LLMNR or NBT-NS isn’t an option for a company, I discovered some fallback measures:
- **Require Network Access Control** to ensure only trusted devices can connect to the network.
- **Enforce Strong Passwords** (e.g., over 14 characters, avoiding common words). The longer and more complex the password, the harder it is for an attacker to crack the hash.

**Note:** If you ever want to try something like this, just make sure it’s in a safe, authorized environment—cybersecurity is all about protecting systems responsibly.
