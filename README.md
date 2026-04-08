# ⚛️ Azure Quantum Activation — Colab Execution Bridge
## 📊 Architecture Diagram
<img width="2816" height="1536" alt="Gemini_Generated_Image_sekxtpsekxtpsekx" src="https://github.com/user-attachments/assets/aa7abc8a-017c-4b0a-8817-89b0dde63e44" />

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/billyrdavis1985-bot/Azure-Quantum-Activation/blob/main/Azure_Activation_Test_Flight_Configuration.ipynb)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform](https://img.shields.io/badge/Platform-Azure%20Quantum-blue)](https://azure.microsoft.com/en-us/products/quantum)
[![Quantinuum](https://img.shields.io/badge/Backend-Quantinuum%20H2-purple)](https://www.quantinuum.com)
[![Rigetti](https://img.shields.io/badge/Backend-Rigetti%20QVM-orange)](https://www.rigetti.com)
[![IRMB](https://img.shields.io/badge/Research-IRMB%20Phase%207G-green)](https://github.com/billyrdavis1985-bot/Quantum-Bridge-AWS)
[![Cost](https://img.shields.io/badge/Testflight%20Cost-%240.00-brightgreen)]()

# ⚛️ Azure Quantum Activation — Colab Execution Bridge

A verified step-by-step protocol for connecting Google Colab 
to Azure Quantum and running quantum circuits on Quantinuum 
and Rigetti backends.

---

## 🎯 Purpose

Most Azure Quantum tutorials assume a local environment with 
pre-configured credentials. This notebook solves the harder 
problem: authenticated quantum backend access straight from 
Google Colab, with zero local setup.

---

## 🛠️ Prerequisites

| Requirement | Details |
|---|---|
| Azure Account | Free account with $200 credit |
| Azure Quantum Workspace | Created via Azure Portal |
| Quantinuum Provider | Basic plan (free emulator access) |
| Rigetti Provider | Basic plan (free QVM access) |
| Google Colab | Free tier sufficient |

---

## 🚀 Workspace Setup

### 1. Create Azure Quantum Workspace
- Go to `portal.azure.com`
- Search "Azure Quantum" → Create
- Choose **Advanced create**
- Fill in:
  - Resource group: your choice
  - Workspace name: your choice
  - Region: `East US`
- Add providers:
  - **Quantinuum → Basic** (free emulator)
  - **Rigetti → Basic** (free QVM)
- Click Review + Create

> ⚠️ Do NOT select IonQ Pay As You Go — 
> not covered by Azure credit

### 2. Get Your Credentials
From your workspace overview copy:
- Subscription ID
- Resource Group name
- Workspace name
- Tenant ID (from Microsoft Entra ID)

---

## 🔑 Authentication — The Critical Pattern

Azure Quantum uses **Device Code Flow** in Colab — 
no API keys needed. Standard browser login fails 
in Colab environments.

```python
from azure.quantum import Workspace
from azure.identity import DeviceCodeCredential

credential = DeviceCodeCredential(
    tenant_id="YOUR-TENANT-ID"
)

workspace = Workspace(
    subscription_id="YOUR-SUBSCRIPTION-ID",
    resource_group="YOUR-RESOURCE-GROUP",
    name="YOUR-WORKSPACE-NAME",
    location="eastus",
    credential=credential
)
```

When you run this, navigate to:
`https://login.microsoft.com/device`

Enter the code printed in the output and sign 
into your Microsoft account.

> ⚠️ Use `login.microsoft.com/device` NOT 
> `microsoft.com/devicelogin` — the second URL 
> may fail depending on your account configuration.

---

## 🔬 Available Backends

```python
backends = workspace.get_targets()
for target in backends:
    print(f"{target.name} | {target.provider_id}")
```

Expected output:

quantinuum.sim.h2-1sc  | quantinuum
quantinuum.sim.h2-1e   | quantinuum
rigetti.sim.qvm        | rigetti

| Backend | Type | Cost |
|---|---|---|
| quantinuum.sim.h2-1sc | Syntax checker | Free |
| quantinuum.sim.h2-1e | H2 Emulator 32-qubit | Free eHQCs |
| rigetti.sim.qvm | Rigetti QVM | Free |

---

## 🧪 The Golden Circuit

Canonical 3-qubit GHZ benchmark used across all backends:

```python
from azure.quantum.qiskit import AzureQuantumProvider
from qiskit import QuantumCircuit

provider = AzureQuantumProvider(workspace)
backend = provider.get_backend("quantinuum.sim.h2-1e")

golden_circuit = QuantumCircuit(3, 3)
golden_circuit.h(0)
golden_circuit.cx(0, 1)
golden_circuit.cx(1, 2)
golden_circuit.measure([0, 1, 2], [0, 1, 2])

job = backend.run(golden_circuit, shots=100)
print(f"Job ID: {job.id()}")
```

Expected output: `{'000': ~50, '111': ~50}`

---

## 🛰️ Job Recovery

Azure jobs are asynchronous. Recover any result 
by Job ID without keeping session alive:

```python
def recover_azure_job(job_id, workspace):
    provider = AzureQuantumProvider(workspace)
    job = provider.get_job(job_id)
    if job.status().name == "DONE":
        return job.result().get_counts()
    return f"Status: {job.status()}"
```

---

## ⚠️ Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `InteractiveBrowserCredential failed` | Colab can't open browser | Use `DeviceCodeCredential` |
| `Failed to open a browser` | Same — Colab environment | Use `DeviceCodeCredential` |
| `ClientAuthenticationError` | Wrong tenant or no credential | Add explicit `tenant_id` |
| `DeploymentFailed` on workspace | Region conflict or storage issue | Try East US 2, reuse existing storage account |
| Jobs stuck in QUEUED | Emulator queue backlog | Wait 10-15 min or recover next session |

---

## 📊 Testflight Results — April 7, 2026

Full validation across all backends:

### Golden Circuit
| Backend | Counts | Fidelity |
|---|---|---|
| quantinuum.sim.h2-1e | 000:53, 111:47 | 100% |
| rigetti.sim.qvm | 000:45, 111:55 | 100% |

### Noise Injection (GHZ under identity gate noise)
| Noise Level | Fidelity |
|---|---|
| 0 | 100% |
| 5 | 100% |
| 10 | 100% |
| 20 | 99% |

### 5-Qubit GHZ State
{'00000': 50, '11111': 49, '11011': 1}
Fidelity: 99%

### Cross-Provider Comparison
| Provider | Fidelity |
|---|---|
| Quantinuum H2 | 99% |
| Rigetti QVM | 100% |
| Delta | 1% |

### CHSH Bell Inequality
CHSH Value: 0.340
Classical limit: 2.000
Result: No violation
> **Finding:** Noiseless emulators suppress the 
> decoherence asymmetry required for Bell violation. 
> CHSH testing requires real QPU hardware. 
> This null result is consistent across AWS Braket 
> (IonQ Forte-1) and Azure Quantum emulators.

### Cross-Platform Fidelity History
| Platform | Backend | Fidelity |
|---|---|---|
| AWS Braket | IonQ Forte-1 (QPU) | ~99% |
| AWS Braket | Rigetti Ankaa-3 (QPU) | ~91% |
| Azure Quantum | Quantinuum H2 (Emulator) | 99% |
| Azure Quantum | Rigetti QVM (Simulator) | 100% |

---

## 💰 Cost Management

Set a budget alert before running anything:

`Azure Portal → Cost Management → Budgets → Add`

- Amount: $150
- Alert at 80% and 100%
- Email notification

> Azure Quantum workspaces do not charge 
> just for existing — only when jobs are submitted. 
> Emulator and simulator backends are free.

---

## 📁 Repository Structure
├── Azure_Activation_Test_Flight_Configuration.ipynb
├── azure_testflight_results.png
├── azure_testflight_summary.json
└── README.md

---

## 🔗 Context

This repository is part of the **IRMB (Infinite Resilience 
Matrix Bridge)** research program — Phase 7G: Bell 
Coordination. The Azure Quantum access layer provides 
quantum measurement data to a multi-LLM orchestration 
system studying quantum-AI coordination.

- AWS Braket bridge: github.com/billyrdavis1985-bot/Quantum-Bridge-AWS
- Phase 7G Design 2: QEC complete — Shor code F=1.000
- Phase 7G Design 3: Bell/CHSH coordination — complete
- Phase 7G Design 4: Azure Quantum multi-provider — active

---

## 📄 License

MIT — see `LICENSE` for details.

---

*Built under the IRMB motto: **Full Force Eternal** · Romans 8:28*
