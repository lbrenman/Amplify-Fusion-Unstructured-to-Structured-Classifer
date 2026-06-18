# Amplify Fusion - Unstructured to Structured Classifer

An Amplify Fusion unstructured data to structured data classifer demonstration implemented as follows:

* Invoice, contract and receipt PDF files are ingested by SFTP
* PDF is converted to text using [LlamaParse](https://www.llamaindex.ai/)
* Text classified and converted to predefined schema by a [Grok](https://groq.com/) `llama-3.1-8b-instant` LLM using the Fusion OpenAI Connector
* Data is written to an API (Webhook.site)
* Structured data is inserted into a [Neon Postgres](https://neon.com/) database for reporting and downstream processing/usage

The Fusion project export, LLM prompt and test PDF sample files are all incuded.

## How to Use

* Download repo
* Import the `Unstr2StructClassifier.zip` project export into your Fusion tenant
* Configure connection with your credentials for LlamaParse and Neon Postgres
* Set Postgres Table using SQL below
* Use an SFTP client such FileZilla to upload pdf files to Fusion

## Postgres DB

* Create the Postgres database table as follows:

    ```sql
    CREATE TABLE documents (
        id                  SERIAL PRIMARY KEY,
        document_type       VARCHAR(20) NOT NULL,
        confidence          NUMERIC(4,3),
        source_filename     TEXT NOT NULL,
        raw_text            TEXT,
        extracted_data      JSONB NOT NULL,
        document_number     TEXT,
        document_date       DATE,
        expiration_date     DATE,
        total_amount        NUMERIC(14,2),
        currency            VARCHAR(3) DEFAULT 'USD',
        party_name          TEXT,
        counterparty_name   TEXT,
        payment_method      TEXT,
        status              TEXT DEFAULT 'processed',
        created_at          TIMESTAMPTZ DEFAULT NOW(),
        updated_at          TIMESTAMPTZ DEFAULT NOW()
    );
    ```

* Auto update the `updated_at` column

    ```sql
    CREATE OR REPLACE FUNCTION update_updated_at()
    RETURNS TRIGGER AS $$
    BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
    $$ LANGUAGE plpgsql;

    CREATE TRIGGER trg_documents_updated_at
        BEFORE UPDATE ON documents
        FOR EACH ROW EXECUTE FUNCTION update_updated_at();
    ```

* Set indexes for common query patterns

    ```sql
    CREATE INDEX idx_documents_type         ON documents (document_type);
    CREATE INDEX idx_documents_date         ON documents (document_date);
    CREATE INDEX idx_documents_expiration   ON documents (expiration_date);
    CREATE INDEX idx_documents_party        ON documents (party_name);
    CREATE INDEX idx_documents_counterparty ON documents (counterparty_name);
    CREATE INDEX idx_documents_amount       ON documents (total_amount);
    CREATE INDEX idx_documents_number       ON documents (document_number);
    CREATE INDEX idx_documents_jsonb        ON documents USING GIN (extracted_data);
    ```

* Useful questies, once table is populated

    ```sql
    -- ── 1. Overview: count and total value by document type
    SELECT
        document_type,
        COUNT(*)                        AS doc_count,
        SUM(total_amount)               AS total_value,
        AVG(total_amount)               AS avg_value,
        MIN(total_amount)               AS min_value,
        MAX(total_amount)               AS max_value
    FROM documents
    GROUP BY document_type
    ORDER BY total_value DESC;


    -- ── 2. All invoices ordered by amount, with vendor and due date
    SELECT
        document_number                                   AS invoice_number,
        party_name                                        AS vendor,
        counterparty_name                                 AS billed_to,
        document_date                                     AS invoice_date,
        expiration_date                                   AS due_date,
        total_amount,
        extracted_data->>'payment_terms'                  AS payment_terms
    FROM documents
    WHERE document_type = 'invoice'
    ORDER BY total_amount DESC;


    -- ── 3. Contracts expiring in the next 90 days
    SELECT
        document_number,
        extracted_data->>'contract_type'                  AS contract_type,
        party_name                                        AS party_a,
        counterparty_name                                 AS party_b,
        expiration_date,
        expiration_date - CURRENT_DATE                    AS days_remaining,
        total_amount                                      AS contract_value
    FROM documents
    WHERE document_type = 'contract'
    AND expiration_date BETWEEN CURRENT_DATE AND CURRENT_DATE + INTERVAL '90 days'
    ORDER BY expiration_date;


    -- ── 4. All receipts with expense category and purchaser
    SELECT
        document_number                                           AS receipt_number,
        document_date,
        party_name                                                AS merchant,
        counterparty_name                                         AS employee,
        extracted_data->>'expense_category'                       AS expense_category,
        extracted_data->>'business_purpose'                       AS purpose,
        total_amount
    FROM documents
    WHERE document_type = 'receipt'
    ORDER BY document_date DESC;


    -- ── 5. High-value invoices (> $5,000) with line item count
    SELECT
        document_number,
        party_name                                                AS vendor,
        total_amount,
        jsonb_array_length(extracted_data->'line_items')          AS line_item_count,
        extracted_data->>'po_number'                              AS po_number
    FROM documents
    WHERE document_type = 'invoice'
    AND total_amount > 5000
    ORDER BY total_amount DESC;


    -- ── 6. Explode invoice line items into rows
    SELECT
        d.document_number,
        d.party_name                                              AS vendor,
        d.document_date,
        item->>'description'                                      AS item_description,
        (item->>'quantity')::NUMERIC                              AS quantity,
        (item->>'unit_price')::NUMERIC                            AS unit_price,
        (item->>'total')::NUMERIC                                 AS line_total
    FROM documents d,
        jsonb_array_elements(d.extracted_data->'line_items') AS item
    WHERE d.document_type = 'invoice'
    ORDER BY d.document_number, line_total DESC;


    -- ── 7. Monthly spend by document type
    SELECT
        DATE_TRUNC('month', document_date)                        AS month,
        document_type,
        COUNT(*)                                                  AS count,
        SUM(total_amount)                                         AS total_spend
    FROM documents
    GROUP BY DATE_TRUNC('month', document_date), document_type
    ORDER BY month, document_type;


    -- ── 8. JSONB search: find all receipts categorized as Travel & Lodging
    SELECT
        document_number,
        document_date,
        party_name                                                AS merchant,
        total_amount,
        extracted_data->'purchaser'->>'name'                      AS employee,
        extracted_data->>'business_purpose'                       AS purpose
    FROM documents
    WHERE document_type = 'receipt'
    AND extracted_data->>'expense_category' = 'Travel & Lodging';


    -- ── 9. JSONB search: contracts with auto_renewal = false expiring this year
    SELECT
        document_number,
        extracted_data->>'contract_type'                          AS type,
        party_name,
        counterparty_name,
        expiration_date,
        total_amount
    FROM documents
    WHERE document_type = 'contract'
    AND (extracted_data->>'auto_renewal')::BOOLEAN = FALSE
    AND expiration_date <= DATE_TRUNC('year', CURRENT_DATE) + INTERVAL '1 year - 1 day'
    ORDER BY expiration_date;


    -- ── 10. Contracts: extract milestone payment schedule
    SELECT
        d.document_number,
        d.party_name                                              AS consultant,
        d.counterparty_name                                       AS client,
        milestone->>'milestone'                                   AS milestone_id,
        milestone->>'description'                                 AS description,
        milestone->>'due_date'                                    AS due_date,
        (milestone->>'amount')::NUMERIC                           AS payment_amount
    FROM documents d,
        jsonb_array_elements(d.extracted_data->'payment_schedule') AS milestone
    WHERE d.document_type = 'contract'
    AND d.extracted_data ? 'payment_schedule'
    ORDER BY d.document_number, (milestone->>'due_date');


    -- ── 11. Tax summary across invoices and receipts
    SELECT
        document_type,
        document_number,
        party_name,
        total_amount,
        (extracted_data->>'tax_amount')::NUMERIC                  AS tax_amount,
        ROUND((extracted_data->>'tax_rate')::NUMERIC * 100, 2)    AS tax_rate_pct
    FROM documents
    WHERE document_type IN ('invoice', 'receipt')
    AND (extracted_data->>'tax_amount')::NUMERIC > 0
    ORDER BY tax_amount DESC;


    -- ── 12. Full-text JSONB containment: find docs from a specific vendor
    SELECT
        document_type,
        document_number,
        document_date,
        total_amount
    FROM documents
    WHERE extracted_data @> '{"vendor": {"name": "CloudTech Solutions LLC"}}';


    -- ── 13. Documents needing review (low confidence)
    SELECT
        id,
        document_type,
        source_filename,
        confidence,
        status
    FROM documents
    WHERE confidence < 0.90
    ORDER BY confidence;


    -- ── 14. Vendor spend leaderboard across invoices
    SELECT
        party_name                                                AS vendor,
        COUNT(*)                                                  AS invoice_count,
        SUM(total_amount)                                         AS total_billed,
        AVG(total_amount)                                         AS avg_invoice_size
    FROM documents
    WHERE document_type = 'invoice'
    GROUP BY party_name
    ORDER BY total_billed DESC;
    ```