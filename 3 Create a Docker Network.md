# Docker Networking and Port Mapping Tutorial

This tutorial will demonstrate how Docker networking works, specifically focusing on port mapping concepts illustrated in the image. We'll create a practical example to show how traffic flows from your host machine, through Docker's networking components, and to containers.



## Understanding the Diagram

The diagram shows:
1. **Docker server** with interface eth0 (10.0.0.7)
2. **Virtual network** with containers connected via docker0 bridge (172.16.23.7)
3. **NAT** (Network Address Translation) handling the mapping between host and container networks
4. **docker-proxy** process that facilitates port forwarding
5. **Container 1** exposing TCP port 80
6. Traffic flow showing both inbound (to container) and outbound (from container) requests

## Tutorial: Demonstrating Docker Port Mapping


### Prerequisites
- Docker installed on your system
- Basic understanding of terminal/command line

### Part 1: Setting Up Our Environment

First, let's create a simple network setup that reflects the diagram:
#### Create a custom bridge network with a specific subnet
```bash
docker network create --driver bridge --subnet=172.16.23.0/24 demo-network
```

#### Now let's verify the network was created:

```bash
docker network ls
```
#### ตรวจสอบว่า "demo-network" ตั้งค่าไว้อย่างไร
```bash
docker network inspect demo-network
```
#### อธิบายผลลัพธ์
ผลลัพธ์นี้แสดงรายละเอียดของเครือข่ายเสมือนที่นักศึกษาสร้างขึ้นในระบบ Docker ชื่อ "demo-network" นักศึกษาอาจนึกภาพเครือข่ายนี้เหมือนสร้างถนนส่วนตัวเพื่อให้คอนเทนเนอร์ (หรือบ้านเล็กๆ) ของนักศึกษาใช้สื่อสารกัน เครือข่ายนี้ถูกสร้างเมื่อวันที่ 31 มีนาคม 2025 และใช้ระบบ "bridge" ซึ่งเป็นเหมือนสะพานที่เชื่อมระหว่างคอนเทนเนอร์กับโลกภายนอก นักศึกษาได้กำหนดให้เครือข่ายนี้ใช้ที่อยู่ IP ในช่วง 172.16.23.0 ถึง 172.16.23.255 (คล้ายกับการจองเลขบ้านในหมู่บ้านของนักศึกษา) ขณะนี้ยังไม่มีคอนเทนเนอร์หรือ "บ้าน" ใดๆ ตั้งอยู่บนถนนนี้ ซึ่งเห็นได้จากส่วน "Containers" ที่ว่างเปล่า เครือข่ายนี้ไม่ได้ถูกล็อกเป็นส่วนตัวภายใน (internal) คอนเทนเนอร์ที่อยู่ในเครือข่ายนี้จึงสามารถเชื่อมต่อกับอินเทอร์เน็ตหรือเครือข่ายอื่นๆ ได้ ทั้งหมดนี้ทำให้นักศึกษามีพื้นที่เฉพาะสำหรับให้คอนเทนเนอร์ของนักศึกษาสื่อสารกันอย่างปลอดภัยและเป็นระเบียบ
### Part 2: Creating and Connecting Containers

#### Container 1: Web Server (Similar to Container 1 in the diagram)

```bash
# Run an nginx container that will serve web content on port 80
docker run -d --name web-server --network demo-network -p 10520:80 nginx:latest
```

