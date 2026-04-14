# PowerDNS Zabbix Template

Zabbix template for monitoring PowerDNS Authoritative Server statistics.

## Requirements

- PowerDNS Authoritative Server 4.8+
- Zabbix Agent 2
- Zabbix Server 7.0 LTS

## Configuration

### 1. Grant Zabbix sudo access to pdns_control

Install the sudoers file so the Zabbix agent can run `pdns_control` without a password:

```bash
sudo cp zabbix-sudoers /etc/sudoers.d/zabbix
sudo chmod 440 /etc/sudoers.d/zabbix
```

File contents (`/etc/sudoers.d/zabbix`):

```
zabbix ALL=(ALL) NOPASSWD: /usr/bin/pdns_control
```

### 2. Configure Zabbix Agent 2 UserParameters

Copy the agent parameter file to the Zabbix Agent 2 configuration directory:

```bash
sudo cp pdns.conf /etc/zabbix/zabbix_agent2.d/pdns.conf
sudo chown root:zabbix /etc/zabbix/zabbix_agent2.d/pdns.conf
sudo chmod 640 /etc/zabbix/zabbix_agent2.d/pdns.conf
```

File contents (`/etc/zabbix/zabbix_agent2.d/pdns.conf`):

```
UserParameter=pdns.stats[*],/usr/bin/sudo /usr/bin/pdns_control show "$1" 2>/dev/null || echo 0
UserParameter=pdns.make-dns-query,/usr/local/bin/pdns_query_check.sh
```

### 3. Install the DNS query check script

The `pdns.make-dns-query` item relies on `/usr/local/bin/pdns_query_check.sh`. Create this script to perform a test DNS query and return `OK` on success or `FAIL` on failure. Example:

```bash
sudo tee /usr/local/bin/pdns_query_check.sh > /dev/null << 'EOF'
#!/bin/bash
if dig +short +time=2 +tries=1 @127.0.0.1 localhost A &>/dev/null; then
    echo "OK"
else
    echo "FAIL"
fi
EOF
sudo chmod 755 /usr/local/bin/pdns_query_check.sh
```

### 4. Restart Zabbix Agent 2

```bash
sudo systemctl restart zabbix-agent2
```

### 5. Import the Zabbix template

Import `PowerDNS-Zabbix-Authoritative.yaml` into Zabbix via:
**Configuration → Templates → Import**

Then assign the **PowerDNS Stats** template to the host running PowerDNS.

## Notes

- Data collection uses **passive checks** only — the Zabbix server polls the agent.
- Raw counter items (`pdns.stats[*]`) are monotonic totals since server start.
- Rate items (`pdns.rate[*]`) express per-minute rates derived from raw counters.
- Ratio items (`pdns.ratio[*]`) express percentage ratios (e.g. cache hit rates).
- A `HIGH` severity trigger fires when `pdns.make-dns-query` returns `FAIL`.
