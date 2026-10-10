# Incident Log

Group: Penguins  
Members: Preston Shah Aman, Ye Myat Aung, Pyae Phyo Paing, Sai Soom Rath, Hashir Khan  
Company / domain: NovaPeak / novapeak.lab

Fill in ONE entry per confirmed fault. Each entry must include actual command output, evidence-based reasoning, the corrective action, and verification.

---

## Fault 1 — Incorrect Forward DNS Zone File Reference

- **Symptom (what we observed):**

  Initial BIND configuration validation reported:

  ```text
  root@ubuntu:~$ named-checkconf -z
  zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
  zone novapeak.lab/IN: not loaded due to errors.
  _default/novapeak.lab/IN: file not found
  zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101
  ```

  The forward zone failed to load, although the reverse zone loaded successfully. This suggested a problem specific to the forward-zone configuration rather than a failure of every DNS zone.

- **Hypothesis (what we suspected and why):**

  BIND was attempting to load a nonexistent file named `db.MISSING`. We suspected that the forward-zone declaration referenced an incorrect filename while the actual DNS zone database existed under a different name.

  The error identified a missing file, so checking the configured path and available files was more appropriate than immediately recreating the zone or reinstalling BIND.

- **Proof (the command that confirmed the cause):**

  Directory inspection confirmed the actual forward-zone file existed:

  ```text
  root@ubuntu:~$ ls -l /etc/bind/
  -rw-r--r-- 1 bind bind 212 Oct 10 04:46 db.novapeak.lab
  ```

  The zone declaration referenced a different filename:

  ```text
  root@ubuntu:~$ cat /etc/bind/named.conf.local
  zone "novapeak.lab" {
          type master;
          file "/etc/bind/db.MISSING";
  };

  zone "19.168.192.in-addr.arpa" {
          type master;
          file "/etc/bind/db.novapeak.lab.rev";
  };
  ```

  The configured forward-zone path did not match the existing file. This confirmed the incorrect file reference as the cause of the zone-loading failure.

- **Fix (what we changed):**

  Preserved the original `named.conf.local` as a backup, then corrected only the forward-zone file reference:

  ```bash
  sed -i 's|/etc/bind/db.MISSING|/etc/bind/db.novapeak.lab|' /etc/bind/named.conf.local
  ```

  The reverse-zone declaration and DNS records were left unchanged. We validated the configuration before restarting BIND.

- **Confirmation (output showing it now works):**

  BIND successfully loaded both company zones:

  ```text
  root@ubuntu:~$ named-checkconf -z
  zone novapeak.lab/IN: loaded serial 2024010101
  zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101
  ```

  The DNS service restarted successfully and reported `active`.

  The subsequent official checker confirmed:

  ```text
  [   2/2 ] C4   named-checkconf -z clean; named-checkzone clean
  TOTAL: 12.0 / 25
  ```

  The checker increased from 11.0/25 to 12.0/25, confirming that the zone-loading fault was resolved.

  However, client DNS connectivity still failed. This established that correcting the BIND configuration did not resolve every DNS-related symptom, and further investigation was necessary.

- **Found/proved by:** Pyae Phyo Paing (6705142024); evidence reviewed and documented by Pyae Phyo Paing (6705142024).

---

## Fault 2 — Firewall Blocking DNS Queries on UDP Port 53

- **Symptom (what we observed):**

  After correcting the forward-zone file reference, BIND successfully loaded its zones, but the client still could not reach the internal DNS server.

  A DNS query from the client namespace failed:

  ```text
  root@ubuntu:~$ ip netns exec client dig @192.168.19.10 www.novapeak.lab A +time=2 +tries=1
  ;; communications error to 192.168.19.10#53: timed out
  ;; no servers could be reached
  ```

  This indicated that a valid DNS configuration alone was not sufficient to make the service accessible to clients.

- **Hypothesis (what we suspected and why):**

  The BIND configuration permitted queries with `listen-on { any; };` and `allow-query { any; };`.

  Because the DNS service was configured to accept queries but the client received no response, we suspected that network filtering was preventing DNS packets from reaching the service.

  The firewall was investigated before making further changes to BIND.

- **Proof (the command that confirmed the cause):**

  The firewall rules showed an explicit DROP rule for UDP destination port 53:

  ```text
  root@ubuntu:~$ iptables -L INPUT -n -v --line-numbers

  Chain INPUT (policy ACCEPT 1648 packets, 135K bytes)
  num   pkts bytes target  prot opt in out source       destination
  1        6   517 DROP    17   --  *  *   0.0.0.0/0    0.0.0.0/0    udp dpt:53
  ```

  After another failed client DNS query, the rule's packet counter increased:

  ```text
  root@ubuntu:~$ iptables -L INPUT -n -v --line-numbers

  Chain INPUT (policy ACCEPT 1847 packets, 150K bytes)
  num   pkts bytes target  prot opt in out source       destination
  1        7   602 DROP    17   --  *  *   0.0.0.0/0    0.0.0.0/0    udp dpt:53
  ```

  The increase from 6 to 7 dropped packets, together with the client timeout, provided direct evidence that the firewall rule was blocking DNS traffic.

