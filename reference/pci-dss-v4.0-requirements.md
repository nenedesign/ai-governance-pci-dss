# PCI-DSS v4.0: Requirements Reference

**Standard:** Payment Card Industry Data Security Standard (PCI-DSS) v4.0  
**Published:** March 2022  
**Published by:** PCI Security Standards Council (PCI SSC)  
**Official source:** https://www.pcisecuritystandards.org/document_library/

---

## Why this file exists

The OWASP LLM Top 10 source document in [ai-governance-owasp10](https://github.com/nenedesign/ai-governance-owasp10) is pinned locally because OWASP publishes under CC BY 4.0, which permits redistribution and adaptation with attribution.

PCI-DSS v4.0 is copyright PCI Security Standards Council, LLC. It is not freely redistributable. You must register at pcisecuritystandards.org to download it. For that reason this file does not reproduce the standard text — it describes the specific requirements this repository's artifacts are built against, in original language, and links to the authoritative source.

---

## Requirements in scope

The artifacts in this repository implement controls against three PCI-DSS v4.0 requirements, each targeting a different point in the cardholder data lifecycle at the LLM inference layer.

---

### Requirement 3: Protect Stored Account Data

**What it requires:** Organizations must minimize cardholder data storage, retain it only as long as necessary, and protect stored account data through access controls, encryption, and masking. Sensitive authentication data (SAD) — including CVV/CVC/CID security codes and full magnetic stripe data — must not be stored after authorization under any circumstances, regardless of encryption.

**Why it applies to LLM deployments:** An LLM context window is a temporary storage mechanism. When a user submits card data to an LLM interface, that data enters the context window, may be echoed in the response, and is typically captured in application logs. None of these constitute compliant storage under Requirement 3. The standard was written before LLMs existed; the inference layer was not anticipated as a data entry point.

**Sub-requirements relevant to this repo:**
- **3.3.1** — Sensitive authentication data is not retained after authorization. CVV/CVC/CID must be purged. A workflow that allows SAD to reach an LLM violates this requirement regardless of what happens after.
- **3.4.1** — Primary account numbers (PANs) are masked when displayed. The Cardholder Data Detector implements this by replacing PANs with `[CARD-REDACTED]` before the masked prompt is forwarded to the model.

**Artifact:** [Cardholder Data Detector](../workflow.json) — scans and masks PANs, CVV/CVC/CID, and expiry dates before any data reaches the LLM API.

---

### Requirement 4: Protect Cardholder Data with Strong Cryptography During Transmission

**What it requires:** Primary account numbers must be protected with strong cryptography during transmission over open, public networks. Unprotected PANs must never be sent via end-user messaging (chat, email, IM). Policies must exist to ensure only trusted keys and certificates are used.

**Why it applies to LLM deployments:** When a user submits a PAN through an LLM interface, that PAN transits the network between the user's browser and the application server, and again from the application server to the LLM API endpoint. The second leg — the call to the LLM API — may carry the PAN in the request body. Masking the PAN before that API call eliminates transmission of cardholder data to the LLM provider entirely.

**Sub-requirement relevant to this repo:**
- **4.2.1** — Strong cryptography is used to safeguard PANs during transmission. Removing the PAN from the request before transmission is a stronger control than encrypting a PAN in transit: the LLM provider never receives it.

**Artifact:** [Cardholder Data Detector](../workflow.json) — by masking before the HTTP Request node, cardholder data is never transmitted to the LLM API.

---

### Requirement 6.4: Protect Public-Facing Web Applications Against Attacks

**What it requires:** Public-facing web applications must be protected against known attack methods through technical controls. This includes automated solutions that detect and prevent web-based attacks, and periodic review or automated scanning for vulnerabilities.

**Why it applies to LLM deployments:** An LLM webhook endpoint is a public-facing web application. Without controls at the application layer, it accepts arbitrary user input — including cardholder data — and passes it directly to the model. The pre-inference scanning pattern implements a protective technical control at the application layer, before input reaches the model.

**Sub-requirement relevant to this repo:**
- **6.4** — The Cardholder Data Detector operates as an application-layer control on the webhook endpoint. It intercepts, scans, and sanitizes all inbound user input before it reaches the LLM API, implementing the automated detection and mitigation the requirement expects.

**Artifact:** [Cardholder Data Detector](../workflow.json) — webhook-layer scanning applies Requirement 6.4 controls at the entry point of the LLM interface.

---

## Artifact-to-requirement mapping

| Artifact | Req 3 | Req 4 | Req 6.4 |
|----------|-------|-------|---------|
| [Cardholder Data Detector](../workflow.json) | Prevents PANs and SAD from entering the context window or being logged | Prevents PANs from being transmitted to the LLM API | Implements automated detection and masking at the public-facing webhook layer |
| [PCI Scope Boundary System Prompt](../pci-scope-boundary-prompt.md) | Instructs the model not to echo, paraphrase, or store card data | Prohibits the model from requesting or acting on transmitted card data | Last-line behavioral control at inference time |

---

## Compliance note

Neither artifact in this repository constitutes PCI-DSS compliance. Formal compliance requires a formal assessment by a Qualified Security Assessor (QSA). These artifacts implement specific technical controls that address cardholder data exposure at the LLM inference layer — a gap not covered by legacy PCI-DSS guidance written before LLM deployments existed. They demonstrate the control patterns a QSA would expect to see in a compliant LLM deployment.

---

## Reference

PCI Security Standards Council. *Payment Card Industry Data Security Standard: Requirements and Testing Procedures, Version 4.0.* March 2022. Available at: https://www.pcisecuritystandards.org/document_library/
