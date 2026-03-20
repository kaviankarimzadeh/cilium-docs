## Fragment Handling

To check whether fragmentation occurred, check the value of the following metrics:

`cilium_bpf_map_pressure{map_name="cilium_ipv4_frag_datagrams"}`
`cilium_bpf_map_pressure{map_name="cilium_ipv6_frag_datagrams"}`

If they’re non-zero, it means that fragmented packets were processed.

---


As well, it’s possible check how many MTU error messages (i.e. ICMP fragmentation_needed or ICMPv6 packet_too_big) have been processed by Cilium’s datapath by checking the following metrics:

`mtu_error_message_total`