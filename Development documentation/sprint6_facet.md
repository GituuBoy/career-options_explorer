# Sprint 6: Facet & Performance

## 1. Narrative Goal

In this "Facet & Performance" sprint, we're elevating the Career Options Explorer by implementing a sophisticated faceted search system and introducing key performance optimizations. By the end of these three days, users will be able to refine their career searches using multiple criteria (like skills, industries, education levels) simultaneously via an intuitive filter panel. We'll also enhance the search results page with robust pagination and a page size selector. On the performance front, we'll implement client-side API response caching, route-based code splitting, and lazy loading for images to ensure the application is fast, responsive, and can gracefully handle a growing dataset. This sprint is about empowering users with precise search tools and delivering a snappy, smooth browsing experience.

## 2. Back-of-the-Envelope Design

```mermaid
graph TD
    subgraph "Client-Side Search & Filter UI (React)"
        SearchPageComp[SearchPage Component]
        SearchPageComp --> SearchBarComp[SearchBar]
        SearchPageComp --> FilterPanelComp[FilterPanel Component]
        SearchPageComp --> SearchResultsArea[SearchResults Area]
        SearchPageComp --> PaginationComp[Pagination Component]
        SearchPageComp --> PageSizeSelectorComp[PageSizeSelector Component]

        FilterPanelComp --> FilterGroupComp[FilterGroup (e.g., Category, Skill)]
        FilterGroupComp --> FilterOptionItem[FilterOption (Checkbox + Count)]
        FilterPanelComp --> ActiveFiltersDisplay[Display Active Filters]
        FilterPanelComp --> ClearAllFiltersBtn[Clear All Filters Button]
        
        SearchResultsArea --> CareerCardComp[CareerCard]
    end

    subgraph "URL State Management for Filters & Pagination"
        FilterPanelComp -- User Interaction --> UpdateURLParams1[Updates URL Search Params (e.g., ?category=tech&skill=123)]
        PaginationComp -- User Interaction --> UpdateURLParams2[Updates URL Search Params (e.g., ?page=2)]
        PageSizeSelectorComp -- User Interaction --> UpdateURLParams3[Updates URL Search Params (e.g., ?limit=24)]
        
        UpdateURLParams1 --> TriggerAPIFetch
        UpdateURLParams2 --> TriggerAPIFetch
        UpdateURLParams3 --> TriggerAPIFetch
    end

    subgraph "API Interaction & Data Flow for Search"
        TriggerAPIFetch[SearchPage useEffect (watches URL params)] --> API_SearchCall["GET /api/v1/careers/search (with all filters, q, page, limit)"]
        API_SearchCall --> APIServer[Express API Server]
        APIServer --> DBQuery[Database Query (complex filtering & faceting)]
        DBQuery --> APIResponseData[API Response: { careers: [], pagination: {}, facets: {} }]
        APIResponseData --> UpdateUIState[SearchPage: Updates Careers, Pagination, FilterPanel Counts]
    end
    
    subgraph "Client-Side Performance Optimizations"
        ApiClientUtil[apiClient.ts with Caching] --> |cachedFetch| API_SearchCall
        AppRouter[App Router with React.lazy] --> LazyLoadedPages[Lazy-Loaded Page Components]
        CareerCardComp --> OptimizedImageComp[OptimizedImage Component (lazy loading)]
    end
```

## 3. Task Checklist

### Faceted Search UI Implementation (Client-Side)

- [ ] **Create `FilterPanel.tsx` Component** (`client/src/components/public/FilterPanel.tsx`):
    *   Accepts `filterGroups: FilterGroup[]` prop (where `FilterGroup` includes `id`, `name`, `options: {id, name, count}[]`, `isExpanded`).
    *   Manages its own expanded/collapsed state for each filter group.
    *   Reads initial active filters from URL search parameters (`useSearchParams`).
    *   On filter selection (checkbox toggle):
        *   Updates URL search parameters (e.g., `?category=Tech,Finance&skill=Python`).
        *   This URL change will trigger data re-fetch in `SearchPage.tsx`.
    *   Displays count next to each filter option (updated based on API response).
    *   Includes a section to display currently active filters with options to remove individual filters.
    *   Includes a "Clear All Filters" button.
    *   Styled for mobile-first (collapsible on mobile, sidebar on desktop).

- [ ] **Create `Pagination.tsx` Component** (`client/src/components/public/Pagination.tsx`):
    *   Accepts `currentPage`, `totalPages` props.
    *   Generates page number links, "Previous," "Next" buttons.
    *   Intelligently displays a limited set of page numbers with ellipses for large `totalPages`.
    *   On page selection, updates `page` URL search parameter.

