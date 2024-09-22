# homeassistant

## Setting up homekit bridge

In order to pair iOS devices with homeassistant I need to setup a [HomeKit
Bridge](https://www.home-assistant.io/integrations/homekit/) integration. iOS devices will try to discover
homeassistant using [mDNS](https://en.wikipedia.org/wiki/Multicast_DNS) querying `homeassistant.local`. If your network
setup does not support those kind of queries you won't be able to integrate iOS devices with homeassistant.

Since my network uses an [OpenWRT](https://openwrt.org) router, I had to setup an `avahi-daemon` in order to support
mDNS queries on my local network. After following this guide: https://blog.christophersmart.com/2020/03/30/resolving-mdns-across-vlans-with-avahi-on-openwrt/
I finally was able to call `ping homeassistant.local` and resolve to my homeassistant instance.
 
