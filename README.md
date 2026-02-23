# Zero Trust Testbed with Dynamic and Context Aware Trust Algorithm

## Project Overiew and Implementation Guide

*This guide details the implementation of a Zero Trust architecture using SDN, policy-based access control, and microsegmentation.
The system integrates identity management, network control, and application-level enforcement to create a dynamic, context-aware security framework.*
---
## Architecture Components
1. Host System: Ubuntu 24.04 LTS with SDN support
2. Containers and Virtualization: Docker, VMware Workstation, GNS3 VM and Mininet VM
3. Network Control: Open vSwitch + OpenDaylight
4. Identity Management: Keycloak
5. Policy Engine: Open Policy Agent (OPA)
6. Application Proxy: Envoy
7. Network Emulation: Mininet, GNS3
8. *High Fidelity Testing and Emulation: GNS3 --Could be future work* 
---

| Component | Purpose | Port | Access |
| :--- | :--- | :--- | :--- |
| **Host System** | Ubuntu 24.04 base with SDN support | - | Local |
| **Open vSwitch** | Network layer microsegmentation | 6653 | Local |
| **OpenDaylight** | SDN controller & policy administrator | 8181 | [http://localhost:8181](http://localhost:8181) |
| **Keycloak** | Identity & device trust anchor | 8080 | [http://localhost:8080](http://localhost:8080) |
| **Open Policy Agent** | Policy decision engine | 8182 | [http://localhost:8182](http://localhost:8182) |
| **Envoy Proxy** | Application-level enforcement | 10000 | [http://localhost:10000](http://localhost:10000) |
| **Mininet** | Network topology emulation | - | CLI |
---
Phase 1: Initialization ($t=0$)
State: Tabula Rasa. The system has exactly one data point per parameter.
Mathematical Consequence: Variance calculation requires at least two data points ($N \ge 2$). $$ \sigma^2 = \frac{\sum (x_i - \mu)^2}{N} $$ At $t=0$, $\sigma^2$ is mathematically undefined or zero.
Functional Weighting: The weighting function $W = \frac{1}{1 + \alpha \sigma^2}$ defaults to $W_{max}$ (or uniform weights).
Significance: Step 0 represents the "Naive Trust" assessment. It reflects the pure sensor readings without the context of stability. A fluctuating sensor that happens to be "high" at $t=0$ will be fully trusted, potentially leading to a Type I error (False Positive) only at this specific instant.
---
# Resource-Limited Zero Trust Testbed

A lightweight, virtualization-based testbed for implementing and validating Zero Trust Architecture (ZTA) principles.
This project utilizes OpenDaylight, Mininet, Open vSwitch, Keycloak, and OPA to demonstrate micro-segmentation, identity-based access control, and dynamic policy enforcement.

## 📋 Table of Contents
- [Prerequisites & Host Prep](#prerequisites--host-prep)
- [Installation Guide](#installation-guide)
  - [Docker & Docker Compose](#1-install-docker--docker-compose)
  - [Network Layer (OVS & Mininet)](#2-install-network-layer-pep--emulation)
- [Component Deployment](#component-deployment)
  - [OpenDaylight (SDN Controller)](#3-deploy-opendaylight-policy-administrator)
  - [Keycloak (Identity Provider)](#4-deploy-keycloak-identity--trust-anchor)
  - [Open Policy Agent (Policy Engine)](#5-deploy-open-policy-agent-opa)
  - [Envoy Proxy (Application PEP)](#6-deploy-envoy-proxy)
- [Topology & Validation](#topology--validation)
- [Advanced Policy Scenarios](#advanced-policy-scenarios)

---

## Prerequisites & Host Prep

**Goal:** Establish a stable Linux base with SDN support.

**System Requirements:**
* **OS:** [Ubuntu Server 24.04 LTS](https://ubuntu.com/download/server)
* **RAM:** 16 GB (Recommended)
* **CPU:** 4 Cores
* **Virtualization:** Nested virtualization enabled (if running inside a VM).

## Installation Guide

**Goal:** Install Necessary Containers and Applications.

  ## Initial Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git net-tools bridge-utils
```
---
  **1. Install Docker and Add Docker User:**
  
  ```bash
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo usermod -aG docker $USER # add current user to the docker group
newgrp docker # create the docker user group
```
  *** Verify Installation:*** 
```bash
docker run hello-world
```
---
**2. Install Network Layer (PEP & Emulation)**

**Goal:** Open vSwitch (OVS): Core micro-segmentation and flow enforcement.

```bash
sudo apt install -y openvswitch-switch
sudo systemctl start openvswitch-switch
ovs-vsctl show
```
**3. Mininet Installation:**

**Goal:** Lightweight, deterministic network topology control.

```bash
sudo apt install -y mininet
sudo mn --test pingall
```
---

## Component Deployment

**4. Deploy OpenDaylight (Policy Administrator)**

**Goal:** Centralized SDN control translating trust decisions to flows.

```bash
docker run -d \
 --name opendaylight \
 -p 8181:8181 -p 6653:6653 \
 opendaylight/odl:latest
```
Controller UI: http://localhost:8181
Default Credentials: admin / admin

**5. Deploy Keycloak (Identity & Trust Anchor)**
**Goal:** Identity-based Zero Trust source of truth.

```bash
docker run -d \
 --name keycloak \
 -p 8080:8080 \
 -e KEYCLOAK_ADMIN=admin \
 -e KEYCLOAK_ADMIN_PASSWORD=admin \
 quay.io/keycloak/keycloak:latest start-dev
```
  * Configuration (Manual Setup Required in UI):
    -Realm: SDP_ZeroTrust
    -Clients: ODL-Client-A, ODL-Client-B
    -Roles: trusted-device, untrusted-device, service-a, service-b

**6. Deploy Open Policy Agent (OPA)**

**Goal:** Declarative, auditable trust logic.

```bash
docker run -d \
 --name opa \
 -p 8182:8182 \
 openpolicyagent/opa run --server
```
  *OPA Policy Configuration*
The  Rego scripts implement logic. Testcase2 logic in table below:

	| User        | Device    | App     | Result      |
	| ----------- | --------- | ------- | ----------- |
	| Admin       | Trusted   | Allowed | **Full**    |
	| Admin       | Untrusted | Allowed | **Limited** |
	| Guest       | Trusted   | Allowed | **Limited** |
	| Guest       | Untrusted | Allowed | **None**    |
	| Any         | Any       | Blocked | **None**    |
	| Blacklisted | Any       | Any     | **None**    |

**7. Deploy Envoy Proxy (Application PEP): Why: L7 Zero Trust enforcement.**

```bash 
        docker run -d \
				 --name envoy \
				 -p 10000:10000 \
				 envoyproxy/envoy:v1.30-latest
 ```
  Envoy integrates:
		-OAuth2 (Keycloak)
    -Policy checks (OPA)

**8. Build the Mininet Zero Trust Topology: Minimal Topology.**
      Clients: *Trusted* and *Untrusted*
		  Service-A *Allowed*
	  	Service-B *Blocked*
  OVS switch (controlled by OpenDaylight)
```bash
	sudo mn \
	 --topo single,3 \
	 --switch ovs \
	 --controller remote,ip=127.0.0.1,port=6653
```
**10. Baseline Validation.** ***Before and After policies:***

  ***Before policies:***
  ```code
      pingall *-- successful access to all nodes and services*
      curl service-a *-- reachable (returns success)*
      curl service-b *-- reachable (returns success)*
```
  ***After policies:***
```code
  Service-A → *allowed*
  
  Service-B → *denied*
```
***This establishes basic Zero Trust enforcement based on a single criteria such as:***
         
         **User:** *[Admin or Blacklisted]*
         **Device:** *[Trusted or Untrusted]*
         **Application/Service:** *[Allowed or Blocked]*
         **Network:** *[Local or Remote]*

**11. Experiment Validation: Custom Multi Domain policy**

This establishes multidomain Zero Trust enforcement such as:
This policy design operationalizes Zero Trust not as a concept, but as an enforceable control plane.
Each pillar is represented as a measurable attribute rather than a static assumption.

_Notably:_
    _ -Identity and device trust are decoupled
    - Application risk influences authorization dynamically  
    - Network location is intentionally excluded_

_This aligns with Zero Trust’s principle of “never trust, always verify”, even for internal traffic._

**12. Experiment Validation:Static Multidomain Zero Trust enforcement**

  This establishes Static Multidomain Zero Trust enforcement
    Numerical values for policies

**13. Experiment Validation: Custom Contextual trust Algorithm for Zero Trust enforcement**

**Goal: ** Custom Contextual trust Algorithm

This establishes dynamic Multidomain Zero Trust enforcement


**14. Experiment Validation: Custom COntextual Trust Algorithm with Temporal Decay for Zero Trust enforcement**

  **Goal: ** Custom COntextual Trust Algorithm with Temporal Decay

This establishes Dynamic, Contextual and Temporal Multidomain Zero Trust enforcement



##END  