- **Fix (what we changed):**

  First, saved the existing firewall rules and verified the backup:

  ```bash
  iptables-save -c > /root/iptables.before-dns-fix.rules
  ```

  ```text
  Firewall backup verified
  ```

  Then removed the specific rule blocking inbound UDP port 53:

  ```bash
  iptables -D INPUT -p udp --dport 53 -j DROP
  ```

  The corrective action targeted the blocking firewall rule rather than changing unrelated DNS or network settings.

- **Confirmation (output showing it now works):**

  The subsequent firewall listing no longer contained the UDP port 53 DROP rule.

  Repeating the same client-side DNS query produced a successful authoritative response:

  ```text
  root@ubuntu:~$ ip netns exec client dig @192.168.19.10 www.novapeak.lab A +time=2 +tries=1

  ;; Got answer:
  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20762
  ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

  ;; ANSWER SECTION:
  www.novapeak.lab.  3600  IN  A  192.168.19.20

  ;; Query time: 0 msec
  ;; SERVER: 192.168.19.10#53(192.168.19.10) (UDP)
  ```

  The official checker improved from 12.0/25 to 18.25/25.

  This confirmed that the firewall was responsible for the DNS connectivity failure. The remaining lost points came from separate DNS record and DHCP integration issues, which required independent corrections.

- **Found/proved by:** Ye Myat Aung (6705142009); recorded evidence reviewed and documented by Sai Soom Rath (6705142034).

---
## Fault 3 — Incorrect API DNS A Record

- **Symptom (what we observed):**

  After restoring DNS connectivity, the client could query the authoritative DNS server, but the API hostname returned an incorrect IPv4 address.

  Inspection of the forward-zone database showed:

  ```text
  root@ubuntu:~$ nl -ba /etc/bind/db.novapeak.lab
  4  ns1 IN A 192.168.19.10
  5  www IN A 192.168.19.20
  6  mail IN A 192.168.19.30
  7  api IN A 192.168.19.1
  ```

  A client-side DNS query confirmed the incorrect response:

  ```text
  root@ubuntu:~$ ip netns exec client dig @192.168.19.10 api.novapeak.lab A +noall +answer +comments

  ;; ANSWER SECTION:
  api.novapeak.lab.  3600  IN  A  192.168.19.1
  ```

- **Hypothesis (what we suspected and why):**

  DNS connectivity was functioning, but the returned address was incorrect. We suspected an inaccurate A record in the authoritative forward-zone database rather than another network or firewall problem.

  The issue appeared isolated to the API hostname because the other company A records were correct.

- **Proof (the command that confirmed the cause):**

  The zone file directly contained the incorrect record:

  ```text
  root@ubuntu:~$ nl -ba /etc/bind/db.novapeak.lab
  7  api IN A 192.168.19.1
  ```

  The client received the same address from the authoritative DNS server. This confirmed that the wrong answer originated from the stored DNS record, not from client-side resolution failure.

- **Fix (what we changed):**

  Backed up the original forward-zone database and verified the backup.

  Corrected the API A record:

  ```bash
  sed -i 's/^api IN A 192\.168\.19\.1$/api IN A 192.168.19.40/' /etc/bind/db.novapeak.lab
  ```

  Updated the forward-zone SOA serial from `2024010101` to `2026101001` to reflect the zone modification.

  No changes were made to the other A records or the reverse-zone database.

- **Confirmation (output showing it now works):**

  The corrected record and updated SOA serial were verified:

  ```text
  2:@ IN SOA ns1.novapeak.lab. admin.novapeak.lab. ( 2026101001 3600 1800 604800 3600 )
  7:api IN A 192.168.19.40
  ```

  Zone validation succeeded:

  ```text
  root@ubuntu:~$ named-checkzone novapeak.lab /etc/bind/db.novapeak.lab
  zone novapeak.lab/IN: loaded serial 2026101001
  OK
  ```

  BIND restarted successfully and reported `active`.

  The client then received the correct API address:

  ```text
  root@ubuntu:~$ ip netns exec client dig @192.168.19.10 api.novapeak.lab A +noall +answer +comments

  ;; ANSWER SECTION:
  api.novapeak.lab.  3600  IN  A  192.168.19.40
  ```

  The official checker confirmed:

  ```text
  [ 3.0/3 ] C2   all four A records correct
  TOTAL: 19.0 / 25
  ```

  The checker increased from 18.25/25 to 19.0/25. The remaining failures concerned DHCP advertising the wrong DNS resolver, not the API record.

- **Found/proved by:** Ye Myat Aung (6705142009); recorded evidence reviewed and documented by Preston Shah Aman (6705142025).

---
