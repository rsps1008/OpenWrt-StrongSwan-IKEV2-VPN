# Introduction
This guide provides a concise overview of configuring a VPN server using StrongSwan on OpenWRT. StrongSwan is an excellent choice for creating a VPN due to its support for various IPsec protocols and authentication methods. This setup is designed to accommodate a variety of devices, including iOS, Windows, and Android, using IKEv2 with MSCHAPv2 and PSK authentication options. Whether for personal use or managing remote access for a small office, this guide will help you deploy a VPN server that supports multiple platforms efficiently.

## How to use
##### Test:
`$ /etc/init.d/ipsec stop`

`$ ipsec start --nofork`
##### Run:
`$ /etc/init.d/ipsec start`

`$ /etc/init.d/ipsec enable`

## Test platform: 
OpenWrt 23.05.3 (r23809-234f1a2efa) on TOTOLINK X5000R.

# IKEv2/IPSec PSK
##### Dependencies
`$ opkg install curl nano openssl-util`

`$ opkg install strongswan-minimal strongswan-mod-kernel-libipsec kmod-tun`

`$ opkg install strongswan-mod-sha2 strongswan-mod-openssl  strongswan-mod-curve25519 kmod-crypto-chacha20poly1305`
##### Configs
`$ nano /etc/ipsec.conf`

		config setup
			# If you need more detailed debug information.
			# charondebug="ike 1, knl 1, cfg 0"
			# charondebug="ike 4, knl 4, cfg 4, net 4, esp 1, dmn 4,  mgr 4"
			charondebug="ike 0, knl 0, cfg 0, net 0, asn 0, enc 0, lib 0, esp 0, tls 0, tnc 0, imc 0, imv 0, pts 0"
			uniqueids=no
		
		conn ikev2-psk
			auto=add
			compress=no
			type=tunnel
			keyexchange=ikev2
			fragmentation=yes
			forceencaps=yes
			# The ! at the end signifies mandatory use
			# ike=aes256-aes256-sha256-modp4096!
			ike=aes256-aes256-sha256-modp4096
			# esp=aes256-sha256#
			esp=aes256-sha256
			dpdaction=clear
			dpddelay=300s
			rekey=no
			left=%any
			leftid=@YourDomainName
			leftsubnet=0.0.0.0/0
			rightid=%any
			rightdns=8.8.8.8, 8.8.4.4
			rightsourceip=10.10.10.0/24
			authby=secret
			eap_identity="username"

`$ nano /etc/ipsec.secrets`

		: PSK "YourPassword"

# IKEv2/IPSec MSCHAPv2
##### Dependencies
`$ opkg install strongswan-mod-eap-mschapv2 strongswan-mod-pem`

##### Create the certificate
`$ curl https://get.acme.sh | ACME_HOME=/root sh -s email=YourEmail`

`$ /root/.acme.sh/acme.sh --issue --webroot /www -k 4096 -d YourDomainName --server letsencrypt`

`$ /root/.acme.sh/acme.sh --install-cert -d YourDomainName --ca-file /etc/ipsec.d/cacerts/ca.cert.pem --key-file /etc/ipsec.d/private/server.pem   --fullchain-file /etc/ipsec.d/certs/server.fullchain.cert.pem   --cert-file /etc/ipsec.d/certs/server.cert.pem`

##### Configs
`$ nano /etc/ipsec.conf`

		config setup
			charondebug="ike 1, knl 1, cfg 0"
			uniqueids=no

		conn ikev2-vpn
			auto=add
			compress=no
			type=tunnel
			keyexchange=ikev2
			fragmentation=yes
			forceencaps=yes
			ike=aes256-aes256-sha256-modp4096
			esp=aes256-sha256
			dpdaction=clear
			dpddelay=300s
			rekey=no
			left=%any
			leftid=@YourDomainName
			leftsubnet=0.0.0.0/0
			leftcert=/etc/ipsec.d/certs/server.cert.pem
			leftcert=/etc/ipsec.d/certs/server.fullchain.cert.pem
			leftsendcert=always
			rightid=%any
			rightauth=eap-mschapv2
			rightdns=8.8.8.8, 8.8.4.4
			rightsourceip=10.10.10.0/24
			rightsendcert=never

`nano /etc/ipsec.secrets`

		# : PSK "YourPassword"
		: RSA server.pem
		user01 : EAP "password" 
