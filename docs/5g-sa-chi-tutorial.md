# Bring Up an Standalone 5G Network on ExPECA Chameleaon


Assume we have reserved a worker node with 3 free baremetal interfaces: `ens5f0`, `eno12399np0`, and `eno12409np1`. We choose interface `ens5f0` for the core network, `eno12399np0` for gNodeB's SDR, and `eno12409np1` for nrue's SDR.
The core network will use `192.168.70.128/26` subnet on `ens5f0`. The following services must be run respectively:

## A. Core Network

### 1. MySQL

name: `5gcn-1-mysql`\
image: `samiemostafavi/expeca-mysql`\
network: `oai-5gcn-net`\
labels: 
```
networks.1.interface=ens5f0,networks.1.ip=192.168.70.131/26
```
	
### 2. NRF

if want to run with nrf

name: `5gcn-1-nrf`\
image: `samiemostafavi/expeca-nrf`\
network: `oai-5gcn-net`\
environment variables: 
```
NRF_INTERFACE_NAME_FOR_SBI=net1
```
labels:
```
networks.1.interface=ens5f0,networks.1.ip=192.168.70.130/26
```
	
### 3. UDR

name: `5gcn-2-udr`\
image: `samiemostafavi/expeca-udr`\
network: `oai-5gcn-net`\
environment variables with `nrf`:
```
UDR_INTERFACE_NAME_FOR_NUDR=net1,USE_FQDN_DNS=no
```
environment variables without `nrf`:
```
UDR_INTERFACE_NAME_FOR_NUDR=net1,USE_FQDN_DNS=no,REGISTER_NRF=no
```
labels: 
```
networks.1.interface=ens5f0,networks.1.ip=192.168.70.136/26
```
We add `REGISTER_NRF=no` to env variables if don't want to run nrf

### 4. UDM
	
name: `5gcn-3-udm`\
image: `samiemostafavi/expeca-udm`\
network: `oai-5gcn-net`\
environment variables with `nrf`:
```
SBI_IF_NAME=net1,USE_FQDN_DNS=no
```
environment variables without `nrf`:
```
SBI_IF_NAME=net1,USE_FQDN_DNS=no,REGISTER_NRF=no
```
labels: 
```
networks.1.interface=ens5f0,networks.1.ip=192.168.70.137/26
```
We add `REGISTER_NRF=no` to env variables if don't want to run nrf

### 5. AUSF

name: `5gcn-4-ausf`\
image: `samiemostafavi/expeca-ausf`\
network: `oai-5gcn-net`\
environment variables with `nrf`: 
```
SBI_IF_NAME=net1,USE_FQDN_DNS=no
```
environment variables without `nrf`: 
```
SBI_IF_NAME=net1,USE_FQDN_DNS=no,REGISTER_NRF=no
```
labels: 
```
networks.1.interface=ens5f0,networks.1.ip=192.168.70.138/26
```
We add `REGISTER_NRF=no` to env variables if don't want to run nrf

### 6. AMF

name: `5gcn-5-amf`\
image: `samiemostafavi/expeca-amf`\
network: `oai-5gcn-net`\
environment variables with `nrf`: 
```
AMF_INTERFACE_NAME_FOR_NGAP=net1,AMF_INTERFACE_NAME_FOR_N11=net1,USE_FQDN_DNS=no
```
environment variables without `nrf`: 
```
AMF_INTERFACE_NAME_FOR_NGAP=net1,AMF_INTERFACE_NAME_FOR_N11=net1,USE_FQDN_DNS=no,NF_REGISTRATION=no,SMF_SELECTION=no
```
labels: 
```
networks.1.interface=ens5f0,networks.1.ip=192.168.70.132/26
```
We add `NF_REGISTRATION=no,SMF_SELECTION=no` to env variables if don't want to run nrf

### 7. SMF

name: `5gcn-6-smf`\
image: `samiemostafavi/expeca-smf`\
network: `oai-5gcn-net`\
environment variables with `nrf`: 
```
USE_FQDN_DNS=no,SMF_INTERFACE_NAME_FOR_N4=net1,SMF_INTERFACE_NAME_FOR_SBI=net1
```
environment variables without `nrf`: 
```
USE_FQDN_DNS=no,SMF_INTERFACE_NAME_FOR_N4=net1,SMF_INTERFACE_NAME_FOR_SBI=net1,REGISTER_NRF=no,DISCOVER_UPF=no
```
labels: 
```
networks.1.interface=ens5f0,networks.1.ip=192.168.70.133/26
```
We add `REGISTER_NRF=no,DISCOVER_UPF=no` to env variables if don't want to run nrf. Make sure the new `child-entrypoint.sh` runs and adds `oai-spgwu` ip address to `/etc/hosts`.
	
