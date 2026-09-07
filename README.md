# AI Governance: PCI-DSS v4.0
### Cardholder Data Protection for LLM Deployments

Practical guardrails for deploying AI in payment card environments. Each artifact implements a specific PCI-DSS v4.0 requirement at the inference layer, where cardholder data most commonly enters an LLM system undetected.

Built for teams deploying AI in financial services, e-commerce, and any context where users interact with an AI assistant that could receive payment card data.

---

## The Problem

Every LLM interface is a potential cardholder data entry point. A user asks about a disputed charge and pastes their card number into the chat. A support agent uses an AI copilot and includes card details in their query. The model receives the data, logs it, and may echo it back.

PCI-DSS v4.0 Requirement 3 prohibits storing sensitive authentication data after authorization. An LLM context window is not a compliant storage mechanism. By default, neither is the application log that captures it.

---

## Artifacts

| Artifact | Type | Requirement | Status |
|----------|------|-------------|--------|
| [Cardholder Data Detector](workflow.json) | n8n workflow | Req 3, 4, 6.4 | Done |
| [PCI Scope Boundary System Prompt](pci-scope-boundary-prompt.md) | System prompt | Req 3, 4, 6.4 | Done |

---

## Cardholder Data Detector

**File:** [workflow.json](workflow.json)

A pre-inference n8n workflow that scans and masks cardholder data in the user prompt before it reaches the LLM. Implements a layered detection approach covering the four major card networks.

**Workflow:** Webhook → Normalize Input → Scan and Mask → Forward to LLM API → Respond to Webhook

**Detects and masks:**
- Primary Account Numbers (PANs): Visa, Mastercard, Amex, Discover — formatted and unformatted
- CVV/CVC/CID security codes — detected by keyword context, not pattern alone
- Card expiry dates — MM/YY and MM/YYYY formats

**Response includes:**
- `X-PCI-Scan: masked | clean` — whether redaction occurred
- `X-PCI-Redaction-Count: N` — how many fields were masked
- `pci_scan` object in the response body for downstream audit logging

**How to import:**
1. Download [workflow.json](workflow.json)
2. Open your n8n instance
3. Click **+** (New Workflow) → **Import from file** → select the file
4. Replace `<__PLACEHOLDER_VALUE__your-llm-api-endpoint__>` with your LLM API URL
5. Replace `<__PLACEHOLDER_VALUE__your-model-name__>` with your model name
6. Add your LLM Bearer token to the **LLM API Key** credential

**Customization:**
- The Code node regex patterns cover the four major networks. Add patterns for additional card types or regional networks in the same node.
- The `responseBody` expression assumes OpenAI-compatible response format (`choices[0].message.content`). Update the path if your LLM returns a different structure.
- Wire the `pci_scan` object into a Supabase or logging node for a persistent audit trail.

---

## PCI Scope Boundary System Prompt

**File:** [pci-scope-boundary-prompt.md](pci-scope-boundary-prompt.md)

A defensive system prompt pattern that instructs the LLM on its PCI-DSS scope boundaries: what cardholder data is, what the model cannot do with it, and how to redirect users to compliant channels when card data is shared.

**Covers:**
- Scope definition: what constitutes cardholder data for the model's purposes
- Hard prohibitions: no echoing, paraphrasing, requesting, or acting on card data
- Redirect instructions: per use-case channel routing (disputes, billing, fraud)
- Positive permissions: what the model can discuss without receiving actual card data
- Masked input handling: how to treat `[CARD-REDACTED]` tokens from the detector workflow

**Usage:** Apply this prompt as your system message. Replace `[SECURE_DISPUTE_CHANNEL]`, `[SECURE_BILLING_CHANNEL]`, and `[SECURE_FRAUD_CHANNEL]` with your organization's PCI-compliant channel URLs before deployment.

**Important:** This prompt is a last line of defense, not a primary control. A system prompt can be overridden by prompt injection. Pair it with the Cardholder Data Detector workflow, which blocks card data at the webhook layer before it reaches the model.

---

## Methodology

Both artifacts implement PCI-DSS v4.0 controls at the inference layer: the point where cardholder data most commonly enters an LLM deployment. The detector operates before the LLM call (pre-inference); the scope boundary prompt operates inside the model at inference time. Together they cover the workflow layer and the behavioral layer.

Neither artifact constitutes PCI-DSS compliance. Formal compliance requires assessment by a Qualified Security Assessor (QSA). These artifacts reduce cardholder data exposure in LLM deployments and demonstrate the control patterns a QSA would expect to see.

**Reference labeling:** Real-world references in this repository are labeled to distinguish their verification level. **[Confirmed]**: a documented public incident with a verifiable outcome. **[Technique]**: an attack method described in security research. **[Scenario]**: a constructed example based on known failure modes in the domain.

---

## Related

- [ai-governance-owasp10](https://github.com/nenedesign/ai-governance-owasp10): OWASP LLM Top 10 v2.0 implementations — security controls for LLM-specific risks

---

## License

MIT License. See [LICENSE](LICENSE).

---

## About

Built by [Neville Ko](https://www.linkedin.com/in/nevilleko/) · [GitHub](https://github.com/nenedesign), AI Product Manager and Builder at [Distinct AI](https://www.fromus.ca/ai-builds).
