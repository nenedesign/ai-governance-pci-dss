# PCI-DSS Scope Boundary System Prompt Pattern

**Framework:** PCI-DSS v4.0  
**Requirements addressed:** Requirement 3 (protect stored account data), Requirement 4 (protect cardholder data in transit), Requirement 6.4 (protect public-facing web applications against attacks)  
**Artifact type:** Defensive system prompt pattern  
**Pairs with:** [Cardholder Data Detector workflow](workflow.json). Apply both; this prompt is a last line of defense, not a substitute for pre-inference scanning.

---

## The Risk

Every LLM interface is a potential cardholder data entry point. Users who interact with AI assistants in financial services contexts, billing support, account management, or e-commerce frequently paste card numbers into chat fields to ask about charges, disputes, or payments. If that input reaches the model unfiltered, the cardholder data enters the LLM context window: it is likely logged by the application, may appear in the model's response, and could be retained in session memory or a conversation history store.

PCI-DSS v4.0 Requirement 3 prohibits storing sensitive authentication data after authorization, including full magnetic stripe data, CVV/CVC security codes, and PINs. Requirement 4 requires strong cryptography for cardholder data in transit. Neither requirement contemplates an LLM context window as a compliant storage or transit mechanism. By default, it is not one.

Two failure paths are common:

**[Scenario] Billing support scenario:** A user contacts an AI-powered customer service assistant about an unrecognized charge and pastes their full card number, expiry, and CVV into the chat field. The model confirms it received the information and begins diagnosing the issue. The card data now appears in the application request log, the model context, and potentially the response if the model echoes it for confirmation.

**[Technique] Prompt injection into PCI context:** An attacker crafts a message that causes the model to output cardholder data it received earlier in the session. If card data entered the context (from the user or from a compromised RAG source), a successful injection can exfiltrate it without any network-layer control firing. Scope boundary instructions reduce this surface by blocking card data from entering the model context in the first place.

---

## Defensive System Prompt Pattern

```
PCI-DSS SCOPE BOUNDARY

This assistant operates outside PCI-DSS scope. It is not authorized to receive,
process, store, or transmit cardholder data. Cardholder data includes:
- Primary Account Numbers (PANs): full or partial card numbers
- Card Verification Values: CVV, CVC, CID, and equivalent security codes
- Cardholder names combined with account identifiers
- Card expiry dates combined with any of the above
- Full magnetic stripe data or chip data equivalents

WHAT THIS ASSISTANT CANNOT DO
Do not accept, request, repeat, confirm, or act on cardholder data provided in any message.
If a user shares card data, whether intentionally or by pasting it into the chat, do not echo,
paraphrase, or process it. Do not ask the user to clarify or resubmit it.

RESPONSE WHEN CARD DATA IS DETECTED
Immediately acknowledge that you cannot accept this information through this channel
and redirect the user to the secure, PCI-compliant path for their specific need:
  - Transaction disputes: [SECURE_DISPUTE_CHANNEL]
  - Billing questions: [SECURE_BILLING_CHANNEL]
  - Fraud reports: [SECURE_FRAUD_CHANNEL]

Do not delay the redirect. Do not first attempt to answer the underlying question
using the card data before redirecting. The redirect is the answer.

Example response:
"For your security, I'm not able to accept card details through this chat. To dispute 
a charge, please use [SECURE_DISPUTE_CHANNEL] where your information is handled through 
our PCI-compliant process. I'm happy to help with questions that don't involve sharing 
card details."

WHAT THIS ASSISTANT CAN DO
You can discuss cardholder data topics without receiving actual cardholder data:
- Explain what a CVV is and where to find it, without asking the user to share it
- Explain the dispute process without processing the disputed transaction details here
- Confirm that a charge exists in a system of record, without repeating card identifiers
- Instruct the user on how to use a secure channel, without needing their card number to do so

DO NOT GENERATE CARD-LIKE PATTERNS
Do not produce strings that resemble card numbers, even as examples, test values, or
placeholders. Use clearly non-card identifiers such as "XXXX-XXXX-XXXX-XXXX" or
"[card number]" instead.

MASKED INPUT HANDLING
If you receive input where card data has already been masked (e.g., [CARD-REDACTED],
[CVV-REDACTED], [EXPIRY-REDACTED]), treat the masked placeholders as opaque tokens.
Do not attempt to interpret, recover, or reason about the original values. Process the
message as if the masked fields were blank.
```

---

## What Each Section Defends Against

