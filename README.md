# TryHackMe Boogeyman 1 Challenge

## Introduction
I am tasked to analyse the Tactics, Techniques, and Procedures (TTPs) executed by a threat group, from obtaining initial access until achieving its objective. 

Julianne, a finance employee working for Quick Logistics LLC, received a follow-up email regarding an unpaid invoice from their business partner, B Packaging Inc. Unbeknownst to her, the attached document was malicious and compromised her workstation. 

The security team was able to flag the suspicious execution of the attachment, in addition to the phishing reports received from the other finance department employees, making it seem to be a targeted attack on the finance team. Upon checking the latest trends, the initial TTP used for the malicious attachment is attributed to the new threat group named Boogeyman, known for targeting the logistics sector.

I am tasked to analyse and assess the impact of the compromise.

### Artefacts
For the investigation, I have been provided with the following artefacts:
- Copy of the phishing email (dump.eml)
- Powershell Logs from Julianne's workstation (powershell.json)
- Packet capture from the same workstation (capture.pcapng)

## Walkhrough

### Email Analysis 

Based on the initial information, the compromise originated from a phishing email. To investigate further, I began by analysing the ‘dump.eml’ file found in the ‘artefacts’ directory. For an easier examination, I uploaded the ‘dump.eml’ to PhishTool, where I was able to identify key details: the sender's email address, the victim's email address, and the third-party mail relay service used by the attacker. These findings were based on the analysis of the DKIM-Signature and List-Unsubscribe headers. 	  
<p align="center">
<img src="https://i.imgur.com/SLtEZCQ.png" height="80%" width="80%" alt="Phishtool1"/>
<img src="https://i.imgur.com/CIHnaIV.png" height="80%" width="80%" alt="Phishtool2"/>

Next, I opened the ‘dump.eml’ in Thunderbird to save the attached file. The attachment was password-protected, but I was able to extract its contents using the password provided in the email body. Once the encrypted archive was unlocked, I used ‘lnkparse’ to further analyse the extracted payload. The analysis revealed an encoded payload embedded in the Command Line Arguments field. 
<p align="center">
<img src="https://i.imgur.com/pJksRvQ.png" height="80%" width="80%" alt="Thunderbird"/>
<img src="https://i.imgur.com/5RC6RPw.png" height="80%" width="80%" alt="lnkparse"/>

### Endpoint Security 
Based on the initial findings, we discovered how the malicious attachment compromised Julianne's workstation:
- A PowerShell command was executed.
- Decoding the payload reveals the starting point of endpoint activities. 

With the following discoveries, I should now proceed with analysing the PowerShell logs to uncover the potential impact of the attack.

Using JQ to print all the ‘ScriptBlockText’ values, I was able to identify several key details related to the attack. These included the domains used by the attacker for file hosting and command-and-control (C2) operations, the name of the enumeration tool downloaded by the attacker, and the specific file accessed using the downloaded ‘sq3.exe’ binary. Additionally, I discovered an extracted KeePass database file (‘protected_data.kdbx’), the use of hex encoding during the exfiltration attempt of the sensitive file, and the ‘nslookup’ tool employed for data exfiltration. 
<p align="center">
<img src="https://i.imgur.com/K859zSx.png" height="80%" width="80%" alt="json"/>

### Network Traffic Analysis 
Based on the PowerShell logs investigation, I have seen the full impact of the attack:
- The threat actor was able to read and exfiltrate two potentially sensitive files.
- The domains and ports used for the network activity were discovered, including the tool used by the threat actor for exfiltration.

Finally, I can complete the investigation by understanding the network traffic caused by the attack.

Using Wireshark, I filtered the HTTP packets related to the file-hosting domain (‘files.bpackaging.xyz’) and followed the TCP stream. This analysis revealed that the attacker had used Python to host their suspected file or payload server. 
<p align="center">
<img src="https://i.imgur.com/OmbtsZJ.png" height="80%" width="80%" alt="python"/>

Filtering HTTP packets associated with the C2 domain ‘cdn.bpackaging.xyz’ revealed that the attacker used the ‘POST’ method to transmit the output of executed commands. 
<p align="center">
<img src="https://i.imgur.com/lR37yyN.png" height="80%" width="80%" alt="Post"/>

Additionally, the earlier analysis of PowerShell logs confirmed that the attacker utilized the DNS protocol for data exfiltration, evidenced by the use of the ‘nslookup’ tool. 

The PowerShell logs indicated that ‘sq3.exe’ was used to access the file ‘plum.sqlite’. The logs also revealed the command ‘SELECT * FROM NOTE’, suggesting that this may be where the password for the file is stored. To locate the output of this command, I filtered the network packets for HTTP traffic containing ‘sq3.exe’ and followed the TCP stream. 
<p align="center">
<img src="https://i.imgur.com/cR08r5R.png" height="80%" width="80%" alt="plum"/>

In the TCP stream, I found the same SQL command reflected in the PowerShell logs. By examining the subsequent stream, I discovered the output of this command. 
<p align="center">
<img src="https://i.imgur.com/VyhmimU.png" height="80%" width="80%" alt="decimal"/>

After converting the extracted data from decimal, I was able to retrieve the password for the file.
<p align="center">
<img src="https://i.imgur.com/YB6NIGK.png" height="80%" width="80%" alt="password"/>

I determined that the file was exfiltrated using DNS. To investigate further, I needed to find all DNS requests sent to ‘bpackaging.xyz’ where the destination IP address was ‘167.71.211.113’. To accomplish this, I used TShark to filter only the hex strings of the relevant packets and output them into a ‘.txt’ file. Afterward, I converted the hex data to ASCII and saved the output as a ‘.kdbx’ (KeePass database) file. 
<p align="center">
<img src="https://i.imgur.com/FoDcPfP.png" height="80%" width="80%" alt="tshark"/>

Using the password we previously acquired, I was able to unlock the file and uncover what the attacker was after.
<p align="center">
<img src="https://i.imgur.com/NyqRwZx.png" height="80%" width="80%" alt="card"/>

## Summary
The TryHackMe "Boogeyman 1" challenge is a valuable experience for learning cybersecurity due to its well-designed scenario that combines realistic threat detection and analysis tasks. It effectively guides you through the investigation of a phishing attack, requiring the use of various tools like Wireshark, TShark, and PowerShell logging, which are essential in real-world incident response. The challenge not only reinforces fundamental skills, such as packet analysis and payload extraction, but also introduces more advanced concepts like DNS-based exfiltration and command-line forensics. Overall, "Boogeyman 1" offers a comprehensive, hands-on approach that enhances both technical expertise and problem-solving abilities, making it an excellent learning opportunity.
