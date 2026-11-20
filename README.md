# OPNsense Firewall Troubleshooting Lab

A write-up of how I diagnosed why I could not reach the OPNsense web interface, what I found, and how I fixed it.

## Environment

- Dell OptiPlex 5060 SFF running Proxmox VE 9.2  
- OPNsense 26.7 running as a VM with two virtual NICs (WAN on `vmbr0`, LAN on a second bridge)  
- Client: Linux laptop on the home network

## Problem

The OPNsense web UI would not load from my laptop. The OPNsense VM was reachable by ping, but a browser and `curl` to the web UI timed out. Turning the firewall off from the console made the page load, so the firewall itself was the cause.

## Diagnosis

Working from the console and the web UI, I used three tools to see what the firewall was doing with my traffic:

1. **Live firewall log** (Firewall \> Log Files \> Live View): showed my connection attempts being blocked.  
2. **tcpdump** on the OPNsense interface: used to watch the packets between my laptop and OPNsense and check whether traffic was reaching the firewall.  
3. **pfctl \-sr**: printed the active ruleset, which showed rules I had not written myself.

That showed three separate things stacked on top of each other, and fixing one alone changed nothing.

## Root causes

I found three separate layers, and fixing one alone did not change the symptom.

**1\. Default deny on the WAN.** OPNsense blocks all new inbound connections on the WAN by default, so my laptop's connection to the web UI was dropped until I added a rule allowing it. The fix was a WAN allow rule for the web UI port from my laptop.

**2\. Block private networks.** This setting drops any inbound packet with a private source address on the WAN. My laptop's home address (192.168.x.x) is private, so it was blocked before my allow rule was ever checked. I turned it off for the lab only, since it should stay on for a real internet-facing WAN.

**3\. Reply-to.** Reply-to forced OPNsense to send replies out through the WAN gateway (my home router) instead of directly back to my laptop on the same network. The router never saw the original request, and since routers are stateful (they track connections from the start), it most likely dropped a reply to a connection it never saw. I disabled reply-to, and the page loaded. I did not capture which device dropped the reply, so the router is the likely cause rather than a proven one.

## Result

With all three changes in place and the firewall turned back on, the web UI loaded from my laptop. A connection with the firewall on is the real test, because it worked with the firewall off from the beginning.

## What I learned

- Firewall problems can stack: one layer fixed, and the symptom stays the same.  
- A stateful firewall only passes replies for connections it saw start, so asymmetric paths break connections even when the allow rule is right.  
- Check the evidence at each layer (log, packet capture, ruleset) instead of guessing, and prove the fix with the firewall enabled.  
- Settings that suit a lab WAN can be unsafe on a real one, so note why each was changed.

## Note on AI assistance

I built this lab with AI assistance (Claude) guiding the troubleshooting, and I am using projects like this to learn networking in small pieces. I ran every command and change myself. After the fix, I worked back through each of the three causes and explained it in my own words until I could describe what it does, why it blocked me, and how to tell it apart from an addressing problem.

## Tools used

OPNsense web UI, OPNsense console shell, `tcpdump`, `pfctl`, `curl`, Proxmox VE