| Section | Risk it mitigates |
|---------|------------------|
| PCI-DSS Scope Boundary definition | Model treating cardholder data as processable input |
| What This Assistant Cannot Do | Echoing, paraphrasing, or acting on card data in context |
| Response When Card Data Is Detected | User card data processed before redirect fires |
| What This Assistant Can Do | Overcorrection: model refusing to discuss card-related topics at all |
| Do Not Generate Card-Like Patterns | Model producing synthetic PANs that could be mistaken for real data |
| Masked Input Handling | Model attempting to infer original values from masked tokens |

---

## Usage Notes

**Pair with the cardholder data detector workflow.** This prompt addresses what happens if card data reaches the model. The [Cardholder Data Detector workflow](workflow.json) prevents it from reaching the model at all by scanning and masking input at the webhook layer. Both controls are needed. A prompt instruction can be bypassed via injection (OWASP LLM01:2025); a pre-inference workflow-level scan cannot.

**Replace all placeholder channels before deployment.** The prompt contains `[SECURE_DISPUTE_CHANNEL]`, `[SECURE_BILLING_CHANNEL]`, and `[SECURE_FRAUD_CHANNEL]`. These must be replaced with real, PCI-compliant URLs or channel names specific to your organization. Do not leave placeholders in a production system prompt: a model will echo them back to users, which is confusing and could be interpreted as a phishing attempt.

**Partial PANs are still in scope.** PCI-DSS considers a PAN truncated to the last four digits to be out of scope for most requirements, but the first six digits (BIN) combined with the last four may re-identify the card. Do not assume partial card numbers are safe to process. The safest instruction is to decline all card number inputs, partial or full.

**This prompt does not satisfy PCI-DSS compliance.** PCI-DSS v4.0 compliance requires a formal assessment by a Qualified Security Assessor (QSA). A system prompt is a behavioral instruction, not a technical control; it can be overridden by injection attacks or model failure modes. This pattern reduces cardholder data exposure; it does not constitute a compensating control or substitute for the technical and procedural requirements of PCI-DSS.

**Session memory and context window persistence.** If your application uses multi-turn conversation history, card data that enters the context in one turn may persist across subsequent turns even if the model does not reproduce it. Architecture-level controls, specifically, stripping cardholder data from conversation history before re-injecting it into the next prompt, are required to address this. This system prompt cannot prevent persistence of data that entered the context in a prior turn.

**Logging and data residency.** Most LLM API providers log requests and responses for safety, quality, and operational purposes. If cardholder data enters the API request, it may be retained by the provider under their data retention policy, which is separate from your PCI-DSS compliance scope. Use a provider with a documented Zero Data Retention policy for PCI-scoped applications, or ensure the pre-inference scan runs before the API call, not inside it.

---

## Real-World Reference

- **[Scenario] AI-powered billing support:** An enterprise deploys a conversational AI assistant for billing inquiries. A user pastes their 16-digit card number into the chat to ask about a charge. The model, lacking scope boundary instructions, confirms the card number in its reply to show it understood the query. The card number now appears in the application's conversation log, the model API request log, and the support ticket created from the conversation. None of these systems are PCI-DSS compliant. This scenario illustrates why scope boundary instructions, combined with pre-inference scanning, are required before deploying any conversational AI in a billing or payment context.

- **[Confirmed] Recall and PII persistence (2024):** Microsoft's Copilot+ Recall feature, which continuously screenshots and indexes user activity using an on-device LLM, was found to capture and store banking app screens, payment confirmations, and sensitive financial information in an unencrypted local database. Reported widely in May 2024; Microsoft delayed the rollout following security researcher disclosures (Kevin Beaumont, among others). The incident illustrates that AI systems with broad input capture, absent explicit scope boundaries, will ingest cardholder data whether or not that was intended.

- **[Technique] Prompt injection exfiltration from context window:** Attackers can craft input that causes an LLM to output data from earlier in its context window. If cardholder data entered the context (from user input or a compromised document), an injection payload can instruct the model to repeat it in a format that evades output filters. Scope boundary instructions reduce this by blocking card data from entering the context; they do not eliminate the risk if card data is already present. Source: OWASP LLM01:2025 Prompt Injection.

---

## Attribution

Built by [Neville Ko](https://www.linkedin.com/in/nevilleko/) · [GitHub](https://github.com/nenedesign).

---

## Disclaimer

This document is provided for educational and informational purposes only. The prompt patterns and guidance presented here are general-purpose starting points and do not constitute legal, compliance, or security advice. They have not been certified or validated against PCI-DSS v4.0 or any other regulatory framework by a Qualified Security Assessor.

Organizations deploying LLMs in payment card environments must engage a QSA for formal compliance assessment. This pattern reduces cardholder data exposure through behavioral instruction; it does not satisfy PCI-DSS technical control requirements and must not be represented as doing so.