- [ ] **Create `PageSizeSelector.tsx` Component** (`client/src/components/public/PageSizeSelector.tsx`):
    *   Dropdown to select number of results per page (e.g., 12, 24, 48).
    *   Accepts `options: number[]` and `currentPageSize: number` props.
    *   On selection, updates `limit` URL search parameter and resets `page` to 1.

- [ ] **Update `SearchPage.tsx`** (`client/src/pages/public/SearchPage.tsx`):
    *   **Layout:** Integrate `FilterPanel` (e.g., in a sidebar on desktop, collapsible on mobile) alongside search results.
    *   **State:** Manage `careers`, `isLoading`, `error`, `paginationInfo` (currentPage, totalPages, totalCareers), `filterGroupsData`.
    *   **Data Fetching:**
        *   `useEffect` hook that listens to changes in `searchParams` (from `useSearchParams`).
        *   Construct API request to `GET /api/v1/careers/search` including all active filters from `searchParams`, search query `q`, `page`, and `limit`.
        *   The API response should include `careers`, `pagination` metadata, and `facets` data (counts for each filter option based on the current search/filter context).
        *   Update `careers`, `paginationInfo`, and pass `facets` data to `FilterPanel` to update option counts.
    *   Integrate `Pagination` and `PageSizeSelector` components, passing necessary props and handling their state/URL updates.

### API Enhancements for Faceted Search (Backend - Server-Side)

- [ ] **Modify `GET /api/v1/careers/search` Endpoint** (`server/src/api/controllers/careerController.ts`):
    *   Accept multiple filter parameters from query string (e.g., `category=Tech,Finance`, `skill=Python,Java`, `education=Bachelor`).
    *   Update repository method (`careerRepository.search` or a new `findByFilters`) to handle complex filtering logic in SQL.
    *   **Crucially:** The API response for this endpoint must now also include `facets` data. This object should contain the counts for each available filter option *within the context of the current search query and other applied filters*.
        *   Example `facets` structure in response:
          ```json
          "facets": {
            "category": [ { "id": "Tech", "name": "Technology", "count": 15 }, /* ... */ ],
            "skill": [ { "id": "Python", "name": "Python", "count": 10 }, /* ... */ ],
            "educationLevel": [ { "id": "Bachelor", "name": "Bachelor's Degree", "count": 20 }, /* ... */ ]
          }
          ```
- [ ] **Update `PostgresCareerRepository.ts`:**
    *   Implement/Refine `search` or create `findByFilters` method to build dynamic SQL queries based on multiple filter criteria.
    *   Implement methods to calculate facet counts efficiently alongside the main query (e.g., using window functions, conditional aggregation, or separate aggregate queries within the same transaction).

- [ ] **(Optional) Create a dedicated `GET /api/v1/careers/facets` endpoint:**
    *   If fetching initial global facet counts (before any search/filter is applied) is too slow with the main search, this endpoint could provide all available filter options and their total counts across all *published* careers. `FilterPanel` could use this for initial population.

### Client-Side Performance Optimizations

- [ ] **Implement Client-Side API Response Caching (`client/src/utils/apiCache.ts` & `apiClient.ts`):**
    *   Create a simple in-memory cache utility (`apiCache.ts`) with configurable expiry.
    *   Create an `apiClient.ts` that wraps `fetch` and uses `apiCache` for GET requests (especially for `/careers/search`, `/careers/:slug`, `/tags/popular`).
    *   Cache keys should be based on the full URL including query parameters.
    *   Ensure cache can be invalidated (e.g., after admin actions, or via a timer).

- [ ] **Implement Route-Based Code Splitting (`client/src/App.tsx`):**
    *   Use `React.lazy` and `<Suspense>` to lazy-load page components (`HomePage`, `SearchPage`, `CareerDetailPage`, admin pages).
    *   Provide a suitable fallback (e.g., a simple spinner) for `<Suspense>`.

- [ ] **Implement Image Lazy Loading & Optimization (`client/src/components/common/OptimizedImage.tsx`):**
    *   Create an `OptimizedImage` component that:
        *   Uses the `loading="lazy"` attribute for native browser lazy loading.
        *   (Optional) Implements a placeholder (e.g., blurred low-quality image or solid color) while the full image loads.
        *   (Stretch for Sprint 7) Can be extended to use `<picture>` element with multiple sources for responsive images and WebP format.
    *   Replace standard `<img>` tags for career-related images (e.g., in `CareerCard`, potentially in `CareerDetailPage` if images are used) with `OptimizedImage`.

## 4. Creative Add-Ons (Stretch)

