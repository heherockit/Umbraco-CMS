# Umbraco Ecommerce Extension: Product Management Spec

Version 0.2 (draft) · Module 1 of the ecommerce feature · Working namespace: `Umbraco.Cms.Ecommerce`

## 1. Purpose and scope

Build an ecommerce feature inside a fork of the open source Umbraco CMS repository. This document specifies **Module 1: Product Management**: how merchants create, organise, price, stock and publish products inside the Umbraco backoffice, and how storefronts read them.

In scope: product catalogue, variants, pricing, inventory, categories, attributes, media, SEO, publishing workflow, read APIs.

Out of scope for this module (later modules): cart, checkout, payments, orders, shipping, tax calculation, customers/accounts, discounts, reporting.

## 2. Assumptions

- A1. Target the current Umbraco LTS release (assumed Umbraco 17 LTS on .NET 10; verify against the Umbraco release roadmap before starting).
- A2. The feature is developed inside a fork of the open source Umbraco-CMS repository (MIT licence), as new projects in the solution plus small, targeted changes to existing projects. It is built and deployed with the fork, not installed as a separate NuGet package.
- A3. Backoffice UI is a new package inside the repository's backoffice client project, built on the Umbraco extension registry (manifests, Lit web components, TypeScript). The legacy AngularJS backoffice is not supported.
- A4. Database access uses EF Core with the same database as Umbraco (SQL Server first, SQLite for dev), using the repository's EF Core persistence project and migration approach (to be confirmed in the Phase 0 spike, since the core otherwise uses NPoco). Own tables use a `umbracoEcommerce_` prefix.
- A5. Storefront is typically headless (Delivery API consumers) or Razor/MVC on the same site; the extension supports both.
- A6. Single store and single currency in v1, but the data model is multi-currency ready.
- A7. Merchant team is small to mid-size; roles are Editor, Merchandiser, Store Admin.
- A8. This is a new build, not a wrapper around Umbraco Commerce (the official paid product). Conflicts and overlaps with it are treated as a product decision, not a technical constraint.

* A9. Fork the Umbraco-CMS repository on GitHub and branch from the release tag of the target LTS version, not from `main`. All work stays on a long-lived feature branch.
* A10. New code goes in new projects under `src/`. Existing core projects are touched only for registration, solution wiring and backoffice package registration, to keep upstream merges small.
* A11. Upstream patch and security releases are merged into the fork on a fixed cadence (monthly assumed).

## 3. Reuse versus build: Umbraco features considered

| Need | Existing Umbraco feature | Decision |
| --- | --- | --- |
| Product marketing content (name, description, rich text, images) | Document Types, Content tree, Block List/Grid, Rich Text editor | **Reuse.** Product is a Document Type. |
| Shared fields across product types | Compositions, Element Types | **Reuse.** `Product Base` composition. |
| Languages / regional content | Content variants (culture), Dictionary | **Reuse** for content; price/stock stay invariant. |
| Publish, schedule, unpublish, rollback | Publishing, scheduled publishing, content versioning | **Reuse** for content fields only. |
| Images, documents, video | Media Library, Media Picker, image cropper | **Reuse.** |
| Routing, URLs, redirects | Routing, URL segments, redirect tracker | **Reuse.** |
| SEO fields | Property editors + composition | **Reuse** (`SEO` composition). |
| Categories | Content tree (hierarchy) or Content Picker | **Reuse** as Document Type `Category`. |
| Permissions | Users, User Groups, granular document permissions, sections | **Reuse** and extend with commerce permissions. |
| Search in backoffice | Examine (Lucene) | **Reuse** for admin search; add a storefront search abstraction. |
| Headless read access | Delivery API | **Reuse** for product content; extend with commerce data. |
| Events | Notifications / webhooks | **Reuse** to react to publish/delete; emit own notifications. |
| Import/export, deploy between environments | uSync, Umbraco Deploy (schema) | **Reuse** for Document Type schema; products' commerce data needs own import/export. |
| Price, stock, SKU, variants | None | **Build.** |
| Product list with bulk edit and filters | Content tree/list view is limited | **Build** a collection view. |

## 4. Key architectural decision

**Hybrid model.**

