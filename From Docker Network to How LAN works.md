# From Docker Network to LAN

The topic started from a practical problem: **how can I make my development environment safer?**

Currently, I have multiple web applications running on a single AWS Lightsail virtual machine. I wanted to manage external access through Nginx while keeping the individual application services hidden from the public internet.

As I learned more about Nginx and Docker networking, I discovered there are two common approaches.

- Option 1: Publish Container Ports

  A Docker service can publish ports using the ports section in docker-compose.yml.
  
  ```yml
  services:
    network-explorer:
      ports:
        - "3000:3000"
  ```
  I can publish the ports. It means docker listens on host machine using `HOST_PORT : CONTAINER_PORT` format, so Browser -> VM:3000 -> Container:3000 chain can work if port 3000 is also opened in the Lightsail firewall. Anyone with the VM's public IP address can access the application directly. 

- Option 2: Use Nginx as the Only Public Entry Point

  A more secure and scalable approach is to only expose Nginx publicly.
  
  I can expose other service ports only inside the docker network. This is a strategy that secure our enviroment by 2 gates, first is the Lightsail port remains closed, second is the host machine is not directly talking to containers, all routing should be from the nginx to those containers.

  ```
    Internet
        ↓
    443 / 80
        ↓
    Nginx
        ↓
    Docker Services
  ```
  
## Ports vs Expose - How does the port mapping mechnism work on VM

### Ports - Use for Nginx

ports publishes a container service onto the host machine.

Example:

```
ports:
  - "3000:3000"
```

Docker creates a listening socket on the virtual machine (host machine)

which forwards requests from VM:3000 to Container:3000

The service can now be reached through the VM if firewall rules permit it.

- 3000:3000 so user hits VM:3000 to go to container 1 directly
- 3939:3939 so user hits VM:3939 to go to container 4 directly

we will need to manage the route in nginx, 

```
nginx:
  ports:
    - "80:80"
    - "443:443"
```
so that:
- /network -> nginx dispatches it to docker name:port name, network:3000
- /freight -> nginx dispatches it to docker name:port name, freight-tool:3939

### Expose - Use for other docker services to communicate inside a docker network

expose makes a service available to other containers on the same Docker network.

Unlike ports, this does not create a listening port on the host machine.

```
expose:
  - "3000"
```

The service remains inaccessible from outside the Docker network unless another component, such as Nginx, forwards traffic to it.

## LAN Knowledge - A Private LAN Inside the VM (Docker network)

One of my biggest realizations was that a Docker network behaves similarly to a small private LAN operating entirely inside the virtual machine.

When building software, we often take for granted how data moves across a wire. I wanted to dig beneath the application layer and map out exactly how machines inside a Local Area Network (LAN) discover each other, manage traffic, and transfer files. 

Here is the breakdown of my journey into low-level networking mechanics.

---

### 1. The Core Architecture: Why We Need Switches

When you connect two local computers together directly with an Ethernet cable, file sharing using an application protocol like **SMB (Server Message Block)** is straightforward. However, this approach does not scale.

#### The Scaling Problem

Connecting every device directly to every other device requires an exponential explosion of hardware:
* To link **2 computers** directly → **1 cable** is needed.
* To link **4 computers** directly → **6 cables** and 3 ports per machine are needed.
* To link **8 computers** directly → **28 cables** and 7 ports per machine are needed.

#### The Solution: Star Topology with a Switch

To prevent an unmanageable mess of cables and hardware ports, a LAN uses a **Star Topology**. Every device plugs into a single, central **Switch**. 

```text
  [ Computer A ]      [ Computer B ]
         \                  /
          \                /
        [ CENTRAL SWITCH ] 
          /                \
         /                  \
  [ Computer C ]      [ Computer D ]
```

* **The Traffic Cop:** Instead of shouting data down every wire, the switch acts as an intelligent coordinator. It learns where devices are located and routes traffic explicitly between the sender and receiver ports.
* **Simultaneous Conversations:** Because the switch isolates port traffic, Computer A can send a file to Computer B at the exact same time Computer C sends a file to Computer D, without any data collisions.

---

### 2. Address Discovery: How ARP Solves the "Blind" Start

When you initiate a file transfer from Computer A to Computer C, Computer A usually only knows Computer C's human-readable network name or its local **IP Address** (e.g., `192.168.1.30`). It does **not** know its physical hardware ID (**MAC Address**). 

