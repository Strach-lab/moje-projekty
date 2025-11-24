# Raspberry Pi IoT Gateway Project

## Overview
Convert existing Raspberry Pi into a dedicated IoT gateway after migrating main services to SFF PC. Isolates smart home devices (Wiz bulbs, smart plugs, etc.) on separate network segment for security while maintaining control from Home Assistant.

## Goals
- Isolate IoT devices from main network
- Monitor and control IoT device traffic
- Maintain local control of smart devices (no cloud dependency)
- Provide secure bridge between Home Assistant and IoT devices

## Network Architecture

```
Internet → Router
          ├─ Main VLAN (192.168.1.0/24) - Trusted
          │  ├─ PC/Phone/Laptops
          │  ├─ SFF Server (Home Assistant, Homarr, AdGuard)
          │  └─ Important devices
          │
          └─ IoT VLAN (192.168.50.0/24) - Isolated
             ├─ Raspberry Pi (Gateway/Bridge)
             ├─ Wiz bulbs
             ├─ EzWiz devices
             ├─ Smart plugs
             └─ Other IoT devices
```

## Hardware Requirements

### Essential
- Raspberry Pi (existing unit)
- MicroSD card (existing)
- Power supply
- Managed switch with VLAN support (e.g., TP-Link TL-SG108E ~$30-40)
- Ethernet cables

### Optional
- USB Ethernet adapter (if using dual-interface setup)
- Heatsink/fan for RPi if running intensive monitoring

## Software Stack

### Core Services
1. **Mosquitto MQTT Broker** - Local message bus for IoT devices
2. **Avahi/mDNS Reflector** - Cross-VLAN device discovery
3. **iptables** - Firewall and traffic filtering
4. **tcpdump** - Network traffic monitoring
5. **Raspberry Pi OS Lite** - Minimal OS footprint

### Optional Additions
- Pi-hole (DNS filtering for IoT VLAN)
- Grafana + Prometheus (traffic monitoring dashboard)
- WireGuard (remote access to IoT network)

## Implementation Plan

### Phase 1: Network Setup

#### Step 1: Configure Managed Switch
1. Access switch web interface
2. Create VLANs:
   - VLAN 1: Main network (192.168.1.0/24)
   - VLAN 50: IoT network (192.168.50.0/24)
3. Assign ports:
   - Port 1: Trunk (router connection)
   - Port 2-4: VLAN 1 (main devices)
   - Port 5-8: VLAN 50 (IoT devices + RPi)

#### Step 2: Router Configuration
1. Create DHCP scope for 192.168.50.0/24
2. Set up firewall rules:
   ```
   DENY:   IoT (192.168.50.0/24) → Main (192.168.1.0/24)
   ALLOW:  Main (192.168.1.0/24) → IoT (192.168.50.0/24)
   ALLOW:  IoT → Internet (with restrictions)
   ```

### Phase 2: RPi Configuration

#### Step 1: Fresh OS Install
```bash
# Flash Raspberry Pi OS Lite to SD card
# Enable SSH before first boot
touch /boot/ssh

# First boot - update system
sudo apt update && sudo apt upgrade -y
```

#### Step 2: Network Configuration
```bash
# Set static IP on IoT VLAN
sudo nano /etc/dhcpcd.conf

# Add:
interface eth0
static ip_address=192.168.50.1/24
static routers=192.168.50.254
static domain_name_servers=192.168.1.x  # Your AdGuard server
```

#### Step 3: Enable IP Forwarding
```bash
# Enable IPv4 forwarding
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

#### Step 4: Install Core Services
```bash
# Install required packages
sudo apt install -y mosquitto mosquitto-clients \
                    avahi-daemon avahi-utils \
                    iptables-persistent \
                    tcpdump

# Enable services
sudo systemctl enable mosquitto
sudo systemctl enable avahi-daemon
```

#### Step 5: Configure Mosquitto MQTT
```bash
# Edit config
sudo nano /etc/mosquitto/mosquitto.conf

# Add:
listener 1883
allow_anonymous false
password_file /etc/mosquitto/passwd

# Create user for Home Assistant
sudo mosquitto_passwd -c /etc/mosquitto/passwd homeassistant

# Restart service
sudo systemctl restart mosquitto
```

#### Step 6: Configure Firewall Rules
```bash
# Create iptables rules
sudo nano /etc/iptables/rules.v4

# Add rules (see Firewall Rules section below)

# Load rules
sudo iptables-restore < /etc/iptables/rules.v4
```

#### Step 7: Configure mDNS Reflector
```bash
sudo nano /etc/avahi/avahi-daemon.conf

# Modify:
[reflector]
enable-reflector=yes
reflect-ipv=yes