- **Product content** (title, descriptions, media, SEO, category, attributes for display) lives in Umbraco as published **content nodes**. This gives variants, versioning, routing, workflow and the Delivery API for free.
- **Commercial data** (SKUs, prices, stock, variant combinations) lives in **own tables**, keyed by the product's content `Guid`. These values change often and must not create content versions or require republishing.
- A custom **Product Commerce** property editor on the product Document Type embeds the SKU/price/stock editing UI inside the normal content workspace, so merchants edit everything on one screen.

Rejected alternatives: (a) everything as content properties (price/stock edits bloat version history, cache churn); (b) everything in custom tables with a bespoke section (loses routing, culture variants, media, workflow, Delivery API).

### Where the code lives in the fork

The module is built as new projects beside the existing ones in the repository's `src` folder. The project names below are proposals to confirm in the Phase 0 spike.

| Layer | Existing project to extend | New code |
| --- | --- | --- |
| Domain models, notifications, service interfaces | `Umbraco.Core` | `Umbraco.Cms.Ecommerce.Core` |
| Services, repositories, migrations, composer | `Umbraco.Infrastructure` | `Umbraco.Cms.Ecommerce.Infrastructure` |
| Backoffice API (SKU, price, stock CRUD) | `Umbraco.Cms.Api.Management` | `Umbraco.Cms.Ecommerce.Api.Management` |
| Storefront read API | `Umbraco.Cms.Api.Delivery` | `Umbraco.Cms.Ecommerce.Api.Delivery` |
| Razor storefront helpers | `Umbraco.Web.Website` | `Umbraco.Cms.Ecommerce.Web` |
| Backoffice UI (property editor, collection, settings) | `Umbraco.Web.UI.Client` | New `ecommerce` package in its `src/packages` folder |
| Test site and sample data | `Umbraco.Web.UI` | Reference to the new projects, seed data |
| Tests | existing `tests` projects | New unit and integration test projects |

The `Product` and `Category` Document Types, the `Product Base` and `SEO` compositions and the Product Commerce Data Type are created by a migration in the new Infrastructure project, so a clean build of the fork has them without uSync. Registration happens through an `IComposer` in that project; the only edits to existing files are project references, the solution file and the backoffice client's package registration.

## 5. Domain model

**Product** (Umbraco content node, Document Type `Product`, composes `Product Base` and `SEO`)

- Name, URL segment, short and long description, media gallery, brand, categories (multi-picker), product type, tax class code, attribute display values.
- Culture variants allowed; commerce data is invariant.

**ProductCommerce** (own table, 1:1 with product node)

- `ProductKey` (content Guid), `ProductTypeKind` (simple or variable), `Status` (draft, active, archived), `CreatedUtc`, `UpdatedUtc`.

**Sku** (own table, many per product)

- `SkuId`, `ProductKey`, `SkuCode` (unique), `Barcode`, `OptionValues` (e.g. Size=M, Colour=Red), `WeightGrams`, `Dimensions`, `IsDefault`, `SortOrder`, `IsActive`.
- Simple products have exactly one SKU.

**Price** (own table)

- `PriceId`, `SkuId`, `CurrencyCode`, `PriceListId`, `Amount`, `CompareAtAmount`, `ValidFromUtc`, `ValidToUtc`. v1 uses one default price list; the schema allows more.

**StockItem** (own table)

- `SkuId`, `LocationId`, `OnHand`, `Reserved`, `ReorderPoint`, `AllowBackorder`, `TrackInventory`. v1 uses one default location.

**ProductOption / ProductOptionValue**

- Option definitions per product (Size, Colour) used to generate SKU combinations. Store-wide reusable option sets are supported (Data Type driven).

**Category** (Umbraco content node, Document Type `Category`)

- Hierarchical via the content tree; optional `Parent Category` for products belonging to several trees.

**ProductAttribute** (Element Type based)

- Merchant-defined specification fields (Material, Warranty) as Block List items so they stay editable and translatable.

## 6. Functional requirements

Priority: M = must, S = should, C = could.

### 6.1 Catalogue management

- FR-1 (M). Merchants create products as Umbraco documents from the `Product` Document Type, using the standard create dialog and tree.
- FR-2 (M). Product supports simple (single SKU) and variable (options and many SKUs) types, switchable before first SKU sale.
- FR-3 (M). Product has a status independent of content publish state: content must be published **and** commerce status active for the product to be purchasable.
- FR-4 (M). Categories are Document Types; products can belong to multiple categories; moving a category in the tree updates breadcrumbs and URLs.
- FR-5 (S). Duplicate product (content plus SKUs, with new SKU codes and unpublished).
- FR-6 (S). Archive product; archived products stay in the database, are hidden from storefront APIs and are blocked from the new-order flow.
- FR-7 (M). Deleting a product to the recycle bin soft-deletes commerce data; restore brings it back; permanent delete is blocked if the product appears in a past order (enforced once the Orders module exists).

