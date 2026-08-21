---
Assignment requirements

- custom rule: alert icmp any any -> 192.168.2.112 any (msg:"ICMP Flood Detected"; classtype:attempted-dos; sid:1000002; rev:1; detection_filter:track by_src,count 20,seconds 1;)

- Brief analysis: Icmp is a type of dos technique, an attacker uses it to send large amount of traffic to a particular server. it attemps to distrupt a server by sending massive traffic to the target, causing the server cpu to be overwhelmed, which can result to slow performance.

![WhatsApp Image 2026-08-18 at 7.21.28 AM (2)](https://hackmd.io/_uploads/B1oxKK-PGx.jpg)

![WhatsApp Image 2026-08-18 at 7.21.28 AM (1)](https://hackmd.io/_uploads/H1-GYFWvzl.jpg)

![WhatsApp Image 2026-08-18 at 7.21.28 AM](https://hackmd.io/_uploads/Bk6MtYWPzl.jpg)

![Screenshot_2026-08-20_19_31_34](https://hackmd.io/_uploads/SJZLfGBDMg.png)

![Screenshot_2026-08-20_19_31_44](https://hackmd.io/_uploads/BkedGGHPGl.png)

![Screenshot_2026-08-20_19_32_16](https://hackmd.io/_uploads/rJ0FGMSPMl.png)


so now there are six pictures the first one i used the rule given in the pdf, now the second one, i used my custom rule added classtype and detection filter.

- classtype- is a lablel category that tells us what type of issues or suspicious activity that's going on in the server.
-- it mainly helps with clearer alert and priority catigorization and default level of importance, like attemted dos normally being priority(2).. stubled upon this during my first attack when it shows no priority.

- Next addition to my rule is {detection filter}- this is a rule i add personally, normally the system would detect the icmp attack but not after it as been overwhelmed, so i add for it to flag out the attack early by adding a rule that 20 request a second should be flaged immediately.

so the diffrence between the before and now is: before no priorty or classification to tell us whats going on how serious it is or what type of attack is being executed towards the server. so with the new customization we now have the details of attack type and priorty level.