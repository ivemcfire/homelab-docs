# Jumphost dual-IP cleanup (issue #21)

Run on jumphost (192.168.100.62) when ready — needs sudo:

```bash
ssh user@192.168.100.62
sudo sed -i 's/address 192.168.100.152/address 192.168.100.62/' /etc/network/interfaces
sudo grep address /etc/network/interfaces  # verify
sudo ip addr del 192.168.100.152/24 dev enp1s0  # immediate effect
sudo ip -br a | grep enp1s0  # confirm only .62 + IPv6 remain
```

Or DHCP path: comment out static block, set `iface enp1s0 inet dhcp` (BE6500 reservation pins `.62`).
