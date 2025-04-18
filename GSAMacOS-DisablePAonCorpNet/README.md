# macOS Corporate DNS Detection Script

This repository provides a Bash script and LaunchDaemon configuration to detect when a macOS device is connected to your corporate network (direct or via VPN) by checking DNS server IPs. On detection, it toggles the `IsPrivateAccessDisabledByUser` key in the `com.microsoft.globalsecureaccess` plist.

---

## Prerequisites

- A macOS machine with administrator (sudo) privileges.  
- Access to Terminal to create files under `/usr/local/bin`, `/var/log`, and `/Library/LaunchDaemons`.

---

## 1. Install the Bash Script

1. **Create the script**  
   Open Terminal and run:
   ```bash
   sudo tee /usr/local/bin/check_corp_dns.sh > /dev/null << 'EOF'
   #!/usr/bin/env bash

   # Logging setup
   LOG_FILE="/var/log/check_corp_dns.log"
   mkdir -p "$(dirname "$LOG_FILE")"
   log() { echo "$(date +'%Y-%m-%d %H:%M:%S') - $*" >> "$LOG_FILE"; }
   log "Script started"

   # Define corporate DNS servers (customize these IPs):
   CORP_DNS=(
     "10.1.10.10"
     "10.1.10.11"
   )

   # Grab current DNS servers
   mapfile -t CURRENT_DNS < <(scutil --dns | awk '/server\[[0-9]+\] :/ {print $3}')
   log "Current DNS servers: ${CURRENT_DNS[*]}"

   # Detect corporate DNS
   FOUND=0
   for dns in "${CURRENT_DNS[@]}"; do
     if [[ " ${CORP_DNS[*]} " == *" $dns "* ]]; then
       FOUND=1
       break
     fi
   done

   log "Corporate network detected? $FOUND"
   log "Setting IsPrivateAccessDisabledByUser = $FOUND"
   defaults write com.microsoft.globalsecureaccess \
     IsPrivateAccessDisabledByUser -int $FOUND
   log "Script completed"
   EOF
   ```
2. **Make it executable**  
   ```bash
   sudo chmod +x /usr/local/bin/check_corp_dns.sh
   ```

---

## 2. Install the LaunchDaemon

1. **Create the plist**  
   ```bash
   sudo tee /Library/LaunchDaemons/com.microsoft.check_corp_dns.plist > /dev/null << 'EOF'
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
     <key>Label</key>
     <string>com.microsoft.check_corp_dns</string>

     <key>ProgramArguments</key>
     <array>
       <string>/usr/local/bin/check_corp_dns.sh</string>
     </array>

     <key>StartInterval</key>
     <integer>20</integer>

     <key>RunAtLoad</key>
     <true/>

     <key>StandardOutPath</key>
     <string>/var/log/check_corp_dns.log</string>

     <key>StandardErrorPath</key>
     <string>/var/log/check_corp_dns.log</string>
   </dict>
   </plist>
   EOF
   ```
2. **Set ownership & permissions**  
   ```bash
   sudo chown root:wheel /Library/LaunchDaemons/com.microsoft.check_corp_dns.plist
   sudo chmod 644   /Library/LaunchDaemons/com.microsoft.check_corp_dns.plist
   ```
3. **Load & enable on boot**  
   ```bash
   sudo launchctl load -w /Library/LaunchDaemons/com.microsoft.check_corp_dns.plist
   ```

---

## 3. Verification & Logs

- **Check daemon status**  
  ```bash
  sudo launchctl list | grep com.microsoft.check_corp_dns
  ```
- **Tail live logs**  
  ```bash
  sudo tail -f /var/log/check_corp_dns.log
  ```

You should see entries every 20 seconds.

---

## 4. Customization

To update your corporate DNS IPs, edit the `CORP_DNS` array in `/usr/local/bin/check_corp_dns.sh` and then reload the daemon:

```bash
sudo launchctl unload /Library/LaunchDaemons/com.microsoft.check_corp_dns.plist
sudo launchctl load -w /Library/LaunchDaemons/com.microsoft.check_corp_dns.plist
```

---

## 5. Reboot Persistence

Because the LaunchDaemon has `RunAtLoad` enabled, it will automatically start on system boot and execute every 20 seconds thereafter.

---

*Share this README with any macOS user to quickly deploy automated corporate DNS detection and plist toggling.*