### 6.2 Variants and SKUs

- FR-8 (M). Define up to 3 options per product, each with up to 50 values; generate the Cartesian SKU matrix with a preview.
- FR-9 (M). Editable SKU grid: code, barcode, price, compare-at price, stock, weight, active toggle, image.
- FR-10 (M). SKU code uniqueness validated across the store with a clear error message.
- FR-11 (S). Adding or removing option values regenerates only the affected SKUs and never silently deletes SKUs that have stock or order history.
- FR-12 (C). Per-SKU media assignment.

### 6.3 Pricing

- FR-13 (M). Price per SKU in the store currency with decimal precision defined per currency.
- FR-14 (S). Compare-at price and a computed discount badge value exposed to storefronts.
- FR-15 (S). Scheduled price changes using `ValidFromUtc`/`ValidToUtc`.
- FR-16 (C). Bulk price update (percentage or fixed) on a filtered selection.
- FR-17 (M). Price history is auditable (who, when, old and new value).

### 6.4 Inventory

- FR-18 (M). Track on-hand quantity per SKU; toggle tracking off for unlimited items.
- FR-19 (M). Manual stock adjustment with a reason and an audit trail.
- FR-20 (S). Low-stock threshold and a backoffice dashboard widget listing low or out-of-stock SKUs.
- FR-21 (S). Allow backorder flag per SKU.
- FR-22 (C). CSV stock import.

### 6.5 Content, media and SEO

- FR-23 (M). Product description, gallery and SEO fields use the standard Umbraco editors; culture variants supported for all content fields.
- FR-24 (M). Media gallery uses the Media Picker with a main image designation and per-image alt text.
- FR-25 (S). Structured data (schema.org Product JSON-LD) generated from product, price and stock data.

### 6.6 Merchandising backoffice experience

- FR-26 (M). **Products collection view**: sortable, filterable table (name, SKU, price, stock, status, category, last updated) with pagination, available as a workspace view on the Product root node and as a dedicated menu entry.
- FR-27 (S). Bulk actions: publish, unpublish, archive, change category, update price.
- FR-28 (S). Inline edit of price and stock in the collection view.
- FR-29 (M). Saved filters per user.

### 6.7 Import and export

- FR-30 (S). CSV export of products and SKUs.
- FR-31 (S). CSV import with validation report and dry-run mode; unmatched rows never fail the whole file.
- FR-32 (M). Document Type schema deployable between environments with uSync or Umbraco Deploy; commerce data excluded and handled by the import/export feature.

### 6.8 Read APIs

- FR-33 (M). Extend the **Delivery API** product item response with a `commerce` block: SKUs, options, price, availability, with ETag caching.
- FR-34 (M). Public endpoints for products by id/slug, list by category, search by text/filter/sort, all paginated.
- FR-35 (M). Management API (backoffice only) for CRUD on SKU, price, stock, with OpenAPI documentation and the same authorisation as the backoffice.
- FR-36 (S). C# services for Razor storefronts: `IProductQueryService`, `IPricingService`, `IInventoryService`.

## 7. Backoffice UX

