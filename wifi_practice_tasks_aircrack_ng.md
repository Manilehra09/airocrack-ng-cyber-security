# WiFi Cracking Practice Tasks — Aircrack-ng Suite

**Created:** September 19, 2025

A progressive set of hands-on practice tasks for learning wireless security testing with the Aircrack-ng suite and companion tools. Each task includes: **Objective**, **Required tools/hardware**, **Step-by-step actions**, **Expected results**, **Hints / troubleshooting**, and **safety/legal checklist**.

---

## Lab setup (required before starting)
- **Hardware**: A Kali Linux machine (VM or physical) with a Wi‑Fi adapter that supports monitor mode & injection (e.g., Alfa AWUS036NHA or similar). If using a VM, attach the USB adapter to the VM.
- **Software**: Kali Linux, Aircrack-ng suite (`airmon-ng`, `airodump-ng`, `aireplay-ng`, `aircrack-ng`), `hcxdumptool`/`hcxtools` (optional), `macchanger`, `hostapd` (for creating test APs), and `dnsmasq` if you build an AP.
- **Safe test targets**: Use either a disposable router/AP you own, a virtual AP created with `hostapd`, or an intentionally vulnerable test network (never test public/third-party networks without permission).
- **Permissions**: Have written permission to test if the AP isn't yours.

---

## Task 1 — Verify adapter capabilities (Beginner)
**Objective:** Confirm your wireless adapter supports monitor mode and injection.

**Required:** Wi‑Fi adapter, Kali.

**Steps:**
1. `sudo rfkill unblock wifi`
2. `iw dev` and `sudo airmon-ng`
3. Start monitor: `sudo airmon-ng start wlan0` → note monitor interface name (e.g., `wlan0mon`).
4. Test injection with a short aireplay test: `sudo aireplay-ng --test wlan0mon`.

**Expected:** Adapter appears in `airmon-ng`, `iw dev` shows device; injection test reports success or supported.

**Hints:** If injection fails, check chipset compatibility and driver; try different USB ports.

---

## Task 2 — Passive scanning and discovery (Beginner)
**Objective:** Discover nearby APs and clients; collect basic info (BSSID, channel, encryption).

**Steps:**
1. `sudo airodump-ng wlan0mon`
2. Let it run for 2–5 minutes; note ESSIDs, BSSIDs, CH, ENC, CIPHER, AUTH.
3. Save output: `sudo airodump-ng --output-format csv -w discover wlan0mon`.

**Expected:** CSV file with list of APs and associated clients.

**Assessment:** Identify a target AP you control for later tasks.

---

## Task 3 — Focused capture of WPA handshake (Beginner → Intermediate)
**Objective:** Capture a WPA/WPA2 4‑way handshake for a target AP you own.

**Required:** Target AP (owned/test AP), at least one client associated.

**Steps:**
1. `sudo airodump-ng --bssid <BSSID> -c <channel> -w /tmp/handshake wlan0mon`
2. In another terminal, force reconnect with deauth: `sudo aireplay-ng --deauth 5 -a <BSSID> wlan0mon` (or target client with `-c <CLIENT_MAC>`).
3. Watch airodump terminal for `WPA handshake: <BSSID>` indicator.
4. Stop capture and verify with `aircrack-ng -J` or by inspecting cap file.

**Expected:** Capture file contains handshake; airodump shows the handshake indicator.

**Hints:** Lock to the channel to avoid missed packets; deauth specific client if general deauth doesn't trigger reconnection.

---

## Task 4 — Crack captured handshake with a wordlist (Intermediate)
**Objective:** Use a wordlist to attempt to crack the captured WPA handshake.

**Required:** Capture from Task 3, wordlist (e.g., `rockyou.txt`).

**Steps:**
1. `sudo aircrack-ng -w /usr/share/wordlists/rockyou.txt -b <BSSID> /tmp/handshake-01.cap`
2. Observe whether passphrase is found.

**Expected:** If passphrase is in the wordlist, aircrack-ng prints it; otherwise, it fails.

**Assessment:** If failure, discuss next steps (custom wordlists, rules, or Hashcat for GPU cracking).

---

