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

## Fault 2

- **Symptom (what we observed):**

  ```text
  <Paste actual command and failure output>
  ```

- **Hypothesis (what we suspected and why):**

  <Explain the technical hypothesis supported by the symptoms>

- **Proof (the command that confirmed the cause):**

  ```text
  <Paste actual command and confirming output>
  ```

- **Fix (what we changed):**

  <Describe the actual corrective action>

- **Confirmation (output showing it now works):**

  ```text
  <Paste actual verification output>
  ```

- **Found/proved by:** <Actual investigator(s)>

---