Extension types to register through the manifest registry, in the new \`ecommerce\` package of the backoffice client:

- **Section**: none required. Products live in the Content section; a **Commerce** section is added later for orders and settings.
- **Property editor UI** `Product Commerce`: tabs for Pricing, Variants and Inventory inside the product workspace.
- **Workspace view**: `Commerce` view on the Product document (summary, status, audit log).
- **Collection** with custom table column and action manifests for the products list (FR-26 to FR-29).
- **Entity actions**: Duplicate, Archive.
- **Dashboard widget**: low stock.
- **Settings tree** (Settings section): option sets, currencies, tax class codes, default location.

UX rules: all lists are keyboard accessible; destructive actions require confirmation; unsaved SKU grid changes warn on leave; the grid supports at least 500 SKUs without freezing (virtualised rows).

## 8. Security and permissions

- New granular permissions: `Commerce.Products.View`, `.Edit`, `.Price`, `.Stock`, `.Archive`, `.Import`.
- Permissions are assigned to User Groups; the default groups `Merchandiser` (all but Price) and `Store Admin` (all) are seeded.
- Public endpoints expose only active and published products; prices and stock are returned only in the shape defined in the API contract.
- All inputs validated server-side; CSV import limited by size and content type.

## 9. Events and extensibility

- Notifications (cancellable where relevant): `ProductSavingNotification`, `ProductSavedNotification`, `SkuPriceChangedNotification`, `StockAdjustedNotification`, `ProductArchivedNotification`.
- Provider interfaces so implementers can replace behaviour: `IPriceProvider` (default: database price list), `IStockProvider` (default: local table, replaceable by an ERP), `ISkuCodeGenerator`.
- Umbraco notification handlers are used to keep product commerce data in sync when a content node is trashed, restored or moved.
- Webhook events published for product and stock changes using Umbraco's webhook system.

## 10. Non-functional requirements

- NFR-1. Public product read API p95 below 150 ms for cached responses and 400 ms uncached with 50,000 SKUs.
- NFR-2. Backoffice collection loads the first page in under 1 s with 50,000 SKUs.
- NFR-3. Works on Umbraco load-balanced setups: cache invalidation uses Umbraco's distributed cache; no in-process only state.
- NFR-4. Database migrations are idempotent and run on start-up through Umbraco's migration plan; every migration has a tested rollback note.
- NFR-5. Localisation: backoffice strings via localization files (English first, structure ready for others).
- NFR-6. Test coverage: unit tests for pricing, SKU generation and stock rules; integration tests in the repository's existing test setup against SQL Server and SQLite; Playwright tests for the main merchant journeys.
- NFR-7. Accessibility to WCAG 2.1 AA for backoffice extensions that Umbraco's own components do not already cover.

* NFR-8. Merging a monthly upstream release into the fork must not need changes in more than 10 existing files; a CI job reports the count of modified upstream files per pull request.
* NFR-9. The fork's CI builds the whole solution, runs the existing upstream tests plus the new tests, and fails when an upstream test breaks.

## 11. Delivery plan

| Phase | Content | Outcome |
| --- | --- | --- |
| 0. Spike (1-2 weeks) | Fork the repository, build and run it locally, add the first new project, prototype the hybrid model, one property editor and one collection view | Validate A1-A4 and the architecture decision |
| 1. Foundation (3-4 weeks) | Project skeletons in the fork, migrations, Document Types, compositions, permissions, notifications, CI | Fork builds and boots with an empty catalogue |
| 2. Simple products (3 weeks) | SKU, price, stock for simple products; Commerce property editor; Delivery API extension | FR-1, 3, 4, 13, 18, 19, 33, 34 |
| 3. Variants (3 weeks) | Options, SKU matrix, SKU grid | FR-2, 8-11 |
| 4. Merchandising (3 weeks) | Collection view, bulk actions, import/export, low stock widget | FR-26 to FR-32 |
| 5. Hardening (2 weeks) | Performance, load-balanced test, docs, sample storefront | NFR-1 to NFR-7 |

## 12. Acceptance criteria (module level)

- A merchant can create a variable product with 12 SKUs, set prices and stock, publish it, and see it through the Delivery API in under 10 minutes without developer help.
- Changing a price or stock value does not create a new content version and is visible on the storefront within the configured cache time.
- A Merchandiser cannot change prices; a Store Admin can; both actions appear in the audit log.
- Trashing and restoring a product keeps its SKUs, prices and stock intact.
- A clean build of the fork boots against an empty database, runs the migrations once and shows the Product and Category Document Types.

## 13. Open questions

1. Is overlap with Umbraco Commerce acceptable, or should the extension interoperate with it (for example by reading its product data)?
2. Which database engines must be supported at launch (SQL Server only, or SQLite and PostgreSQL too)?
3. Is multi-currency or multi-store required within the first year? It changes the Price and Stock schema design.
4. Should prices include tax? Decide before the Pricing phase; it affects the Price schema and API contract.
5. Who owns the storefront: a headless front end team or Razor templates on the same site? This sets priority between FR-33/34 and FR-36.
6. Is a digital product type (downloads, no stock) needed in v1?

7) Which release tag is the baseline for the fork, and how often are upstream changes merged in?
8) Is the fork private, or should the module be offered back to the upstream project as a pull request? This decides how much existing core code it may change.
