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
| ------------- | ------------ | -------------- |
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

The current deployed vm has port 2222 for SSH via the frontend Ip

| Load balancer | Location     | Frontend Ip    | Link    |
| ------------- | ------------ | -------------- | -------------- |
| RTMP-20-A         | North America      | 52.254.75.4    |[Azure](https://portal.azure.com/#@opsforte.onmicrosoft.com/resource/subscriptions/8dcb675f-f4f6-4659-afc3-81c779dd6266/resourceGroups/shared/providers/Microsoft.Compute/virtualMachines/RTMP-20-A/overview)|
| RTMP-20-B     | EU          | 52.156.204.252  |[Azure](https://portal.azure.com/#@opsforte.onmicrosoft.com/resource/subscriptions/8dcb675f-f4f6-4659-afc3-81c779dd6266/resourceGroups/shared/providers/Microsoft.Compute/virtualMachines/RTMP-20-B/overview)|
| RTMP-20-C      | AUS | 20.213.248.10 |[Azure](https://portal.azure.com/#@opsforte.onmicrosoft.com/resource/subscriptions/8dcb675f-f4f6-4659-afc3-81c779dd6266/resourceGroups/shared/providers/Microsoft.Compute/virtualMachines/RTMP-20-C/overview)|
| RTMP-20-QA      | North America | 20.232.116.15 |[Azure](https://portal.azure.com/#@opsforte.onmicrosoft.com/resource/subscriptions/8dcb675f-f4f6-4659-afc3-81c779dd6266/resourceGroups/shared/providers/Microsoft.Compute/virtualMachines/RTMP-20-qA/overview)|

`ssh -i -/. ssh/rtmp-20-keys pem forte@FrontEndIP -p 2222`

**A new virtual machine can be deployed on the azure portal.** 

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


## Agora SDK setup 

To install the agora SDK run the following command on the newly created VM. Run the following commands

```
 cd /home/forte
```
```
wget https://github.com/AgoraIO-Solutions/server_side_custom_video_source/raw/master/release/dist-agora-rtmp.tar.gz
```
```
tar -xvzf dist-agora-rtmp.tar.gz
```
```
cd dist-agora-rtmp
```
```
sudo ./install.sh
```
Reboot the box with `sudo reboot`

## Test

Test the stream at rtmp://FrontendIP:1935/live?appid=xxxxxxxxxxxxxxxxxxxxx&channel=xxxxxxxxxxxxxxxxxxxxx&uid=2959968645&abr=150000&dual=true&dfps=24&dvbr=500000&dwidth=640&dheight=360&end=true

## Datadog integration

REF:
- https://docs.datadoghq.com/agent/logs/?tab=tailfiles
- https://docs.datadoghq.com/agent/logs/?tab=tailfiles

Install Datadog agent (currently v7) using

```shell
DD_AGENT_MAJOR_VERSION=7 DD_API_KEY=8cbeadcadc78159956f4814379b8f61b DD_SITE="us3.datadoghq.com" bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"
```
You can update datadog integration using the Datadog Agent configuration: /etc/datadog-agent/datadog.yaml

To collect live tail agora logs: 
- Create a custom agora configuration `mkdir -p /etc/datadog-agent/conf.d/agora.d`
- Create a tail log configuration file directed at /tmp/agora.log `nano conf.yaml`

```
logs:
  - type: file
    path: "/tmp/agora.log"
    service: “RTMPG”
    source: “AGORA_QA”
```

- Activate live tail on agent configuration file `logs_enabled: false`
- Restart datadog agent `systemctl restart datadog-agent`

