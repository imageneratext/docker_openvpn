# 🔒 Docker openvpn server

- Create a [openvpn](https://openvpn.net/) server for connect to 🔗 private networks

- This project uses [kylemanna openvpn](https://hub.docker.com/r/kylemanna/openvpn) 🐋 docker image

## ℹ️ Context

### 😬 Problem

- We want 🚀 deploy an docker application in the ☁️ cloud **restricting** the access via _VPN_ (it is an internal company app)

- In the clients we only want to ↪️ redirect the **traffic** to the _VPN_ when we go to the url of the internal application, the rest going through the 📍 local network

### 💼 Solution

- Create a _VPN_ server and specify the _IP_ 🛣️ route of the internal app

- In the internal app server, we restrict all 🔌 connections _IPs_ except of the _VPN_ server (image from [heavymetaldev](https://heavymetaldev.com/openvpn-with-docker))

![vpn_diagram](https://user-images.githubusercontent.com/22328176/126044983-3883e6e1-276c-430d-8610-850a425fc562.png)

## ⚙️ Configure open vpn server

### 🐧 Intall linux dependencies

```shell
sudo apt update && sudo apt install -y docker.io docker-compose
sudo usermod -aG docker $USER && newgrp docker
```

📝 Note: We **recommend** not setup with root user (you can create user with sudo permissions following next [🦮 guide](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04))

### 📥 Clone project

```shell
git clone https://github.com/imageneratext/docker_openvpn.git
```

### 👨‍🔧 Configure open-vpn

```shell
export PUBLIC_SERVER_IP=$(curl ifconfig.me.)
export ROUTE="route 222.222.222.222 255.255.255.255"
docker-compose run --rm openvpn ovpn_genconfig -N -d -u udp://${PUBLIC_SERVER_IP} -p "route 172.17.0.0 255.255.0.0" -p ${ROUTE}
```

📝 Notes:

- `PUBLIC_SERVER_IP` is the 📍 public _IP_ of _VPN_ server (it could specify the domain)
- `ROUTE` indicates the _domain/IP_ which the _VPN_ will 🛣️ route the traffic from client (it can be a _IPs_ range like `ROUTE="route 222.222.222.0 255.255.255.0"` or several `-p` arguments). Once run, we also can add routes ✏️ editing the config file `openvpn-data/conf/openvpn.conf`
- The route `172.17.0.0 255.255.0.0` is the default 🐋 docker subnet

### 🔑 Create CA key passphrase

Run the next command and set a _CA_ passphrase (it ask ✅ serveral comfirmations)

```shell
docker-compose run --rm openvpn ovpn_initpki
```

### 🆙 Up open-vpn server

```shell
docker-compose up -d openvpn
```

### 👤 Create and copy client certificates

- ➕ Generate one providing a **password** for the client and specifying the _CA_ **passphrase**

  ```shell
  export CLIENT_NAME="client_1"
  ssh user@public_server_ip "cd docker_openvpn && docker-compose run --rm openvpn easyrsa build-client-full $CLIENT_NAME"
  ```

  📝 Note: To generate it without password add `nopass` argument

- 📥 Get and copy `.ovpn` file to local host

  ```shell
  ssh user@public_server_ip "cd docker_openvpn && docker-compose run --rm openvpn ovpn_getclient $CLIENT_NAME" > $CLIENT_NAME.ovpn
  ```

### 🧹 Revoke client certificates

```shell
# Keep the corresponding certificate, key and files
ssh user@public_server_ip "cd docker_openvpn && docker-compose run --rm openvpn ovpn_revokeclient $CLIENT_NAME"

# Remove the corresponding certificate, key and req files
ssh user@public_server_ip "cd docker_openvpn && docker-compose run --rm openvpn ovpn_revokeclient $CLIENT_NAME remove"
```

## 💻 Configure the client

- 🔛 Enable client _VPN_ via shell

  ```shell
  sudo apt-get install openvpn
  sudo openvpn --config "$CLIENT_NAME.ovpn"
  ```

### 🖱️ Configure client with _GUI_ (Ubuntu)

1. Install _network-manager-openvpn_

   ```shell
   sudo apt-get -y install network-manager-openvpn
   ```

2. Open 📶 network settings and add a new _VPN_ target

3. Click in _"Import from file"_. <details><summary>See image</summary>![vpn_settings_ubuntu_](https://user-images.githubusercontent.com/22328176/126045438-8a314b4e-819c-4832-bf65-a1e4d35d5ec8.png)</details>

4. Set the user 🔑 password. <details><summary>See image</summary>![pass_vpn_settings](https://user-images.githubusercontent.com/22328176/126045431-23ae3f16-e6c6-4360-b5f0-c856349e3a32.png)</details>

5. Go to _IPv4_ section and ✅ check _"Use this connection only for resources on its network"_ (this let us ↪️ redirect to _VPN_ only traffic of routes added). <details><summary>See image</summary>![ipv4_vpn_setting](https://user-images.githubusercontent.com/22328176/126045421-a7c1a4f7-e6b5-4cde-8386-44f64ce010d2.png)</details>

For automatically 🔛 turn on VPN

1. 🐚 Run `nm-connection-editor`

2. ➡ Click in _"Wired connection 1"_. <details><summary>See image</summary>![network_connection](https://user-images.githubusercontent.com/22328176/134207485-d72481a9-3649-4094-a608-421257dd818d.png)</details>

3. Go to _"General"_ tab, ☑️ check _"Automatically connect to VPN"_ and choose the desired connection. <details><summary>See image</summary>![wired_connection](https://user-images.githubusercontent.com/22328176/134207430-938d2bba-07e3-44da-a452-dabe8657fea4.png)</details>

4. Ensure ✅ check "Store the password for all users" in vpn settings to avoid [secrets request errors](https://unix.stackexchange.com/a/436130/479835). <details><summary>See image</summary>![pass_save_vpn_config](https://user-images.githubusercontent.com/22328176/135126949-7bc87a02-8b09-4065-b143-1460af6f3824.png)</details>

## 📱 Configure internal app

- Check external interface (e.g: `eth0`)

  ```shell
  ip route list default
  # eg output: default via 139.59.160.1 dev eth0 proto static
  ```

- 🔐 Restricts connections to all _IPs_ except of the _VPN_ server via `iptables` how say in docker [📘 doc](https://docs.docker.com/network/iptables/#restrict-connections-to-the-docker-host)

  ```shell
  sudo iptables -I DOCKER-USER -i eth0 ! -s 111.111.111.111 -j DROP
  ```

  📝 Note: This restrict **outbound** connections during image 🏢 building, follow this [🦮 guide](https://medium.com/swlh/how-to-whitelist-ip-addresses-to-access-desired-docker-containers-5f6c8fcfa7f6) or configure 🧑‍🚒 firewall rules in your cloud service for restrict it

- ❤️ Useful commands

  ```shell
  # to show iptables rules
  sudo iptables -L --line-numbers

  # to remove iptables rules
  sudo iptables -D DOCKER-USER -i eth0 ! -s 111.111.111.111 -j DROP
  ```

## 🖇️ References

- [💬 Issue](https://github.com/kylemanna/docker-openvpn/issues/288) and [📙 tutorial](https://heavymetaldev.com/openvpn-with-docker) for specify openvpn routes

- Kylemanna openvpn docker image [📄 doc](https://github.com/kylemanna/docker-openvpn#openvpn-for-docker) and [▶️ video](https://www.youtube.com/watch?v=Ulew2JHUHfE) tutorial

- Kylemanna openvpn [🐙 docker-compose doc](https://github.com/kylemanna/docker-openvpn/blob/master/docs/docker-compose.md)
