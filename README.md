<div id="top"></div>

<br />
<div align="center">
  <a href="https://forte.fit">
    <img src="forte-logo.png" alt="FORTË Logo" width="300">
  </a>

  <h3 align="center">Agora RTMP Server Configuration</h3>
</div>


This note explains the process of setting up a new agora RTMP server and references why these decisions were made

---
## Load Balancers and Configuration

Each RTMP server is deployed for redundancy ([ref](https://fortefit.atlassian.net/browse/DEV-19?atlOrigin=eyJpIjoiODU1NTFlMjBmYzVkNDU3OGEzODFjNDdkYjY1OTM0YTEiLCJwIjoiaiJ9)) behind a load balancer. 

Find [load balancers here >>](https://portal.azure.com/#view/Microsoft_Azure_Network/LoadBalancingHubMenuBlade/~/loadBalancers)

| Load balancer | Location     | Frontend Ip    |
|---------------|--------------|----------------|
| QA-LB         | EAST US      | 52.254.75.4    |
| QA-LB-AUS     | AUS          | 20.213.248.10  |
| QA-LB-EU      | North Europe | 52.156.204.252 |

## Load Balancer rule
>A load-balancing rule maps a the LB frontend IP  and port to all the instances within the [backend pool](https://docs.microsoft.com/en-us/azure/load-balancer/components#backend-pool).

- **Action**: point RMTP port (1935) from target LB frontend Ip to target backend

## Inbound NAT Rule
>An inbound NAT rule forwards incoming traffic sent to a specific virtual machine or instance in the backend pool. 

- **Action**: Create SSH rule `FRONTEND-IP:PORT <---->VM:22`

## Outbound Rule
>An outbound rule configures outbound NAT for all virtual machines or instances identified by the backend pool.

- Agora (Ben) recommends that all outbound port/protocol be open 

- **Action**: Create outbound rule to open all ports

---

## Deploying a VM

A new virtual machine can be deployed on the azure portal. 

NB: 
VM specification
- OS: Linux (ubuntu 20.04)
- SSH [RMTP Key here](https://portal.azure.com/#@opsforte.onmicrosoft.com/resource/subscriptions/8dcb675f-f4f6-4659-afc3-81c779dd6266/resourceGroups/shared/providers/Microsoft.Compute/sshPublicKeys/rtmp-20-keys/overview)
- VM should not be assigned a public IP 
- Check box to place machine behind an existing load balancer
  ![Alt](/lb.png "load")
- All VMs in a backend pool must be in the same virtual network, note when selecting target VNET. 

### VM Networking

- Set inbound NAT rule to open 22 ,80, 433 and 1935
- Set Outbound rule to open all port


# Agora SDK setup 

To install the agora SDK run the following command on the newly created VM.

```
```