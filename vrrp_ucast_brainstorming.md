# VRRP Unicast RFC Draft Brain Storming
VRRP protocol has withstood the test of time with stable deployments across enterprise, datacenter and service provider networks.
It's FSM is very simple and can be incorporated in many network designs.
The network deployments are evolving quickly and in many networks L2 and L3 multicast are not available. For example, by default public cloud environments like AWS, Azure, GCP etc. do not support L2 and L3 multicast.
But VRRP RFC restricts its operation on Multicast addresses.
For the future networks, a RFC is required.


### Keepalived Documentation
https://www.keepalived.org/manpage.html
- Search `unicast_peer` configuration.
- Min TTL and Max TTL configurations are also there.


### Huawei Documentaion
https://support.huawei.com/hedex/hdx.do?docid=EDOC1100512202&id=EN-US_CONCEPT_0172351673 --> TTLs will change, packets go across the networks.
https://support.huawei.com/hedex/hdx.do?docid=EDOC1100413563&id=EN-US_TASK_0172361754 --> configuration


### Cisco IOS-XR Documentation
https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/79x/b-ip-addresses-cg-ncs5500-79x/implementing-vrrp.html#unicast-vrrp-xr-27747-feat-2249


### Configuration
Need to configure the list of unicast peers IPs to which VRRP advertisements need to be sent.


### Modification to RFC9568 1.7 Definitions
- Virtual Router MAC Address --> Mentioned as multicast Ethernet MAC. Is the sentence confusing or conlating 'VRRP VMAC' with 'VRRP Multicast MAC for sending out the advertisements'?

### Modification to RFC9568 3. VRRP Overview
- It mentions multicast explicitly.

### Modification to RFC9568 5. Protocol
- Mentions multicast address.

### Modification to RFC9568 5.1.1. IPv4 Field Descriptions
- 5.1.1.1 --> The source address will remains the same.
- 5.1.1.2 --> Instead of multicast destination address, use the unicast address which is configured.
- 5.1.1.3 --> TTL == 255 check should remain.
- 5.1.1.4 --> No change in Protocol number

### Modification to RFC9568 5.1.2. IPv6 Field Descriptions
- 5.1.2.1 --> The source link local address remains the same.
- 5.1.2.2 --> Instead of multicast link local destination address, use the unicast link local address which is configured.
- 5.1.2.3 --> Hop limit check 255
- 5.1.2.4 --> No change in Next Header Number

### Modification to RFC9568 7.2. Transmitting VRRP Packets
- "Send the VRRP packet to the VRRP IPvX multicast group" --> modify this

