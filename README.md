# B2C Commerce Cloud — Storefront Architecture Assessment

**Author:** Luciano Diaz — Principal Solution Engineer at Salesforce

**Salesforce Credentials:**
- Certified B2C Commerce Developer
- Certified Data Cloud Consultant
- Certified Agentforce Specialist
- Certified AI Associate
- Certified Platform Foundations

**Date:** June 2026

---

## Document Objective

This document aims to provide a technical-functional assessment of the available implementation alternatives for Salesforce B2C Commerce Cloud in the context of an architecture evaluation for an organization seeking to modernize or evolve its digital commerce channel. Unlike a commercial comparison or a general overview of capabilities, this analysis focuses on explaining the architectural, functional, and operational implications of the three main storefront building approaches on B2C Commerce Cloud: Storefront Reference Architecture, PWA Kit / Composable Storefront, and a headless or composable approach at a technological level. The purpose is to deliver a clear foundation for understanding how each modality addresses the digital commerce experience, what level of flexibility it offers, which Salesforce components are involved, what their technical dependencies are, and what considerations must be evaluated in terms of time-to-market, maintainability, scalability, integration, and future evolution.

This document takes as its starting point the need to understand, in greater depth, the implementation options that Salesforce B2C Commerce Cloud offers to support a modern digital commerce strategy. In this regard, the assessment does not seek to define solely a technological preference, but rather to establish decision criteria that allow each alternative to be contrasted against business needs, the technological maturity of the current ecosystem, the capacity for integration with corporate systems, the expected experience for end users, and the operational model required to sustain the platform over time. For each approach, the role of the storefront, the relationship with B2C Commerce Cloud, the use of APIs, the separation between user experience and commerce logic, the degree of coupling with the platform, and the scenarios in which each alternative may be most suitable depending on the organization's context will be analyzed.

---

## 1. Storefront Reference Architecture (SFRA)

### What is SFRA

Storefront Reference Architecture (SFRA) is the official reference storefront provided by Salesforce as the foundation for building B2C Commerce Cloud storefronts. It is a server-side rendered application that runs entirely within the B2C Commerce Cloud platform infrastructure, meaning both the commerce logic and the presentation layer are tightly coupled and executed on Salesforce's servers.

SFRA replaced the previous reference architecture known as SiteGenesis, offering a more modular and maintainable codebase while preserving the same fundamental server-side rendering paradigm. It serves as both a functional starting point for new implementations and a demonstration of platform best practices.

### Architectural Model

SFRA follows a Model-View-Controller (MVC) pattern executed server-side on the B2C Commerce Cloud application server:

- **Controllers** are server-side JavaScript files that handle incoming HTTP requests, orchestrate business logic, and prepare data for rendering. They run on the Salesforce B2C Commerce script engine (Rhino-based) and have direct access to platform APIs for catalog, cart, checkout, customer, and order operations.

- **Models** are JavaScript objects that encapsulate and transform raw platform data into structured representations consumed by the view layer. They act as an abstraction between the platform's internal data model and what the templates need to render.

- **Views** are ISML (Internet Store Markup Language) templates — a proprietary Salesforce templating language that generates HTML on the server before sending the response to the browser. ISML supports loops, conditionals, template inclusion, and decorator patterns for layout composition.

### Cartridge System and Inheritance

The core extensibility mechanism in SFRA is the **cartridge path** — an ordered list of cartridges that determines how the platform resolves controllers, templates, static assets, and scripts at runtime. Each cartridge is essentially a self-contained module with a defined directory structure.

When a request arrives, the platform traverses the cartridge path from left to right, resolving the first matching controller or template it finds. This enables a layered override model:

- **Base cartridges** (`app_storefront_base`) contain the default SFRA implementation.
- **Custom cartridges** are placed earlier in the path and can override or extend any controller, template, or model from the base.
- **Plugin cartridges** provide optional functionality (e.g., `plugin_applepay`, `plugin_wishlist`) and are inserted between the custom and base cartridges.

This inheritance model allows teams to customize behavior without modifying the base code, which simplifies upgrade paths when Salesforce releases new versions of SFRA.

### Server-Side Rendering and Request Lifecycle

Every page request in SFRA follows this lifecycle:

1. The browser sends an HTTP request to B2C Commerce Cloud's CDN/edge layer.
2. If not cached, the request reaches the application server and is routed to the appropriate controller.
3. The controller executes business logic using platform script APIs (accessing catalogs, pricing, promotions, inventory, etc.).
4. The controller passes a view model to an ISML template.
5. The template renders the final HTML on the server.
6. The complete HTML response is returned to the browser.

Client-side JavaScript in SFRA is limited to progressive enhancement — form validation, AJAX calls for mini-cart updates, and UI interactions. The page structure and content are always resolved server-side.

### Relationship with Business Manager

SFRA is deeply integrated with Business Manager, the B2C Commerce Cloud back-office application. Merchants use Business Manager to manage:

- Product catalogs, categories, and pricing
- Promotions and campaigns
- Content slots and page layouts
- Site preferences and configuration
- Customer segments and A/B testing

Changes made in Business Manager are immediately reflected in the SFRA storefront without requiring code deployments. This tight coupling between the merchandising tool and the storefront is a defining characteristic of the SFRA model.

### Development Workflow

SFRA development relies on the following tools and processes:

- **Prophet Debugger / B2C Commerce IDE extensions** for VS Code provide code upload, debugging, and log streaming directly against sandbox instances.
- **Code deployment** is done by uploading cartridges to a sandbox via WebDAV or through CI/CD pipelines using the Salesforce B2C Commerce CLI (sfcc-ci).
- **Sandboxes** are individual development environments provided by the platform, each running a full instance of B2C Commerce Cloud.
- **ISML linting and compilation** happens at upload time — there is no local build step for templates.

The development feedback loop involves uploading code to a remote sandbox and testing against that environment, as there is no local runtime that replicates the B2C Commerce script engine.

### Technology Stack Summary

| Layer | Technology |
|-------|-----------|
| Runtime | B2C Commerce Cloud application server (Rhino-based JS engine) |
| Controllers | Server-side JavaScript (CommonJS modules) |
| Templates | ISML (Internet Store Markup Language) |
| Client-side | jQuery, vanilla JavaScript, AJAX |
| Styling | SCSS compiled to CSS (via build tools) |
| Build tools | Webpack (for client-side assets only) |
| Deployment | WebDAV upload / sfcc-ci CLI |
| Back-office | Business Manager |
| APIs consumed | Platform Script API (server-side, internal) |

### When to Use SFRA

SFRA is the standard approach for launching a storefront on B2C Commerce Cloud. It works with native JavaScript and the platform's own ecosystem (ISML, cartridges, Business Manager) without relying on external frontend libraries like React or Vue. It represents the first degree of implementation — the direct and complete path, with everything integrated.

### Strengths

- Comprehensive out-of-the-box solution with full commerce functionality from day one.
- Immediate merchandising capabilities through Business Manager — no code deployment needed for content, promotions, or catalog changes.
- Lower operational complexity — no separate frontend infrastructure to host, scale, or monitor.
- Mature and battle-tested at scale, with spectacular production sites across industries worldwide.

### Considerations

- The frontend is coupled to the platform, which simplifies operations but means that UX development is resolved within the B2C Commerce ecosystem rather than with external libraries like React or Vue.
- Development requires uploading code to remote sandboxes — there is no local runtime for the B2C Commerce script engine.
- The server-side rendering model means page transitions involve full round-trips to the server, unlike single-page application architectures.