This command:
- Creates a container named "web-server"
- Connects it to our demo-network
- Maps port 10520 on the host to port 80 in the container (similar to the diagram's inbound request)

#### Container 2: Observer Container

```bash
# Run an Alpine container that we'll use to observe network traffic
docker run -d --name observer --network demo-network alpine sleep infinity
```

### Part 3: Examining the Network Configuration

Let's examine what Docker has set up:

```bash
# List running containers with network info
docker ps
docker inspect -f '{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web-server
docker inspect -f '{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' observer
```

Now, let's check the docker-proxy process (the component in the diagram responsible for port forwarding):

```bash
# On Linux
ps aux | grep docker-proxy

# On Windows
netstat -ano | findstr 10520
```

You should see a docker-proxy process listening on port 10520, forwarding traffic to the container.

### Part 4: Testing Inbound Connection Flow

Let's trace the path of a request as shown in the diagram:

1. **Inbound request to exposed port** (Host to Container)

```bash
# Access the web server through the mapped port
curl http://localhost:10520
```

Here's what happens behind the scenes (following the red arrows in the diagram):
1. The request hits your host's eth0 interface (10.0.0.7 in the diagram)
2. Docker's NAT routing sends it to the docker-proxy
3. docker-proxy forwards it to the docker0 bridge (172.16.23.7)
4. The bridge routes it to Container 1 on port 80

### Part 5: Testing Outbound Connection Flow

Now let's test the outbound flow (Container to Outside - green arrows in diagram):

1. First, install curl in our observer container:

```bash
docker exec -it observer sh -c "apk add --no-cache curl"
```

2. Make a request from the observer container to the web server:

```bash
# This simulates internal container-to-container communication
docker exec -it observer curl http://web-server:80
```

3. Make a request from the container to an external site:

```bash
# This simulates outbound traffic from container to internet
docker exec -it observer curl https://example.com
```

Here's what happens (following the green arrows in the diagram):
1. For container-to-container: The request goes directly through the docker0 bridge
2. For outbound internet requests: The traffic goes through docker0, then NAT, and finally through the host's network interface

### Part 6: Visualizing Network Traffic

To better visualize what's happening, let's run a continuous ping and see the network path:

```bash
# Install tools in observer container
docker exec -it observer sh -c "apk add --no-cache iputils tcptraceroute"

# Trace route to the web server
docker exec -it observer traceroute web-server

# If external access is needed
docker exec -it observer traceroute example.com
```

### Part 7: Advanced Port Mapping Demonstration

Let's set up multiple port mappings to better understand the concept:

```bash
# Stop and remove existing web-server
docker stop web-server
docker rm web-server

# Create a new container with multiple port mappings
docker run -d --name multi-port-server --network demo-network \
  -p 10520:80 -p 10521:8080 \
  nginx:latest
```

Now let's modify the nginx container to also listen on port 8080:

```bash
# Copy the default nginx config to the container
docker cp multi-port-server:/etc/nginx/conf.d/default.conf ./default.conf

# Add a second server block for port 8080
cat > ./default.conf << EOF
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}

server {
    listen       8080;
    listen  [::]:8080;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        add_header Content-Type text/plain;
        return 200 'This is coming from port 8080\n';
    }
}
EOF

# Copy the updated config back and reload nginx
docker cp ./default.conf multi-port-server:/etc/nginx/conf.d/default.conf
docker exec multi-port-server nginx -s reload
```

Now test both port mappings:

```bash
# Test the original port mapping (10520 -> 80)
curl http://localhost:10520

# Test the new port mapping (10521 -> 8080)
curl http://localhost:10521
```

### Part 8: Understanding the Docker Proxy in Detail

The docker-proxy component in the diagram is crucial for port mapping. Let's examine it more closely:

```bash
# For each mapped port, there's a docker-proxy process
ps aux | grep docker-proxy  # On Linux
netstat -ano | findstr docker  # On Windows
```

Each docker-proxy listens on the host interface (eth0 in the diagram) for connections to the mapped port, then forwards traffic to the appropriate container.

### Part 9: Cleanup

Let's clean up our resources:

```bash
docker stop multi-port-server observer
docker rm multi-port-server observer
docker network rm demo-network
rm ./default.conf
```

## How This Relates to the Diagram

The tutorial has simulated all components in the diagram:

1. **eth0 interface (10.0.0.7)**: This is your host's network interface
2. **NAT**: Docker's built-in NAT handles the address translation
3. **docker-proxy**: We saw this process in action, listening on host ports
4. **docker0 (172.16.23.7)**: We created a custom bridge network with a similar address range
5. **Container 1 (TCP 80)**: Our Nginx container exposed port 80
6. **Inbound requests**: We tested these with curl from host to container
7. **Outbound requests**: We tested these from container to outside world

By following this tutorial, you've experienced firsthand how Docker networking and port mapping work, just as illustrated in the diagram.
