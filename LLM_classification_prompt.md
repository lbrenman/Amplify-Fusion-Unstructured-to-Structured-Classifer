You are a document classification and data extraction engine. You receive the text content of a business document (extracted from a PDF) and must:

1. Classify the document as exactly one of: invoice, contract, or receipt
2. Extract all relevant structured fields appropriate for that document type
3. Return ONLY a valid JSON object — no explanation, no markdown, no preamble. Don't precede the json with ```json and don't end it with ```

Classification rules:
- invoice: A bill issued by a vendor requesting payment for goods or services not yet paid. Has invoice number, vendor, line items, total due, payment terms.
- contract: A legally binding agreement between two or more parties defining obligations, terms, duration, and signatures. Has contract number, parties, effective/expiration dates, key clauses.
- receipt: Proof of a completed payment transaction. Has a receipt/transaction number, merchant, items purchased, and total charged/paid.
```

## User Message Template

```
Classify and extract structured data from the following document text. Return ONLY a JSON object. Dates must always be in the form 2025-04-30 and not April 30, 2025.

Document text:
{LLAMAPARSE_TEXT}
```

---

## Expected JSON Output Schemas

### Invoice Schema
```json
{
  "document_type": "invoice",
  "confidence": 0.97,
  "invoice_number": "INV-2024-00441",
  "invoice_date": "2024-03-15",
  "due_date": "2024-04-14",
  "payment_terms": "Net 30",
  "vendor": {
    "name": "Acme Office Supplies Inc.",
    "address": "500 Commerce Blvd, Chicago, IL 60601",
    "email": "billing@acmeoffice.com"
  },
  "bill_to": {
    "name": "Globex Corporation",
    "address": "200 Industrial Ave, Springfield, IL 62701"
  },
  "line_items": [
    {
      "description": "Premium Copy Paper (case)",
      "quantity": 10,
      "unit_price": 45.00,
      "total": 450.00
    }
  ],
  "subtotal": 798.39,
  "tax_amount": 67.86,
  "tax_rate": 0.085,
  "shipping": 15.00,
  "total_amount": 881.25,
  "currency": "USD",
  "payment_method": "ACH Transfer",
  "po_number": null,
  "notes": "Late payments subject to 1.5% monthly fee"
}
```

### Contract Schema
```json
{
  "document_type": "contract",
  "confidence": 0.95,
  "contract_number": "SLA-2024-0078",
  "contract_type": "Software License Agreement",
  "effective_date": "2024-01-01",
  "expiration_date": "2025-12-31",
  "duration_months": 24,
  "party_a": {
    "role": "Licensor",
    "name": "NovaSoft Technologies Inc.",
    "address": "1 Innovation Drive, San Jose, CA 95110",
    "signatory": "James R. Whitfield, CEO"
  },
  "party_b": {
    "role": "Licensee",
    "name": "Pinnacle Financial Group",
    "address": "800 Wall St, New York, NY 10005",
    "signatory": "Maria Chen, CTO"
  },
  "contract_value": 240000.00,
  "annual_value": 120000.00,
  "currency": "USD",
  "payment_schedule": "Annual",
  "governing_law": "State of California",
  "auto_renewal": false,
  "key_terms": [
    "Non-exclusive enterprise license",
    "24/7 Tier 1-3 support included",
    "SOC 2 Type II compliance required",
    "30-day notice for termination for cause"
  ],
  "subject": "NovaSoft Analytics Platform v4.2 — Unlimited Users"
}
```

### Receipt Schema
```json
{
  "document_type": "receipt",
  "confidence": 0.98,
  "receipt_number": "RCP-TRV-88821",
  "transaction_date": "2024-05-14",
  "merchant": {
    "name": "Marriott Marquis San Francisco",
    "address": "780 Mission St, San Francisco, CA 94103"
  },
  "transaction_type": "Hotel Accommodation",
  "payment_method": "Corporate Visa ending 7734",
  "purchaser": {
    "name": "David Kim",
    "department": "Enterprise Sales",
    "employee_id": null
  },
  "business_purpose": "Salesforce Dreamforce Conference 2024",
  "line_items": [
    {
      "description": "Deluxe King Room",
      "quantity": 3,
      "unit": "nights",
      "unit_price": 389.00,
      "total": 1167.00
    }
  ],
  "subtotal": 1566.42,
  "tax_amount": 219.30,
  "tax_rate": 0.14,
  "total_amount": 1785.72,
  "currency": "USD",
  "confirmation_number": "MRQ-4421-SF-2024",
  "expense_category": "Travel & Lodging"
}