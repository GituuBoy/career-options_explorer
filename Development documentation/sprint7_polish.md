# Sprint 7: Polish & Release Readiness

## 1. Narrative Goal

In this final "Polish" sprint, our objective is to transform the Career Options Explorer from a functional application into a refined, professional, and production-ready product. Over these three days, we will meticulously focus on:
1.  **Accessibility (A11y):** Ensuring the application is WCAG 2.1 AA compliant, usable by people with diverse abilities, through keyboard navigation enhancements, ARIA attribute implementation, screen reader compatibility checks, and color contrast adjustments.
2.  **Performance Optimization:** Targeting Lighthouse scores of ≥ 90 for Performance and Accessibility by fine-tuning image delivery (next-gen formats), optimizing font loading, verifying code-splitting, and refining caching strategies.
3.  **Comprehensive Documentation:** Producing clear and thorough `README.md`, API documentation (`API.md`), component guides (`COMPONENTS.md`), and a `CHANGELOG.md`.
4.  **Release Preparation:** Finalizing environment configurations, creating basic deployment scripts, and tagging a v1.0 release.
This sprint is about meticulous refinement, ensuring a high-quality user experience for everyone, robust performance, and a well-documented codebase for future maintainability and development.

## 2. Back-of-the-Envelope Design (Focus Areas)

```mermaid
graph TD
    subgraph "Accessibility (A11y) Focus - WCAG 2.1 AA"
        A11yAudit[A11y Audit (axe-core, Lighthouse)] --> KeyboardNav[Full Keyboard Navigation]
        A11yAudit --> ScreenReaderTesting[Screen Reader Testing (NVDA/VoiceOver)]
        A11yAudit --> AriaImplementation[ARIA Attributes & Roles]
        A11yAudit --> ColorContrastCheck[Color Contrast Ratios]
        A11yAudit --> FocusManagement[Logical Focus Order & Visible Focus Styles]
        KeyboardNav --> SkipLinks[Skip Links Implementation]
    end

    subgraph "Performance Optimization Focus - Lighthouse >= 90"
        PerfAudit[Lighthouse Performance Audit] --> ImageOpt[Image Optimization (WebP, Responsive Sizes)]
        PerfAudit --> FontOpt[Font Loading Strategy (preload, display:swap)]
        PerfAudit --> CodeSplittingVerify[Verify Route & Component Code Splitting]
        PerfAudit --> CachingReview[Review Client & Server Caching]
        PerfAudit --> BundleAnalysis[Bundle Size Analysis (vite-bundle-visualizer)]
        PerfAudit --> ResourceHints[Resource Hints (preconnect, preload, prefetch)]
    end

    subgraph "Documentation Suite"
        DevDocs[Developer Documentation]
        UserDocs[User-Facing Info (About/FAQ - from Sprint 5)]
        
        DevDocs --> MainReadme[Comprehensive README.md (Setup, Arch, Scripts)]
        DevDocs --> APIDocs[API.md (Endpoints, Schemas, Auth)]
        DevDocs --> ComponentDocs[COMPONENTS.md (Key Component Props & Usage)]
        DevDocs --> ChangelogFile[CHANGELOG.md (Version 1.0.0)]
    end

    subgraph "Release Preparation"
        FinalTesting[Final End-to-End Testing] --> EnvConfig[Environment Config (.env.production.example)]
        EnvConfig --> BuildProcess[Production Build (Client & Server)]
        BuildProcess --> DeployScripts[Basic Deployment Scripts]
        DeployScripts --> VersionTagging[Git Tag v1.0.0]
    end
```

## 3. Task Checklist

### Accessibility (A11y) Enhancements - WCAG 2.1 AA Target

- [ ] **Setup `react-axe` for Development-Time Audits** (in `client/src/main.tsx` or `index.tsx`):
  ```bash
  cd client
  npm install --save-dev axe-core @axe-core/react 
  ```
  ```typescript
  // client/src/main.tsx (or index.tsx)
  import React from 'react';
  import ReactDOM from 'react-dom/client';
  import App from './App';
  import './index.css';
  import { HelmetProvider } from 'react-helmet-async'; // If not already there

  if (process.env.NODE_ENV !== 'production') {
    const axe = await import('@axe-core/react');
    axe.default(React, ReactDOM, 1000, {}); // Empty config object for default rules
  }

  ReactDOM.createRoot(document.getElementById('root')!).render(
    <React.StrictMode>
      <HelmetProvider> {/* Ensure HelmetProvider wraps App */}
        <App />
      </HelmetProvider>
    </React.StrictMode>
  );
  ```
