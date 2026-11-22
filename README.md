Hospital-SOC: Security Operations Center for a Smart Hospital

A small SOC lab built in VMware for a FAST-NUCES cybersecurity course project. A hospital web app sits behind a firewall, and three realistic attacks are run against it and detected with a SIEM and an IDS.

Team: Abdul Rafay (23I-2008), Hamiz Rathore (23I-2093), Irbaz (23I-2075)

<img width="1600" height="953" alt="Architecture diagram" src="https://github.com/user-attachments/assets/495b9e49-fb0e-4f0c-91a3-cb4e526c1fdd" />
Lab components
VM	Role
FortiGate-VM 7.4	Perimeter firewall (MGMT, HOSPITAL_LAN, WAN), sends deny logs to Wazuh over syslog
Wazuh 4.14	SIEM: collects agent logs, FortiGate syslog and Suricata alerts, with 15 custom rules
App server (Ubuntu)	Flask hospital app (port 5000, SQLite) with Suricata IDS (ET ruleset) and a Wazuh agent
Client (Ubuntu)	Normal user and insider-threat source
Kali Linux	External attacker
Custom detection rules

Rules 100001-100015 in local_rules.xml (group Hospital) cover failed logins, repeated failures, account lockout, rate limiting, unauthorized patient and admin access, and FortiGate deny / port-scan detection.

Attack scenarios

Each scenario has a written expected result, the actual result, a normal-activity comparison, Wireshark evidence and Wazuh alerts.

#	Scenario	Source -> Target	Detected by	Wazuh rules
1	Brute-force login	LAN client -> app :5000	App audit log, rate limiter	100003, 100004, 100013
2	Unauthorized record access by a nurse (/patients/1, /admin/, both 403)	LAN client -> app	App audit log	100002, 100005
3a	Outside port scan of the firewall WAN IP	Kali -> FortiGate port3	FortiGate implicit deny, syslog	100015
3b	Web attack: sqlmap user agent, SQL injection URI, nmap scripting engine	LAN client -> app	Suricata (ET SCAN / ET WEB_SERVER)	86601
Scenario 1: Brute-force login

Five failed logins for one account in about 10 seconds. The app locked the account for 15 minutes and the sixth attempt got HTTP 429. Wazuh raised 100003 for each failure, 100004 for the burst and 100013 for the lockout. The firewall saw nothing, because client and app share one LAN segment.

Scenario 2: Unauthorized record access

A logged-in nurse requested a patient who was not assigned to her and the admin page. Both returned 403 and fired rules 100002 and 100005. A normal run (an assigned patient, HTTP 200) raised no alerts above level 3.

Scenario 3a: Outside reconnaissance

An nmap SYN scan from Kali against the firewall WAN IP on ports 22, 80, 443 and 5000 was denied by the default policy. FortiGate logged action=deny, forwarded it by syslog, and Wazuh raised rule 100015. Suricata saw nothing, since the traffic never reached the LAN.

Scenario 3b: Web attack from inside

Three probes were run against the hospital app: a sqlmap user agent, a UNION SELECT string in the URL, and nmap -sV -sC. Suricata logged ET SCAN and ET WEB_SERVER signatures and Wazuh raised each as rule 86601. Suricata is detection-only, so the requests were not blocked.

Evidence

Screenshots (app logs, raw logs, Wireshark captures, Wazuh dashboard) are in s1-evidence/, s2-evidence/, s3a-evidence/, s3b-evidence/ and wazuh dashboard/. The full write-up is in reports/ (43-page report with protocol stack analysis, mitigations and limitations).

Mitigations discussed in the report

HTTPS and secure cookies, stricter rate limiting, anchored detection regexes, IPS mode for Suricata, tuning alert levels, and a segmented network with LAN-to-LAN traffic routed through the firewall.

Limitations
Lab-scale: one run per scenario, with separate runs for the evidence sets.
The app serves HTTP only, which is by design for traffic analysis.
sqlmap was simulated through its user agent with curl.
Suricata alerts all arrive at level 3 in Wazuh and need tuning.

Secrets, keys and packet captures are intentionally not included in this repository.
