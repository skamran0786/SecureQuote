# SecureQuote — CCTV Quotation Maker

A standalone, local-first quotation workspace for SecureView Solutions. This application does not have a backend or require login: quotations, customers, products, and settings stay in IndexedDB in the browser and device where they are created. Browser storage is not a backup; export important quotations as PDFs and keep a separate copy of any records you need to retain.

## Architecture

```text
React pages and forms
        ↓
Repository interfaces (src/data/repositories.ts)
        ↓
IndexedDB adapter (src/data/indexedDbRepository.ts)
        ↓
Browser-local object stores
```

UI components depend on typed repository contracts, not on IndexedDB APIs. `DataProvider` is the UI-facing access boundary; `IndexedDbRepository` implements those contracts today. A hosted database can be added by implementing the same interfaces and selecting an adapter at the composition root, without rewriting quotation pages. Domain types and deterministic price calculations live in `src/domain`, independently of UI and persistence.

Routes:

- `/` — dashboard
- `/quotations` — searchable quotation register
- `/quotations/new` — create a quotation
- `/quotations/:id` — view and export a quotation
- `/quotations/:id/edit` — edit a draft quotation
- `/products` — product catalogue
- `/customers` — customer directory
- `/settings` — company, GST, and quote defaults

## Data model and calculation conventions

`src/domain/models.ts` defines customers, products, immutable quotation-item and company snapshots, quotations, company and tax settings, and application preferences. Changing catalogue prices, company details, or tax defaults does not silently change an existing quote. All persisted monetary amounts are integer **paise**; display conversion to rupees happens only at input/output boundaries. Quantities are stored as integer thousandths; GST and percentage discounts are integer basis points (100 basis points = 1%).

`src/domain/calculations.ts` is the single source of truth:

1. Round each extended item amount to the nearest paise, half up.
2. Apply and round each line discount to the nearest paise, half up.
3. For GST-exclusive prices, calculate tax on the discounted taxable amount and round tax per line, half up. For GST-inclusive prices, extract tax from the discounted gross; taxable amount is gross minus that tax.
4. For intra-state supplies, split each line's rounded GST into CGST (floor half) and SGST (remaining paise). For inter-state supplies, apply the full amount as IGST.
5. Sum already-rounded line values. Optional round-off moves the payable total to the nearest whole rupee; its signed value is disclosed on the quote.

These conventions avoid floating-point money storage and make the displayed totals, persisted values, and exported PDF consistent.

## Development roadmap and acceptance criteria

1. **Foundation and domain** — typed entity contracts, integer-paise calculation rules, independent Vite/TypeScript project, and calculation tests. *Accept when strict type checking and unit tests pass.*
2. **Local persistence** — IndexedDB object stores, repository contracts, starter demonstration records, and first-run loading/error feedback. *Accept when edits survive a reload on the same browser/device and the UI has no backend dependency.*
3. **Quotation workflow** — dashboard, search/register, create/edit/view, status changes, GST breakdown, and PDF export. *Accept when an end-to-end quotation persists and its PDF matches the calculated quote.*
4. **Business directories and settings** — create/edit/delete customers and products, company identity, GST configuration, and preference persistence. *Accept when quote snapshots remain stable after catalogue edits and settings affect new calculations.*
5. **Release readiness** — responsive layouts, keyboard and screen-reader support, input validation, privacy/security review, production build, browser smoke test, and static-host deployment guidance. *Accept when test/build checks pass and no API keys or customer data are required at build time.*

## Quality, accessibility, and security requirements

- Run `npm.cmd test`, `npm.cmd run typecheck`, and `npm.cmd run build` before release.
- Test deterministic calculations at paise boundaries, GST-inclusive/exclusive, CGST/SGST/IGST splits, and optional round-off. Add browser-level checks for quote creation, persistence, edit, and PDF export before production rollout.
- Use semantic landmarks, explicit labels, visible keyboard focus, keyboard-operable controls, meaningful status text, and responsive/reflow-friendly pages. Respect reduced-motion preferences.
- Validate user input at form boundaries with Zod. Render user-supplied strings as React text; do not inject HTML. Do not store credentials, payment data, or secrets in browser storage.
- IndexedDB is origin-scoped, device-local storage—not encryption, authentication, synchronization, or backup. Anyone with access to this browser profile may access this data. Clear browser site data only if you intend to erase local records.
- Generated PDFs are local downloads. Verify business identity, GSTIN, state, tax mode, prices, and applicable Indian tax requirements with a qualified professional before issuing real invoices/quotations.

## Run locally and deploy

```powershell
npm.cmd install
npm.cmd run dev
```

For a deployable static build, run `npm.cmd run build` and publish `dist/` to any static host. Configure SPA fallback/rewrite-to-`index.html` for nested routes; no server-side environment variables or database credentials are needed. To preview the production build locally, run `npm.cmd run preview`.

## Credits

Built by [Mohd Kamran Siddiquee](https://mohdkamransiddique.vercel.app/) · [Developer profile](https://mohdkamransiddique.vercel.app/)
