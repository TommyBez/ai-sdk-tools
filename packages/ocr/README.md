# @ai-sdk-tools/ocr

Structured OCR for invoices and receipts with automatic provider fallback, PDF-aware text extraction, and Zod schemas. Pass any image or PDF and get back typed data.

---

## Installation

```bash
npm install @ai-sdk-tools/ocr
```

---

## Supported inputs

`ocr(input, ...)` accepts:

- `Buffer`
- `File` (browser / edge)
- `string` (http/https URL, file path, base64 string, or `data:` URI)

Inputs are normalized internally—PDFs remain PDFs so they can be piped through the PDF-specific fallback.

---

## Quick start

```ts
import { ocr, invoiceSchema, type InvoiceData } from '@ai-sdk-tools/ocr';
import fs from 'node:fs/promises';

const pdf = await fs.readFile('./invoices/acme.pdf');

const invoice = await ocr<InvoiceData>(pdf, 'invoice', {
  providers: {
    mistral: { model: 'mistral-small-latest' },
    gemini: { model: 'gemini-1.5-pro' },
  },
  qualityThreshold: {
    requireCurrency: true,
    requireTotal: true,
  },
});

console.log(invoice.vendor_name, invoice.total_amount);
```

Need a custom schema? Pass any Zod schema instead of `'invoice' | 'receipt'`:

```ts
const contractSchema = z.object({
  parties: z.array(z.string()).nullable(),
  effectiveDate: z.string().nullable(),
  summary: z.string().nullable(),
});

const contract = await ocr(pdfBuffer, contractSchema, { timeout: 30_000 });
```

---

## How the pipeline works

1. **Normalize input** – Detect media type, convert URLs/base64/files into raw data.
2. **Primary attempt (Mistral)** – Runs the configured Mistral vision model with your schema-specific prompt.
3. **Quality check** – Ensures required fields (total, currency, vendor, date) are present. You can override the thresholds.
4. **Fallback (Gemini)** – Gemini runs the same prompt/schema if the first attempt fails or quality is low. When both succeed, their fields are merged via `mergeResults`.
5. **PDF OCR fallback** – For PDFs only, text is extracted with OCR and re-run through the LLM as a last resort.
6. **Error reporting** – If every attempt fails, `OCRError` is thrown with detailed `attempts` metadata.

---

## Options reference

```ts
type OCROptions = {
  providers?: {
    mistral?: { model?: string; apiKey?: string };
    gemini?: { model?: string; apiKey?: string };
  };
  timeout?: number;              // per-attempt timeout (default 20s)
  retries?: number;              // per-attempt retries w/ backoff (default 3)
  qualityThreshold?: QualityThreshold;
};

interface QualityThreshold {
  requireTotal?: boolean;        // default true (checks total_amount)
  requireCurrency?: boolean;     // default true
  requireVendor?: boolean;       // default true
  requireDate?: boolean;         // default true (invoice_date/due_date/date)
}
```

If no provider configuration is provided, defaults are used (`mistral-small-latest` and `gemini-1.5-pro`). API keys fall back to `MISTRAL_API_KEY` / `GEMINI_API_KEY` environment variables.

---

## Schemas & types

- `invoiceSchema` / `receiptSchema`
- `type InvoiceData = z.infer<typeof invoiceSchema>` (same for receipts)
- Bring your own schema to extract any structure—Zod validation guarantees outputs.

---

## Error handling

```ts
import { ocr, OCRError } from '@ai-sdk-tools/ocr';

try {
  const receipt = await ocr(buffer, 'receipt');
} catch (error) {
  if (error instanceof OCRError) {
    console.error(error.message);
    console.table(
      error.attempts.map((attempt) => ({
        provider: attempt.provider,
        success: attempt.success,
        durationMs: attempt.duration,
        error: attempt.error?.message,
      })),
    );
  }
  throw error;
}
```

`ProviderAttempt` entries include the provider name, success flag, result/error, and duration so you can log or alert on failures.

---

## Example API route (Next.js)

```ts
import { NextResponse } from 'next/server';
import { ocr } from '@ai-sdk-tools/ocr';

export async function POST(req: Request) {
  const formData = await req.formData();
  const file = formData.get('file');

  if (!(file instanceof File)) {
    return NextResponse.json({ error: 'missing file' }, { status: 400 });
  }

  try {
    const data = await ocr(file, 'invoice');
    return NextResponse.json({ data });
  } catch (error) {
    if (error instanceof OCRError) {
      return NextResponse.json({ error: error.message, attempts: error.attempts }, { status: 422 });
    }
    throw error;
  }
}
```

---

## Tips

- **PDF vs image** – Only PDFs trigger the OCR text fallback. Images rely on the vision models.
- **Custom prompts via schema** – Model performance improves when your schema field names are descriptive.
- **Quality tuning** – Loosen thresholds if you have sparse receipts or partial documents; tighten them when you need hard guarantees.
- **Retries** – `retries` applies per provider attempt, using exponential backoff while still respecting `timeout`.

---

## License

MIT © [Midday](https://midday.ai)
