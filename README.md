# Wireshark PCAP Investigation

## Part 1 — Getting Familiar with the PCAP
- Total packets: ~8224
- Protocols observed: LLMNR, NBNS, ARP, UDP, TCP, HTTP, DNS, TLSv1.2, DHCPv6...
- Frequent source IP: 192.168.0.7 (private)
- Public vs private: both observed
![Screenshot_2026-09-14_21-06-54](https://hackmd.io/_uploads/rJcGz18Kfl.png)


## Part 2 — Identify the Main Host
- Filter used: ip.addr == 192.168.0.7
- Other IPs communicating with host: 199.192.165.120, 209.197.110.119, 104.100.120.242
- Private or also public: also public
- External IPs: 209.x.x.x, 199.x.x.x, 104.x.x.x
![Screenshot_2026-09-11_16-28-11](https://hackmd.io/_uploads/ryeaXyIKMe.png)


## Part 3 — DNS Investigation
- Filter used: dns
- DNS server IP: 68.105.28.11
- www.-google.com resolved to IP: 173.194.219.105 and others.
- Other domains queried: 
-- Device 192.168.0.7 sent a DNS query asking for the ip address behind www.gstatic.com the DNS server at 68.105.28.11 responded with the ip addr 216.58.194.67
-- Device 192.168.0.7 sent a DNS query asking for the ip address behind www.udemy.com the DNS server at 68.105.28.11 responded with the ip addr 151.101.24.175
-- Device 192.168.0.7 sent a DNS query asking for the ip address behind connect.facebook.net the DNS server at 68.105.28.11 responded with the ip addr 31.13.77.12
- Why DNS matters: DNS matters cause without it the web exist but humans can't use it, unless they can cram all ip adress... its important cause it is what helps one locate the ip of a particular site one is sending a request to.
![Screenshot_2026-09-11_16-49-33](https://hackmd.io/_uploads/ByDY9yUFMl.png)


## Part 4 — TCP Investigation
- Filter used: tcp
- Source IP / Port: 192.168.0.7 /60070
- Destination IP / Port: 151/101.24.175 /80
- SYN packet:[SYN] Seq=0 Win=8192 Len=0 MSS=1460 WS=256 SACK_PERM
- SYN-ACK packet:[SYN, ACK] Seq=0 Ack=1 Win=30660 Len=0 MSS=1460 SACK_PERM WS = 1024
- ACK packet: [ACK] Seq=1 Ack=1 Win=65536 Len=0
- Purpose of handshake: the purpose is to establish a connection before communication.
![Screenshot_2026-09-14_21_56_57](https://hackmd.io/_uploads/SydWZe8Ffg.png)


## Part 5 — Ports
- Port 80: HTTP (unencrypted web)
- Port 443: HTTPS (encrypted web)
- Source ports observed: 60058, 60064, 60054
- Why they change:source port changes for each connection so that it can keep multiple connection seperated and match incoming replies correctly to the right one.


## Part 6 — HTTP Investigation
- GET request: GET /HTTP/1.1
- Source/Destination IP: from 192.168.0.7 to 72.246.125.34
- 200 OK example: HTTP /1.1 200 OK (text/html)
- 404 example: The host 192.168.0.7 made an http request to 72.246.125.34 the server responded with multiple messages two returned 404 not found (meaning some request resource does not exist on the server). while four returned 200 ok (those request were found and returned).
- Request vs response: request is when i connect to a server and ask for data, while response: is when the server acknowledge my request and provide me with the data i requested for.
![Screenshot_2026-09-14_22-39-04](https://hackmd.io/_uploads/BJnjPlLKGg.png)

## Part 7 — HTTPS / TLS
- SNI: www.clw.ford.com
- Port: 443
- Why analysts can still see info despite encryption: Because encryption hides the content that travels through the internet traffic, but that doesn't mean it hide all the information.
- HTTP vs HTTPS security difference: HTTPS is protected because TLS encrypts data in transit, authenticates the website, and helps detect -tampering. while HTTP sends data without tls to encrypt it.
![Screenshot_2026-09-14_22-43-07](https://hackmd.io/_uploads/BJklhgLFfl.png)

## Part 8 — Follow a TCP Stream
Hosts: 192.168.0.7 and www.udemy.com server
Protocol: HTTP
Observation: 301 redirect to HTTPS
Why useful to analyst: It's useful cause it gives some details like (hostname, user-agent, connection type)..etc

## Part 9 — Find Something Interesting
- Finding: udemy.com HTTP → HTTPS redirect
- How found: Follow TCP Stream
- Meaning: I followed a TCP stream and saw this conversation between the device 192.168.0.7 and Udemy's server.
- Further investigation needed: Reading through the HTTP traffic, I followed a TCP stream between 192.168.0.7 and Udemy's server. I found that the host made an HTTP GET request to www.udemy.com, and the server responded with HTTP 301 Moved Permanently, redirecting the connection to https://www.udemy.com/. This means that the server enforcing HTTPS. The unencrypted request never received actual data, it was redirected to the secure version.the behaviour of udemy is normal, no suspicious activity.
![Screenshot_2026-09-14_15-43-50](https://hackmd.io/_uploads/r11UbbUKGx.png)


## Part 10 — SOC Analyst Challenge
- Who: main internal host= 192.168.0.7 
-- External hosts communicating with internal host:(199.192.165.120, 209.197.110.119, 216.58.218.98)...etc
- What protocols: LLMNR, NBNS, ARP, UDP, TCP, HTTP, DNS....etc
- Where: destination ips: 68.105.28.11, 173.194.219.105, 216.58.195.46... etc
- How: Tcp.port = 80 source port= 60064
-- Tcp.port = 443 source port = 60054
- Why: Based on the above data the user is casually browsing through the internet, jumping from one website to another.


## FINAL CONCLUSION 
- The captured traffic shows a Windows host 192.168.0.7 browsing multiple websites. Before visiting each site, the device sent DNS queries to resolve domain names into IP addresses. Once connected, most communication was encrypted using TLS/HTTPS, protecting the data in transit. Sites that initially received plain HTTP connections (port 80) responded with 301 redirects, bouncing the user to the secure HTTPS version instead. ARP was used too, to find the router's MAC address, which is how the device was able to send packets out to the internet. final verdict, no malicious or suspicious activity was identified — this capture reflects normal, secure web browsing behavior.