- [ ] **Debounced API Calls for Filters:** When a user rapidly clicks multiple filter checkboxes, debounce the API call to avoid excessive requests, only fetching data after a short pause in user activity.
- [ ] **"Sticky" Filter Panel on Desktop:** Make the filter panel stick to the side of the screen as the user scrolls through search results on larger screens.
- [ ] **Visual Feedback for Loading Facet Counts:** Show spinners or placeholders within the `FilterPanel` while facet counts are being updated after a search.
- [ ] **Advanced Pagination:** Implement more sophisticated pagination UI, e.g., input field to jump to a specific page.
- [ ] **Performance Profiling:** Use React DevTools Profiler and browser performance tools to identify and address any client-side rendering bottlenecks.

## 5. Run & Verify

### Setup & Start

1.  Ensure backend API server (Sprint 3, with facet API enhancements) is running.
2.  Start the client development server: `cd client && npm run dev`.

### Manual Browser Checks & User Flows

1.  **Faceted Search (`/search` page):**
    *   Verify `FilterPanel` loads with initial global facet counts (if `/careers/facets` endpoint is used) or updates after first search.
    *   Select various filter options from different groups (e.g., Category: "Technology", Skill: "Python").
        *   Confirm URL updates correctly with selected filter parameters.
        *   Confirm search results update to reflect the applied filters.
        *   Confirm facet counts in the `FilterPanel` update to reflect the narrowed-down result set.
    *   Test "Clear All Filters" button.
    *   Test removing individual active filters.
    *   Verify behavior with search query `q` in combination with facets.
2.  **Pagination & Page Size:**
    *   On the `/search` page with enough results, test the `Pagination` component. Navigate to different pages and verify URL updates and correct data is shown.
    *   Test the `PageSizeSelector`. Change the number of items per page and verify results and pagination update accordingly.
3.  **Performance Optimizations:**
    *   **API Caching:** Open browser dev tools (Network tab). Perform a search. Perform the same search again (or navigate away and back) – check if subsequent requests for the same data are served from cache (e.g., quick response, or no network request if fully client-cached).
    *   **Code Splitting:** Monitor Network tab when first navigating to different main pages (Home, Search, Detail, Admin pages). You should see JavaScript chunks being loaded on demand.
    *   **Image Lazy Loading:** On pages with multiple `CareerCard`s, scroll down and observe in the Network tab that images further down the page are only loaded as they enter the viewport.
4.  **Responsiveness:**
    *   Thoroughly test the `FilterPanel` and search results layout on mobile, tablet, and desktop screen sizes. Ensure filters are easily accessible and usable on mobile.

### Success Criteria

- ✅ `FilterPanel` allows users to select and deselect multiple facet filters, updating URL parameters and search results.
- ✅ Facet counts in `FilterPanel` dynamically update based on the current search/filter context.
- ✅ `Pagination` component allows navigation through paged search results.
- ✅ `PageSizeSelector` allows users to change the number of results per page.
- ✅ API responses for search include facet data.
- ✅ Client-side API caching is implemented and reduces redundant network requests for repeated queries.
- ✅ Route-based code splitting is implemented, improving initial load time.
- ✅ Images on pages like Search Results (CareerCards) use lazy loading.
- ✅ The search interface remains responsive and user-friendly across devices.
- ✅ Lighthouse scores for performance and accessibility show improvement or remain high. (Target > 80-85 for perf in this sprint, polish to >90 in Sprint 7).

## 6. Artifacts to Commit

- **Branch name**: `sprint-6-facet`
- **Mandatory files**:
    - `client/src/components/public/FilterPanel.tsx`
    - `client/src/components/public/Pagination.tsx`
    - `client/src/components/public/PageSizeSelector.tsx`
    - `client/src/utils/apiCache.ts` (if not already from a previous suggestion)
    - `client/src/utils/apiClient.ts` (if not already from a previous suggestion)
    - `client/src/components/common/OptimizedImage.tsx`
    - Updated `client/src/pages/public/SearchPage.tsx` to integrate new components and logic.
    - Updated `client/src/App.tsx` for `React.lazy` and `Suspense`.
    - Backend:
        - Updated `server/src/api/controllers/careerController.ts` to support faceted search and return facet counts.
        - Updated `server/src/repositories/implementations/PostgresCareerRepository.ts` with new filtering and faceting query logic.
- **PR Checklist**:
    - [ ] Faceted search filters are implemented and correctly update results and URL.
    - [ ] Facet counts are displayed and update dynamically.
    - [ ] Pagination and page size selection are functional.
    - [ ] Client-side API caching is working.
    - [ ] Code splitting for routes is implemented.
    - [ ] Image lazy loading is implemented for relevant images.
    - [ ] UI remains responsive and usable.
    - [ ] Backend API supports faceted search queries and returns facet data.
    - [ ] Code is well-commented, especially complex filtering or state logic.