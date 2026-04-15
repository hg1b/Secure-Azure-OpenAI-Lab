# Secure-Azure-OpenAI-Lab

# Azure OpenAI Secure Lab — Zero-Trust Networking

This project replicates how an enterprise would securely deploy an AI service in Azure using zero-trust networking principles — no implicit trust, every connection must be explicitly allowed, and the attack surface is minimised by removing public access entirely.


---
<img width="1152" height="642" alt="Azure AI Model drawio (1)" src="https://github.com/user-attachments/assets/ace33ac5-9038-463a-bc88-ec937fba30c8" />


## Architecture Overview

### Region (Australia East)
Physical location of the Azure data centre where all resources are deployed.

### Subscription
Billing container — every resource created gets charged to this.

### Resource Group (`rg-openai-secure-lab`)
Logical container where all resources for the project are stored and managed together.

### Virtual Network (`vnet-openai-secure-lab` / `10.0.0.0/16`)
- Private network inside Azure — your own isolated LAN
- Nothing and no one can reach anything inside unless explicitly allowed
- Resources communicate over Microsoft's backbone without touching the public internet
- Without a VNet, every resource would be exposed publicly by default

### VM Subnet (`vm-subnet` / `10.0.0.0/24`)
- Subnet within the VNet for the test VM
- Separated so we can apply specific security rules to the VM without affecting other resources

### Private Subnet (`private-subnet` / `10.0.1.0/24`)
- Subnet dedicated to the private endpoint
- Separate from the VM subnet because the private endpoint has different security requirements — it needs to accept inbound connections from the VM but nothing else
- Having its own subnet means we can attach NSG rules specific to protecting the private endpoint without affecting the VM's rules

### Network Security Group (`nsg-openai-secure-lab`)
- Acts as a firewall at the subnet level
- Has a list of allow/deny rules that control what traffic can flow in and out
- Even inside a private network, we still restrict traffic to only what's necessary
- By default every inbound/outbound connection is denied. Rules configured:
  1. Allow SSH (port 22) from a trusted IP only
  2. Allow HTTPS (port 443) from `vm-subnet` to `private-subnet` (port 443 because that's the port Azure OpenAI's API runs on)
- Without an NSG, anything inside the VNet could talk to anything on any port — not least-privilege and not zero-trust

### Virtual Machine (`vm1`)
- Test workstation inside the VNet running Ubuntu Server
- Has two IPs:
  - A private IP (`10.0.0.4`) for communicating within the VNet
  - A temporary public IP for SSH access from a trusted external IP
- The VM is the only thing that can reach Azure OpenAI
- Once inside the VM, we can verify the whole architecture works by testing connectivity to the OpenAI service:

```bash
curl "https://<your-openai-resource>.openai.azure.com/openai/deployments/ada-embedding/embeddings?api-version=2023-05-15" \
  -H "Content-Type: application/json" \
  -H "api-key: <your-api-key>" \
  -d '{"input":"Hello world"}'
```

- `curl` — command line tool to make HTTPS requests
- `-H "Content-Type: application/json"` — tells the server the payload is JSON
- `-H "api-key: ..."` — sends your API key for authentication
- `-d '{"input":"Hello world"}'` — the request body, sending text for the embedding model to convert into a vector

### Private Endpoint (`pe-openai` / `10.0.1.4`)
- Acts as a doorway from the VNet to a specific Azure service
- When the PE is created, a network interface (NIC) is automatically provisioned with a private IP inside the VNet
- Sits in the private subnet and forwards traffic to Azure OpenAI through Private Link (Microsoft's internal backbone)
- Normally, Azure PaaS services like OpenAI are publicly accessible — in this lab, public access is disabled in Azure OpenAI's config
- When creating the private endpoint, the Azure OpenAI service is selected as the target resource

### Private DNS Zone (`privatelink.openai.azure.com`)
- When a device wants to reach Azure OpenAI, it calls the domain (e.g. `<your-openai-resource>.openai.azure.com`)
- If the resource were public, that domain resolves to a public IP — but public access is disabled, so that would fail
- The Private DNS Zone solves this. When configuring the private endpoint, private DNS zone integration is enabled
- This intercepts the DNS lookup and resolves the domain to the private endpoint's IP (`10.0.1.4`) instead of the public IP
- When the VM makes a request, it still queries Azure's DNS server (`168.63.129.16`). Azure DNS checks if there's a private DNS zone linked to the VNet — if yes, it returns the private endpoint IP

### Azure OpenAI Service (`text-embedding-ada-002` / public access disabled)
- The actual AI service running the model
- Running `text-embedding-ada-002`, which converts text into numerical vectors. Use cases include:
  - **Semantic search** — convert documents into vectors, convert a search query into a vector, find which documents are closest in meaning
  - **Alert classification** — convert security alerts into vectors, compare against known false-positive patterns, flag which ones are worth investigating
- By default Azure OpenAI has a public endpoint anyone on the internet can attempt to hit
- Public access is disabled so the only way to reach it is through the private endpoint inside the VNet

---

## Traffic Flow

This is the path a request takes from the VM to Azure OpenAI:

**1. DNS Resolution**
The VM calls `<your-openai-resource>.openai.azure.com`. Before any traffic goes anywhere, it needs to resolve the domain to an IP. The Private DNS Zone intercepts the lookup and returns the private endpoint IP (`10.0.1.4`).

**2. NSG Evaluation**
The VM now knows to send traffic to `10.0.1.4` on port 443. The packet leaves the VM in `vm-subnet` headed for `private-subnet`. The NSG evaluates the traffic against its rules — source, destination, and port must all match. Since there's a rule allowing HTTPS from `vm-subnet` to `private-subnet`, the traffic passes through.

**3. Private Endpoint**
Traffic arrives at the private endpoint's NIC in the private subnet. The PE receives it and forwards it through Azure Private Link directly to Azure OpenAI.

**4. Azure OpenAI Service**
The request arrives at the OpenAI resource, the service processes it, generates the response (embedding vectors), and sends it back along the same path in reverse.

---

## Results

<img width="1703" height="953" alt="Successful connection to model secure" src="https://github.com/user-attachments/assets/3fb5275e-3820-4215-aeda-001972c2c133" />
<img width="1858" height="568" alt="Access Denied secure" src="https://github.com/user-attachments/assets/01e1a70e-9c97-4d6f-a002-d8c65de5c0e6" />

---

## Potential Improvements
- Replace the temporary public IP on the VM with Azure Bastion for SSH access, removing the public attack surface entirely
- Add diagnostic logging and send NSG flow logs to a Log Analytics workspace for monitoring
- Implement Azure Key Vault to store the API key instead of passing it inline
- Add a second VM outside the VNet to formally demonstrate that access is denied from external sources
