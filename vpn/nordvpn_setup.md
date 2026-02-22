# NordVPN Setup

## Installation
curl -sSf https://downloads.nordcdn.com/apps/linux/install.sh | sh
nordvpn login --token YOUR-TOKEN

## Configuration
nordvpn set killswitch on
nordvpn set lan-discovery enable
nordvpn whitelist add subnet 10.0.0.0/24
nordvpn connect

## qBittorrent Binding
In qBittorrent: Tools → Options → Advanced → Network Interface → nordlynx