### 8. SPGWU
	
This service is responsible for the 5G egress point. It must be running with more capabalities and permissions compared to the other services:
```
cap_add:
    - NET_ADMIN
    - SYS_ADMIN
cap_drop:
    - ALL
privileged: true
```
Create the container in Openstack with the following parameters

name: `5gcn-7-spgwu`\
image: `samiemostafavi/expeca-spgwu`\
network: `oai-5gcn-net`\
environment variables with `nrf`:
```
SGW_INTERFACE_NAME_FOR_S1U_S12_S4_UP=net1,SGW_INTERFACE_NAME_FOR_SX=net1,PGW_INTERFACE_NAME_FOR_SGI=net1,USE_FQDN_NRF=no
```
environment variables without `nrf`:
```
SGW_INTERFACE_NAME_FOR_S1U_S12_S4_UP=net1,SGW_INTERFACE_NAME_FOR_SX=net1,PGW_INTERFACE_NAME_FOR_SGI=net1,USE_FQDN_NRF=no,REGISTER_NRF=no
```
labels:
```
networks.1.interface=ens5f0,networks.1.ip=192.168.70.134/26,capabilities.privileged=true,capabilities.add.1=NET_ADMIN,capabilities.add.2=SYS_ADMIN,capabilities.drop.1=ALL
```

We add `REGISTER_NRF=no` to env variables if don't want to run nrf.
	
## B. Radio Access Network
	
### 1. gNodeB
	
name: `5g-gnodeb-1`\
image: `samiemostafavi/expeca-oai-gnb`\
networks: 1) `oai-cn5g-net`, 2) `sdr1-net`\
env variables:
```
USE_SA_TDD_MONO_E320=yes,GNB_ID=e00,GNB_NAME=gNB-OAI,MCC=001,MNC=01,MNC_LENGTH=2,TAC=1,NSSAI_SST=1,NSSAI_SD=1,AMF_IP_ADDRESS=192.168.70.132,GNB_NGA_IF_NAME=net1,GNB_NGA_IP_ADDRESS=192.168.70.139,GNB_NGU_IF_NAME=net1,GNB_NGU_IP_ADDRESS=192.168.70.139,ATT_TX=0,ATT_RX=0,MAX_RXGAIN=114,SDR_ADDRS=addr=10.40.3.1,THREAD_PARALLEL_CONFIG=PARALLEL_SINGLE_THREAD,USE_ADDITIONAL_OPTIONS=--sa --usrp-tx-thread-config 1 -E --gNBs.[0].min_rxtxtime 6
```
Note: you can try with `--continuous-tx` for better timing stability.

labels:
```
networks.1.interface=ens5f0,networks.1.ip=192.168.70.139/26,networks.2.interface=eno12399np0,networks.2.ip=10.40.2.1/16,capabilities.privileged=true,resources.limits.memory=32000Mi,resources.limits.cpu=15,resources.requests.memory=32000Mi,resources.requests.cpu=15
```
	
### 2. nrUE
	
name: `5g-nrue-1`\
image: `samiemostafavi/expeca-oai-nr-ue`\
network: `sdr2-net`\
env variables:
```
FULL_IMSI=001010000000001,FULL_KEY=fec86ba6eb707ed08905757b1bb44b8f,OPC=C42449363BBAD02B66D16BC975D77CC1,DNN=oai,NSSAI_SST=1,USE_ADDITIONAL_OPTIONS=-r 106 --numerology 1 --band 78 -C 3619200000 --nokrnmod --sa -E --uicc0.imsi 001010000000001 --uicc0.nssai_sd 1 --usrp-args "addr=10.40.3.2" --ue-fo-compensation --ue-rxgain 120 --ue-txgain 0 --ue-max-power 0
```

labels:
```
networks.1.interface=eno12409np1,networks.1.ip=10.40.2.1/16,capabilities.privileged=true,resources.limits.memory=32000Mi,resources.limits.cpu=10,resources.requests.memory=32000Mi,resources.requests.cpu=10
```

# References

https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_CN5G_gNB_USRP_COTS_UE_Tutorial.md
