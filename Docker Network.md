# Docker Network :
#### They connect container with other container and connect container to the host machine
### NOTE :
Multiple containers inside the same host machine can communicate throught `BRIDGE` Topology
The default topology is BRIDGE . Containers created by `BRIDGE NETWORK` can communicate with each other using `IP address`.It will be used in communication within the same cluster
and cannot communicate with other .

![Bridge](./Bridge%20Network.png)

## How it works ?
- Each container on the default bridge network gets an IP from a private subnet (usually 172.17.x.x) managed by Docker’s internal NAT.
- Containers can talk to each other on that bridge network using these private IPs (or container names if DNS is enabled).
- From outside (host machine or internet), you can’t directly access those IPs — you must publish ports using -p or --publish.

## Example :

`docker run -d --name myapp nginx`

- Container gets a private IP, e.g., 172.17.0.2 (visible with docker inspect myapp).
- This IP works only inside the Docker host.
- This IP works only inside the Docker host.

`docker run -d -p 8080:80 --name myapp nginx`

-Now, port 8080 on your host machine is mapped to the container’s port 80.

In a cluster (like ECS or Kubernetes)

- Bridge network still means private container IPs.
- Public access is handled via:
  
 Host port mapping (ECS task definition / Kubernetes Service NodePort)

  Load balancer (ALB/NLB)

  Elastic IP assigned to a host