- [ ] **Implement `SkipLink.tsx`** (`client/src/components/a11y/SkipLink.tsx`) and integrate into `PublicLayout.tsx` and `AdminLayout.tsx`. (Refining Suggestion 7 & initial A11y work).
- [ ] **Thorough Keyboard Navigation Review:**
    *   Ensure all interactive elements (links, buttons, form fields, tabs, custom controls) are focusable and operable via keyboard.
    *   Logical focus order throughout the application.
    *   Implement custom keyboard handling for components like dropdowns, modals, or custom menus if needed (using `useKeyboardNavigation` hook from Suggestion 7 or similar).
- [ ] **ARIA Attributes Implementation:**
    *   Review all components and add appropriate ARIA roles, states, and properties (e.g., `aria-label`, `aria-labelledby`, `aria-describedby`, `aria-expanded`, `aria-current`, `aria-live` for dynamic content/notifications).
    *   Ensure form inputs have associated labels.
- [ ] **Focus Management & Visible Focus Styles:**
    *   Ensure clear and consistent visible focus indicators for all focusable elements (Tailwind's default focus rings are a good start, customize if needed for iOS style).
    *   Manage focus programmatically where necessary (e.g., when modals open, focus should move into the modal; on close, return to the trigger). (Use `useFocusTrap` hook from Suggestion 7).
- [ ] **Color Contrast Check:**
    *   Use browser developer tools or online contrast checkers to ensure text and UI elements meet WCAG AA contrast ratios (4.5:1 for normal text, 3:1 for large text/graphics). Adjust Tailwind theme colors if necessary.
- [ ] **Screen Reader Testing:**
    *   Manually test key user flows with a screen reader (e.g., NVDA on Windows, VoiceOver on macOS).
    *   Verify content is read out logically, interactive elements are announced correctly, and images have descriptive alt text.
- [ ] **Semantic HTML Review:** Ensure proper use of HTML5 semantic elements (`<nav>`, `<main>`, `<article>`, `<aside>`, `<section>`, `<footer>`, `<header>`).

### Performance Optimizations - Target Lighthouse Scores ≥ 90 (Performance & A11y)

- [ ] **Run Lighthouse Audits:**
    *   Generate Lighthouse reports for key pages (Home, Search, Career Detail).
    *   Identify primary bottlenecks and areas for improvement.
    *   `lighthouse http://localhost:3000 --output html --output-path ./lighthouse-reports/home.html --view` (after `npm run build && npx serve -s client/dist`)

- [ ] **Image Optimization (Asset Pipeline):**
    *   Create a script (`scripts/optimize-images.js` in root or `server/scripts/`) using `sharp` (install `npm install sharp`) to:
        *   Convert images in `client/public/images/` (or other source) to WebP format.
        *   Generate multiple responsive sizes (e.g., 320w, 640w, 1280w).
        *   Output optimized images to a specific directory (e.g., `client/public/images/optimized/`).
    *   This script is run manually by a developer as part of asset preparation, not an automated build step unless CI is set up.
- [ ] **Implement `ResponsiveImage.tsx` Component** (`client/src/components/common/ResponsiveImage.tsx`):
    *   Uses the `<picture>` element with `<source type="image/webp" srcSet="...">` for WebP and responsive sizes, and a fallback `<img>` tag.
    *   Integrate this component where static images are used (e.g., hero banner, "How it Works" icons).
    *   (Note: `OptimizedImage.tsx` from Sprint 6 focused on `loading="lazy"` and placeholders; `ResponsiveImage.tsx` focuses on format/resolution negotiation). These can be combined or used separately.

- [ ] **Font Optimization:**
    *   Ensure modern font formats (WOFF2) are used.
    *   Use `font-display: swap;` in CSS for faster perceived text rendering.
    *   Preload critical font files using `<link rel="preload">` in `client/public/index.html`. (As per Suggestion 7).

- [ ] **Verify Code Splitting & Lazy Loading:**
    *   Confirm `React.lazy` and `<Suspense>` are effectively splitting code for routes.
    *   Check Network tab in dev tools to see JS chunks loading on demand.
    *   Ensure `OptimizedImage` component is correctly lazy-loading images not in the initial viewport.

- [ ] **Review and Refine Caching Strategies:**
    *   **Client-Side:** Verify `apiCache.ts` (from Sprint 6) is working effectively for API GET requests.
    *   **Server-Side (API):** Consider adding `Cache-Control` headers for static API responses (e.g., list of all tags if they rarely change) if `apicache` middleware (Sprint 3 stretch) was implemented.

- [ ] **Bundle Analysis (Optional but Recommended):**
    *   Use a tool like `vite-bundle-visualizer` (`npm install -D rollup-plugin-visualizer` then configure in `vite.config.ts`) to inspect the production JavaScript bundle sizes and identify large dependencies or opportunities for further splitting.

- [ ] **Implement Resource Hints** (`client/public/index.html`):
    *   Add `preconnect` for critical third-party origins (e.g., API domain if different, font providers).
    *   Add `preload` for critical above-the-fold assets (main CSS, JS, hero image, key fonts).

### Documentation

- [ ] **Finalize `README.md` (Project Root):**
    *   Comprehensive setup instructions (covering all steps from clone to running dev servers, DB setup, migrations, seeding - from Suggestion 21).
    *   Project overview, tech stack, features.
    *   Directory structure explanation.
    *   Key NPM scripts (root, client, server).
    *   VS Code setup recommendations (workspace, extensions).
    *   Testing and debugging guidelines.
    *   Contribution guidelines (simple).
    *   License information.

- [ ] **Finalize `API.md` (Project Root or `server/docs/`):**
    *   Detailed documentation for all API endpoints (public and admin).
    *   Include request/response examples, query parameters, path parameters, status codes, and authentication requirements (even if placeholder auth for now).
    *   Update with any new endpoints from Suggestion 8 (feedback) or Suggestion 17 (slug history related, if API exposed).

- [ ] **Create `COMPONENTS.md` (Project Root or `client/docs/`):**
    *   Document key reusable React components (both public and admin).
    *   Props, usage examples, purpose. (e.g., CareerCard, FilterPanel, FileDropzone, AdminLayout, PublicLayout).

- [ ] **Create `CHANGELOG.md` (Project Root):**
    *   Initialize with a `[1.0.0] - YYYY-MM-DD` entry detailing features of the first release.
    *   Follow "Keep a Changelog" format.

### Release Preparation

- [ ] **Final End-to-End Testing:**
    *   Perform thorough testing of all user flows (public and admin) on different browsers and devices.
    *   Verify all implemented suggestions are working as expected.

- [ ] **Create/Finalize Environment Configuration Examples:**
    *   `server/.env.example` (local development)
    *   `server/.env.production.example` (for production deployment)
    *   `client/.env.example` (if client uses runtime env vars beyond VITE_ prefix for build time)
    *   Ensure all necessary variables are documented.

- [ ] **Create Basic Deployment Scripts (`scripts/deploy/deploy.sh` at project root):**
    *   A shell script that automates (or outlines steps for) building client and server, and deploying them to a target environment (e.g., using `rsync` or SCP for simple deployments, or interacting with a PaaS CLI).
    *   Should be parameterizable for different environments (staging, production). (As outlined in Suggestion 7's polish part).

- [ ] **Git Tagging:**
    *   Once all Polish tasks are complete and testing is successful, create a Git tag for version 1.0.0:
      ```bash
      git tag -a v1.0.0 -m "Version 1.0.0 - Initial Release"
      git push origin v1.0.0
      ```

## 4. Creative Add-Ons (Stretch Goals for this Sprint or Future)

- [ ] **Dark Mode Support (Client):** Implement a theme toggle using `useDarkMode` hook and Tailwind's `dark:` variants. (From Suggestion 7).
- [ ] **Keyboard Shortcuts (Client):** Implement `useKeyboardShortcuts` for common actions (e.g., `/` to focus search, `n` for new upload). Display shortcuts in a help modal. (From Suggestion 7).
- [ ] **Animated Page Transitions (Client):** Use `framer-motion` or similar for smooth transitions between routes. (From Suggestion 7).
- [ ] **Basic Offline Support (Client - PWA elements):** Implement a simple service worker to cache static assets and key API GET requests for basic offline viewing or a custom offline page. (From Suggestion 7).
- [ ] **Automated A11y Tests in CI:** Integrate `axe-core` with Jest/Playwright for automated accessibility checks in a CI pipeline (future).

## 5. Run & Verify

### Tools & Setup

1.  **Accessibility Tools:**
    *   `react-axe` configured in `main.tsx` (check browser console during development).
    *   Browser extensions: axe DevTools, Lighthouse, WAVE.
    *   Screen readers: NVDA (Windows), VoiceOver (macOS).
2.  **Performance Tools:**
    *   Lighthouse (CLI or browser DevTools).
    *   Browser Network and Performance tabs.
    *   `vite-bundle-visualizer` (if configured).
3.  **Deployment Script:**
    *   Test the `deploy.sh` script against a mock/staging environment if possible.

### Verification Steps

1.  **Accessibility Audit:**
    *   Run `react-axe` during development and address reported issues.
    *   Perform full site audit with axe DevTools/Lighthouse on key pages.
    *   Manually test keyboard navigation for all interactive elements.
    *   Manually test with a screen reader for key user flows.
    *   Verify color contrast ratios meet WCAG AA.
2.  **Performance Audit:**
    *   Build client for production: `cd client && npm run build`.
    *   Serve the production build: `npx serve -s client/dist -p 5000` (or similar).
    *   Run Lighthouse audits against `http://localhost:5000` for Home, Search, and a Detail page.
    *   Analyze reports and implement further optimizations if scores are below 90. Check First Contentful Paint (FCP), Largest Contentful Paint (LCP), Time to Interactive (TTI), Cumulative Layout Shift (CLS).
3.  **Documentation Review:**
    *   Read through `README.md`, `API.md`, `COMPONENTS.md`, `CHANGELOG.md`.
    *   Verify accuracy, completeness, and clarity.
    *   Ask a colleague (or try as a "new developer") to follow the `README.md` setup instructions.
4.  **Release Script & Config Test:**
    *   Review `.env.production.example` for completeness.
    *   Dry-run or test parts of the `deploy.sh` script if feasible (e.g., build steps).

### Success Criteria

- ✅ **Lighthouse Accessibility Score ≥ 90** for key public pages (Home, Search, Detail).
- ✅ **Lighthouse Performance Score ≥ 90** for key public pages.
- ✅ Application is fully navigable and operable using only a keyboard.
- ✅ Key user flows are understandable and operable with a screen reader.
- ✅ All interactive elements have clear, visible focus states.
- ✅ Text and UI elements meet WCAG AA color contrast requirements.
- ✅ Image optimization (WebP, responsive sizes, lazy loading) is implemented and effective.
- ✅ Font loading is optimized.
- ✅ Resource hints are in place for critical assets.
- ✅ `README.md` provides comprehensive project setup and operational guidance.
- ✅ `API.md` accurately documents all API endpoints.
- ✅ `COMPONENTS.md` documents key reusable components.
- ✅ `CHANGELOG.md` is initiated for v1.0.0.
- ✅ Example environment configurations (`.env.*.example`) are complete.
- ✅ Basic deployment script is created and functional (or clearly outlines manual steps).
- ✅ Git tag `v1.0.0` is ready to be applied upon successful completion.

## 6. Artifacts to Commit

- **Branch name**: `sprint-7-polish`
- **Mandatory files**:
    - Updated `client/src/main.tsx` with `react-axe`.
    - `client/src/components/a11y/SkipLink.tsx` (and other A11y-specific components/hooks).
    - All client components updated with ARIA attributes and A11y improvements.
    - `scripts/optimize-images.js` (or similar image processing script).
    - `client/src/components/common/ResponsiveImage.tsx` (and updates to use it).
    - Updated `client/public/index.html` with font links and resource hints.
    - Updated `client/tailwind.config.js` for font families or any theme adjustments.
    - `README.md` (final version).
    - `API.md` (final version).
    - `COMPONENTS.md`.
    - `CHANGELOG.md`.
    - `scripts/deploy/deploy.sh`.
    - `.env.example`, `.env.production.example`, `.env.staging.example` files for `server/` and `client/` (if applicable).
- **PR Checklist**:
    - [ ] Accessibility audit (manual and tool-assisted) completed and issues addressed.
    - [ ] Performance audit (Lighthouse) completed, and target scores met.
    - [ ] All documentation (`README`, `API`, `COMPONENTS`, `CHANGELOG`) is complete and reviewed.
    - [ ] Release preparation tasks (env examples, deployment script basics) are done.
    - [ ] Final end-to-end testing confirms application stability and functionality.
    - [ ] Code is thoroughly reviewed and cleaned up.