Because local network delivery relies on MAC addresses, the network triggers a protocol called **ARP (Address Resolution Protocol)** to bridge the gap.

#### The Step-by-Step ARP Lifecycle

1. **The Broadcast (The Shoutout):** Computer A creates an ARP request packet asking, *"Who has the IP address 192.168.1.30? Tell 192.168.1.20!"* It flags this packet as a **Broadcast**. The switch reads the broadcast flag and duplicates the message to **every single connected port**.
2. **The Filter:** Every device on the network evaluates the request. If the IP address does not match theirs, they drop the packet. 
3. **The Unicast (The Reply):** Only Computer C recognizes its own IP. It sends a direct, targeted reply (**Unicast**) back through the switch to Computer A: *"That's me! My MAC address is AA:BB:CC:11:22:33."*
4. **The ARP Cache (The Memory):** Computer A saves this mapping inside a local, temporary digital cheat sheet called an **ARP Cache**. For subsequent data transfers over the next few minutes, Computer A entirely skips the broadcast step and pulls the MAC address instantly from memory.

---

### 3. Differences: the Switch vs. the Router

One of my biggest network revelations was understanding exactly where the switch draws the line and where a router must step in. A standard switch is **completely blind to IP addresses**.

| Feature | The Network Switch | The Network Router |
| :--- | :--- | :--- |
| **OSI Layer** | **Layer 2** (Data Link Layer) | **Layer 3** (Network Layer) |
| **Core Brains** | Understands **MAC Addresses** and physical ports | Understands **IP Addresses** |
| **Internal Table** | **MAC-to-Port Table** (e.g., Port 3 → `AA:BB:CC:...`) | **Routing Table** & **ARP Table** (IP-to-MAC mappings) |
| **Primary Scope** | Connects devices *inside* the same local network | Connects entirely *different* networks (e.g., LAN to Internet) |
| **Analogy** | **Internal Mailroom Clerk:** Only checks room numbers and names to pass envelopes down the hall. | **GPS / Border Control:** Checks zip codes and country codes to route items out of the building across the highway. |

Because a standard switch operates at Layer 2, **it cannot build an IP-to-MAC map**. It cannot intercept or answer an ARP request on behalf of a computer. It must broadcast the request to let the true owner reply.

---

### 4. What is a Protocol? (The Blueprint of data)
If physical wires or Wi-Fi radio frequencies represent the highway, **protocols are the traffic laws, the language, and the vehicle blueprints combined.** Without protocols, physical connections are just chaotic spikes of electrical voltage or radio static.

Protocols handle four distinct assignments simultaneously:
1. **Signal Translation:** Defining exactly what voltage or frequency patterns equate to digital `1`s and `0`s (e.g., **Ethernet / Wi-Fi standards**).
2. **Data Packetization:** Slicing a massive 5 GB movie file into tiny, digestible segments wrapped in a digital "envelope" marked with sequence tracking numbers (e.g., **TCP**).
3. **Standardized Language:** Establishing rules for how distinct operating systems (Windows, macOS, iOS, Linux) talk to each other to check permissions, request files, and browse directories (e.g., **SMB**, **HTTP**).
4. **Error Correction:** Running mathematical sanity checks on arriving data envelopes. If a packet was corrupted by a wireless drop or electrical spike, the protocol automatically drops it and requests a clean copy from the sender.

#### The Anatomy of an Ethernet Frame Rule
To prove how rigid and elegant these protocol blueprints are, look at how data is organized sequentially when moving across a wire inside an **Ethernet Frame (IEEE 802.3)**:

```text
+----------+-----------------+----------------+--------+---------+-----+

| Preamble | Destination MAC |   Source MAC   |  Type  | Payload | FCS |
| (WakeUp) |    (6 Bytes)    |   (6 Bytes)    | (Type) | (Data)  |     |
+----------+-----------------+----------------+--------+---------+-----+
```

* **Destination MAC is FIRST:** The protocol strictly dictates putting the recipient's MAC address at the absolute front of the data stream. This allows the switch to read the destination instantaneously and route the signal to the correct port without waiting for the rest of the payload to arrive.
* **Payload:** This contains the actual piece of the file being sent.
* **FCS (Frame Check Sequence):** The mathematical safety check at the tail end to guarantee integrity.

---