## Task 5 — PMKID capture with hcxdumptool (Intermediate)
**Objective:** Capture PMKID from AP (useful when client handshakes are hard to obtain) and convert to Hashcat format.

**Steps:**
1. Put adapter into monitor mode (`wlan0mon`).
2. `sudo hcxdumptool -o dump.pcapng -i wlan0mon --enable_status=1`
3. Run until you capture valid PMKID entries.
4. Convert: `hcxpcapngtool -o hashcat.22000 dump.pcapng`

**Expected:** `hashcat.22000` file ready for Hashcat cracking with `-m 22000`.

**Hints:** Some APs may not reveal PMKID; ensure driver supports required features.

---

## Task 6 — WEP IV collection & crack (Legacy / Educational)
**Objective:** Practice WEP cracking in a controlled lab (understand why WEP is insecure).

**Required:** A WEP-encrypted network you control.

**Steps:**
1. `sudo airodump-ng --bssid <BSSID> -c <CH> -w wepcap wlan0mon`
2. Use `packetforge-ng` to craft ARP and `aireplay-ng --arpreplay` to inject, increasing IVs.
3. After collecting many IVs, run `aircrack-ng wepcap-01.ivs`.

**Expected:** With sufficient IVs, aircrack-ng recovers WEP key.

**Warning:** WEP is deprecated and insecure; this is for educational purposes only.

---

## Task 7 — MAC spoofing & staying anonymous (Beginner)
**Objective:** Change your MAC and verify it takes effect.

**Steps:**
1. `sudo ip link set wlan0 down`
2. `sudo macchanger -r wlan0`
3. `sudo ip link set wlan0 up`
4. Verify with `ip link show wlan0`.

**Assessment:** Confirm the new MAC appears and that you can still operate in monitor mode.

---

## Task 8 — Create a virtual AP using hostapd (Intermediate)
**Objective:** Build a controlled test AP so you can safely practice attacks without affecting other devices.

**Steps (high level):**
1. Create a `hostapd.conf` with SSID, channel, and passphrase.
2. Start hostapd: `sudo hostapd hostapd.conf`.
3. Optionally configure DHCP via `dnsmasq`.
4. Connect a test client (phone/VM) to the AP and practice captures.

**Expected:** Running AP that you control; use this for repeated practice.

---

## Task 9 — Advanced: Use Kismet & Bettercap for passive reconnaissance (Advanced)
**Objective:** Use Kismet to passively map networks and Bettercap to explore active attack surfaces in a lab.

**Steps:**
1. `sudo kismet` — run for 10–20 minutes, export a timeline & client maps.
2. `sudo bettercap -iface wlan0mon` — experiment with passive sniffing and (if allowed) local MITM techniques against your test AP.

**Safety:** Bettercap is intrusive; only run against owned test targets.

---

## Task 10 — Challenge: Full test & report (Capstone)
**Objective:** Combine skills: discover target AP, capture handshake/PMKID, attempt cracking with strategies, log steps, and produce a short report.

**Deliverables:**
- Raw captures (pcap/pcapng).
- Commands used (history or script).
- Cracking attempts & outcomes (wordlists tested, Hashcat parameters if used).
- A 1‑page report: objective, tools, methods, results, mitigation suggestions for the AP owner.

**Scoring (suggested):**
- Discovery & capture: 30 pts
- Cracking attempt & methodology: 30 pts
- Report clarity & mitigation advice: 40 pts

---

## Safety, ethics & checklist (must read before every lab session)
- Only test networks you own or have explicit permission for (written).
- Log everything (timestamps, commands, outputs).
- Avoid public networks and never capture traffic containing PII unless explicitly permitted and consented.
- When using disruptive attacks (deauth, MDK), run them only in isolated lab environments.

---

## Additional resources & wordlists
- `/usr/share/wordlists/rockyou.txt`
- SecLists (rockyou variants, rules)
- Aircrack-ng docs, hcxtools docs, Kismet docs

---

If you'd like, I can:
1. Also save this as a PDF (done),  
2. Add individual ready-to-run bash scripts for Tasks 1–4, or  
3. Create a printable scoring checklist for the Capstone task.