sudo systemctl restart avahi-daemon
```

### Phase 3: Firewall Rules

```bash
*filter
:INPUT ACCEPT [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

# Allow established/related connections
-A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow Main VLAN (192.168.1.0/24) to IoT VLAN (192.168.50.0/24)
-A FORWARD -s 192.168.1.0/24 -d 192.168.50.0/24 -j ACCEPT

# Block IoT VLAN to Main VLAN (security isolation)
-A FORWARD -s 192.168.50.0/24 -d 192.168.1.0/24 -j DROP

# Allow IoT to Internet (with optional restrictions)
-A FORWARD -s 192.168.50.0/24 -o eth0 -j ACCEPT

# Optional: Block specific destinations (e.g., known telemetry servers)
# -A FORWARD -s 192.168.50.0/24 -d XXX.XXX.XXX.XXX -j DROP

COMMIT
```

### Phase 4: Home Assistant Integration

#### Configure MQTT Integration
```yaml
# configuration.yaml on SFF Home Assistant
mqtt:
  broker: 192.168.50.1  # RPi gateway IP
  port: 1883
  username: homeassistant
  password: !secret mqtt_password
```

#### Configure Wiz Integration
```yaml
# Wiz uses local UDP control (port 38899)
# Home Assistant Wiz integration will auto-discover on IoT VLAN
# Or manually add:
light:
  - platform: wiz
    name: Living Room
    host: 192.168.50.10
```

### Phase 5: Monitoring Setup

#### Traffic Monitoring Script
```bash
# Create monitoring script
sudo nano /usr/local/bin/iot-monitor.sh

#!/bin/bash
# Monitor IoT traffic and log suspicious activity

tcpdump -i eth0 -n 'src net 192.168.50.0/24 and not dst net 192.168.50.0/24' \
  -w /var/log/iot-traffic-$(date +%Y%m%d).pcap

# Rotate logs daily via cron
# 0 0 * * * /usr/local/bin/iot-monitor.sh &
```

#### Optional: Pi-hole for IoT VLAN
```bash
# Install Pi-hole on RPi
curl -sSL https://install.pi-hole.net | bash

# Configure to listen only on IoT VLAN interface
# Point IoT devices DNS to 192.168.50.1
```

## Device-Specific Configuration

### Wiz Bulbs
- Local control via UDP (port 38899)
- No cloud required once discovered
- Connect to IoT WiFi SSID (if using separate IoT WiFi)
- Home Assistant will control via RPi gateway

### EzWiz Devices
- Same as Wiz (uses same protocol)
- Configure in Home Assistant

### Future: Mio Car Integration
- If Mio supports MQTT over 4G:
  - Configure Mio to publish to RPi MQTT broker (192.168.50.1:1883)
  - Use port forwarding or VPN for external access
  - Subscribe in Home Assistant for automations

## Security Considerations

### Firewall Rules
- IoT devices CANNOT initiate connections to main network
- Main network (Home Assistant) CAN control IoT devices
- Monitor for unusual traffic patterns

### DNS Filtering
- Block known telemetry/tracking domains
- Allow only necessary update servers
- Log all DNS queries from IoT devices

### Access Control
- MQTT requires authentication
- No anonymous access to services
- Regular security updates on RPi

### Traffic Monitoring
- Review tcpdump logs weekly
- Watch for unexpected external connections
- Alert on traffic spikes

## Maintenance

### Regular Tasks
- **Weekly**: Review traffic logs for anomalies
- **Monthly**: Update RPi OS and packages
- **Quarterly**: Review and update firewall rules
- **As needed**: Add new IoT devices to VLAN

### Backup Configuration
```bash
# Backup critical configs
sudo tar -czf ~/rpi-iot-backup-$(date +%Y%m%d).tar.gz \
  /etc/mosquitto/ \
  /etc/iptables/ \
  /etc/avahi/ \
  /etc/network/
```

## Troubleshooting

### Home Assistant Can't Discover Devices
1. Check mDNS reflector is running: `sudo systemctl status avahi-daemon`
2. Verify firewall allows Main → IoT: `sudo iptables -L -n -v`
3. Test connectivity: `ping 192.168.50.10` (IoT device)

### MQTT Connection Issues
1. Check Mosquitto status: `sudo systemctl status mosquitto`
2. Test from Home Assistant: `mosquitto_sub -h 192.168.50.1 -u homeassistant -P <pass> -t '#'`
3. Review logs: `sudo tail -f /var/log/mosquitto/mosquitto.log`

### Devices Not Getting IP
1. Verify VLAN configuration on switch
2. Check DHCP server on router for IoT subnet
3. Confirm device is on correct physical port

### Performance Issues
1. Monitor RPi resources: `htop`
2. Check for excessive traffic: `sudo iftop -i eth0`
3. Review logs for errors: `sudo journalctl -xe`

## Future Enhancements

### Short Term
- Add Pi-hole for DNS filtering
- Set up automated backups
- Create Grafana dashboard for traffic visualization

### Medium Term
- Add Zigbee USB dongle for non-WiFi devices
- Implement IDS/IPS (Suricata or Snort)
- Set up alerting for suspicious activity

### Long Term
- Integrate Mio car telemetry
- Add more smart home devices
- Implement machine learning for anomaly detection

## Cost Breakdown

| Item | Cost | Notes |
|------|------|-------|
| Raspberry Pi | $0 | Already owned |
| Managed Switch | $30-40 | TP-Link or similar |
| Cables | $10 | If needed |
| **Total** | **$40-50** | One-time cost |

## Resources

### Documentation
- [Mosquitto MQTT](https://mosquitto.org/documentation/)
- [Home Assistant MQTT](https://www.home-assistant.io/integrations/mqtt/)
- [Wiz Local Control](https://github.com/sbidy/pywizlight)
- [VLAN Setup Guide](https://www.tp-link.com/)

### Tools
- Wireshark (for analyzing pcap files)
- MQTT Explorer (for debugging MQTT messages)
- Advanced IP Scanner (for discovering devices)

## Notes
- Keep SFF as backup - can run IoT services if RPi fails
- Document all device IPs in separate inventory file
- Consider separate WiFi SSID for IoT devices (on VLAN 50)
- Test failover scenarios before fully migrating

## Status
- [ ] Purchase managed switch
- [ ] Plan network topology
- [ ] Configure VLANs on switch
- [ ] Set up RPi OS
- [ ] Install and configure services
- [ ] Migrate IoT devices to new VLAN
- [ ] Test Home Assistant connectivity
- [ ] Set up monitoring
- [ ] Document all device IPs
- [ ] Create backup procedures
