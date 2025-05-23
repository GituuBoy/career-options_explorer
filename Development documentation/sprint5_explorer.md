# Sprint 5: Explorer

## 1. Narrative Goal

This "Explorer" sprint brings the Career Options Explorer to life for the public! Our mission is to craft a beautiful, intuitive, and mobile-first user interface that empowers users to discover and delve into various career paths. By the end of these six days, we will deliver:
1.  A welcoming **Home Page** featuring a prominent search bar, curated featured careers, and quick links to popular categories or recently viewed items.
2.  A dynamic **Search Results Page** where users can find careers based on keywords or tags, with clear presentation of results.
3.  A rich **Career Detail Page** that comprehensively displays all parsed information for a selected career, including its overview, detailed sections, skills, salary (if available), and related career paths, along with sharing and feedback options.
We will focus on clean design adhering to iOS aesthetics, seamless navigation, dynamic meta tags for SEO, and ensuring all content is presented clearly and engagingly. This sprint is about transforming our structured data into a valuable, accessible resource for anyone exploring their career options.

## 2. Back-of-the-Envelope Design (Public UI)

```mermaid
graph TD
    %% --- Overall Structure ---
    AppRouter[App Router (React Router DOM)] --> PublicLayoutRoute{Route: /}
    PublicLayoutRoute --> PublicLayoutComp[PublicLayout Component]
    PublicLayoutComp --> Header[Shared Header (Logo, Nav: Home, Explore, Admin)]
    PublicLayoutComp --> MainOutlet[Main Content (Outlet for Pages)]
    PublicLayoutComp --> Footer[Shared Footer (Copyright, Links: About, FAQ)]
    PublicLayoutComp --> SkipLink[Accessibility: Skip to Content]

    %% --- Home Page ---
    MainOutlet -- Route: / --> HomePageComp[HomePage]
        HomePageComp --> HeroBanner[Hero Banner (Title, Subtitle, SearchBar)]
        HomePageComp --> FeaturedSection[Featured Careers Section]
        FeaturedSection --> CareerCardComp1[CareerCard Component]
        HomePageComp --> PopularCategories[Popular Categories/Tags Section]
        HomePageComp --> RecentlyViewed[Recently Viewed Careers Section (localStorage)]
        HomePageComp --> HowItWorks[How It Works / Value Prop Section]
        
    %% --- Search Results Page ---
    MainOutlet -- Route: /search --> SearchPageComp[SearchPage]
        SearchPageComp --> Breadcrumbs1[Breadcrumbs Component]
        SearchPageComp --> SearchBarComp2[SearchBar Component (pre-filled from query)]
        SearchPageComp --> FilterPanelPlaceholder[FilterPanel Area (Full impl. Sprint 6)]
        SearchPageComp --> SearchResultsList[SearchResults List]
        SearchResultsList --> CareerCardComp2[CareerCard Component]
        SearchPageComp --> PaginationPlaceholder[Pagination Controls (Basic impl. now, enhanced Sprint 6)]
        
    %% --- Career Detail Page ---
    MainOutlet -- Route: /careers/:slug --> DetailPageComp[CareerDetailPage]
        DetailPageComp --> HelmetComp[Helmet (for Dynamic Meta Tags)]
        DetailPageComp --> Breadcrumbs2[Breadcrumbs Component]
        DetailPageComp --> CareerHeaderSection[CareerHeader (Title, Snapshot, Core Tags)]
        DetailPageComp --> InPageNav[In-Page Navigation (Tabs or Sticky Nav for sections - Suggestion 2)]
        InPageNav --> CareerSectionDisplay[Dynamic Section Display Area]
        DetailPageComp --> CareerSectionsContent[Rendered Career Sections]
        DetailPageComp --> RelatedCareersSection[Related Careers Section]
        RelatedCareersSection --> CareerCardComp3[CareerCard Component]
        DetailPageComp --> ShareWidgetComp[ShareCareerWidget (Copy Link, Social)]
        DetailPageComp --> FeedbackWidgetComp[FeedbackWidget (Helpful? Yes/No)]
        
    %% --- Static Info Pages ---
    MainOutlet -- Route: /about --> AboutPageComp[AboutPage]
    MainOutlet -- Route: /faq --> FAQPageComp[FAQPage]
    MainOutlet -- Route: /contact --> ContactPageComp[ContactPage]

    %% --- API Interactions ---
    subgraph "Client-API Interactions"
        HomePageComp --> API_FeaturedCareers["GET /api/v1/careers?limit=X&sort=recent_or_curated"]
        HomePageComp --> API_PopularTags["GET /api/v1/tags/popular"]
        SearchBarComp1 --> API_Search["GET /api/v1/careers/search?q={query}"]
        SearchBarComp2 --> API_Search
        SearchPageComp --> API_Search
        DetailPageComp --> API_CareerDetail["GET /api/v1/careers/:slug"]
        DetailPageComp --> API_RelatedCareers["GET /api/v1/careers/search?tag={topTagId}&limit=3"]
        FeedbackWidgetComp --> API_SubmitFeedback["POST /api/v1/feedback"]
    end
```

## 3. Task Checklist

### Client-Side Setup & Core Public Components (Extending Sprint 0 & 4)

- [ ] **Install Additional Client-Side Dependencies** (in `client/` directory):
  ```bash
  npm install react-helmet-async # For dynamic meta tags
  # Consider a lightweight animation library like 'react-spring' or 'framer-motion' for stretch goals
  # npm install lucide-react (or other icon library)
  ```

- [ ] **Create `PublicLayout.tsx` Component** (`client/src/components/public/PublicLayout.tsx`):
    *   Shared header with logo, navigation (Home, Explore, (Admin - if user is admin or dev mode)).
    *   `<Outlet />` for page content.
    *   Shared footer (copyright, links to About, FAQ, Contact).
    *   Integrate `SkipLink.tsx` (from Sprint 7 a11y prep / Suggestion 7).
    *   Styled with TailwindCSS, iOS aesthetic.

- [ ] **Create Reusable `Breadcrumbs.tsx` Component** (`client/src/components/common/Breadcrumbs.tsx` - Suggestion 7):
    *   Takes an array of `{ label: string, path?: string }`.
    *   Renders linked breadcrumb trail.
    *   (Stretch) Generates JSON-LD structured data.

- [ ] **Create `SearchBar.tsx` Component** (`client/src/components/public/SearchBar.tsx`):
    *   Input field and submit button.
    *   On submit, navigates to `/search?q={query}`.
    *   Should be reusable on Home page and Search page.

- [ ] **Create `CareerCard.tsx` Component** (`client/src/components/public/CareerCard.tsx`):
    *   Displays career `title`, `snapshot`, and a few key `tags`.
    *   Links to the `CareerDetailPage` using career `slug`.
    *   Styled for a clean, engaging look.

- [ ] **Create `ShareWidget.tsx` Component** (`client/src/components/public/ShareWidget.tsx` - Suggestion 10):
    *   Buttons/icons for "Copy Link", Email, LinkedIn, Twitter, Facebook.
    *   Uses `navigator.clipboard` and constructs share URLs.

- [ ] **Create `FeedbackWidget.tsx` Component** (`client/src/components/public/FeedbackWidget.tsx` - Suggestion 8):
    *   "Was this page helpful? [Yes] [No]" buttons.
    *   (Optional) Text area for comments if "No".
    *   Submits feedback to `POST /api/v1/feedback`.

### Home Page (`client/src/pages/public/HomePage.tsx`)

- [ ] **Implement Hero Banner Section:**
    *   Large title, subtitle/tagline.
    *   Prominent `SearchBar` component.
    *   Visually appealing background (iOS style).

- [ ] **Implement "Featured Careers" Section:**
    *   Fetch a curated list or N most recent/popular careers from `GET /api/v1/careers` (e.g., `?limit=6&sort=updated_at_desc` or a dedicated `?featured=true` if API supports).
    *   Display using a grid of `CareerCard` components.
    *   Link to "View All Careers" (navigates to `/search`).

- [ ] **Implement "Popular Categories/Tags" Section:**
    *   Fetch popular tags/categories from `GET /api/v1/tags/popular`.
    *   Display as clickable links/pills that navigate to `/search?category={name}` or `/search?tag={id}`.

- [ ] **Implement "Recently Viewed Careers" Section** (Suggestion 4):
    *   Fetch from `localStorage`.
    *   Display as a list/grid of small cards or links.
    *   If empty, show an appropriate message.

- [ ] **(Stretch) Implement "How It Works" or Value Proposition Section:**
    *   Simple graphic/text explaining how to use the explorer.

### Search Results Page (`client/src/pages/public/SearchPage.tsx`)

- [ ] **Fetch and Display Search Results:**
    *   Get search query `q` and other filter params (e.g., `tag`, `category`) from URL using `useSearchParams`.
    *   Fetch careers from `GET /api/v1/careers/search` with query parameters.
    *   Display results as a grid of `CareerCard` components.
    *   Handle loading, error, and "no results found" states.

- [ ] **Integrate `SearchBar`:**
    *   Pre-fill with the current search query.
    *   Submitting updates the URL and re-fetches results.

- [ ] **Display `Breadcrumbs`**.

- [ ] **Basic Filter Placeholder:**
    *   A simple UI area where more advanced filters will go in Sprint 6. For now, it might just show active query/tag.

- [ ] **Basic Pagination:**
    *   If results exceed a page limit, display simple "Next"/"Previous" buttons or page numbers. Full pagination component in Sprint 6.
    *   API should support `page` and `limit` query params.

### Career Detail Page (`client/src/pages/public/CareerDetailPage.tsx`)

- [ ] **Fetch and Display Career Details:**
    *   Get career `slug` from URL params (`useParams`).
    *   Fetch full career data (including sections and all tags) from `GET /api/v1/careers/:slug`.
    *   Handle loading, error, and "career not found" states.

- [ ] **Dynamic Meta Tags & Title using `react-helmet-async`** (Suggestion 12):
    *   Set page `<title>`, meta description, canonical URL, OG tags, and Twitter Card tags dynamically based on career data.
    *   Wrap `App.tsx` with `<HelmetProvider>`.

- [ ] **Display `Breadcrumbs`**.

- [ ] **Implement Career Header Section:**
    *   Prominent career `title`.
    *   Display `snapshot`.
    *   Display key tags (e.g., top 5-7 most relevant).

- [ ] **Implement In-Page Section Navigation** (Suggestion 2):
    *   Tabs or a sticky sub-navigation bar linking to different career `sections` (Overview, Skills, Education, etc.).
    *   Clicking a nav item scrolls to or displays the relevant section content.
    *   Clearly indicate the active section.

- [ ] **Display Career Sections:**
    *   Iterate through `career.sections` and render `title` and `content` for each.
    *   Format content appropriately (e.g., handling newlines in `content`).

- [ ] **Implement "Related Careers" Section:**
    *   Based on common tags or a predefined relationship (if data supports). Fetch a few related careers using the API (e.g., search by top tag of current career, excluding current career).
    *   Display using `CareerCard` components.

- [ ] **Integrate `ShareWidget.tsx`**.

- [ ] **Integrate `FeedbackWidget.tsx`**.

### Static Informational Pages (Suggestion 20)

- [ ] **Create `AboutPage.tsx`** (`client/src/pages/public/AboutPage.tsx`):
    *   Static content: mission, data sources.
- [ ] **Create `FAQPage.tsx`** (`client/src/pages/public/FAQPage.tsx`):
    *   Static content: common questions and answers.
- [ ] **Create `ContactPage.tsx`** (`client/src/pages/public/ContactPage.tsx`):
    *   Static content: `mailto:` link or placeholder for a contact form.

### Routing Updates (`client/src/App.tsx`)

- [ ] **Add Routes for Public Pages:**
    *   `/` -> `HomePage`
    *   `/search` -> `SearchPage`
    *   `/careers/:slug` -> `CareerDetailPage`
    *   `/about` -> `AboutPage`
    *   `/faq` -> `FAQPage`
    *   `/contact` -> `ContactPage`
    *   Ensure these routes use the `PublicLayout`.

## 4. Creative Add-Ons (Stretch)

- [ ] **"Career Path Visualization" on Detail Page:** Based on "Career Path & Growth" section data, render a simple visual flow of typical progression. (Refined from Suggestion 9).
- [ ] **Pastel Gradient Hero with Gentle Scroll-Parallax:** Implement the visual flourish for the home page hero banner.
- [ ] **Skeleton Loading Screens:** Instead of simple "Loading..." text, use skeleton UI placeholders for `CareerCard`s and `CareerDetailPage` content sections for a smoother perceived loading experience.
- [ ] **Enhanced "No Results" on Search Page:** Suggest alternative search terms or popular categories if a search yields no results.

## 5. Run & Verify

### Setup & Start

1.  Install `react-helmet-async`: `cd client && npm install react-helmet-async`.
2.  Ensure backend API server (Sprint 3) is running.
3.  Start the client development server: `cd client && npm run dev`.

### Manual Browser Checks & User Flows

1.  **Home Page (`/`):**
    *   Verify hero banner, search bar, featured careers, popular categories, and recently viewed (after viewing some careers) display correctly.
    *   Test search bar navigation.
    *   Click on featured careers and category links.
2.  **Search Results Page (`/search`, `/search?q=...`, `/search?tag=...`):**
    *   Perform various searches. Verify results are relevant and displayed using `CareerCard`s.
    *   Check "No results found" state.
    *   Test basic pagination if implemented.
    *   Verify breadcrumbs are correct.
3.  **Career Detail Page (`/careers/:slug`):**
    *   Navigate from Home or Search page.
    *   Verify all career information (title, snapshot, overview, sections, tags) is displayed.
    *   Test in-page section navigation (tabs/sticky nav).
    *   Check "Related Careers" section.
    *   Test "Share" functionality (copy link, social links).
    *   Test "Feedback" widget submission (check network tab for API call).
    *   View page source or use browser dev tools to verify dynamic meta tags (title, description, OG, Twitter).
    *   Verify breadcrumbs are correct.
4.  **Static Pages (`/about`, `/faq`, `/contact`):**
    *   Verify pages load with their static content.
    *   Check footer links.
5.  **Responsiveness:**
    *   Test all public pages on various screen sizes (desktop, tablet, mobile) using browser developer tools. Ensure layout is adaptive and usable.
6.  **Navigation:**
    *   Ensure all links (header, footer, internal content links) work correctly.
    *   Test browser back/forward buttons.

### Success Criteria

- ✅ Home Page is engaging and provides multiple entry points for exploration.
- ✅ Search Results Page correctly displays careers based on queries/filters.
- ✅ Career Detail Page comprehensively presents all career data with good readability and structure.
- ✅ In-page navigation on Career Detail Page enhances UX for long content.
- ✅ Dynamic SEO meta tags are correctly generated for career detail pages.
- ✅ Share and Feedback widgets are functional.
- ✅ Recently Viewed careers feature works using localStorage.
- ✅ Static informational pages (About, FAQ, Contact) are accessible.
- ✅ All public pages are responsive and adhere to the iOS-inspired design.
- ✅ Navigation is smooth and intuitive across the public site.
- ✅ Basic accessibility considerations (semantic HTML, keyboard navigability for new components) are in place.

## 6. Artifacts to Commit

- **Branch name**: `sprint-5-explorer`
- **Mandatory files**:
    - `client/src/pages/public/` (HomePage, SearchPage, CareerDetailPage, AboutPage, FAQPage, ContactPage).
    - `client/src/components/public/` (PublicLayout, SearchBar, CareerCard, ShareWidget, FeedbackWidget).
    - `client/src/components/common/Breadcrumbs.tsx`.
    - Updates to `client/src/App.tsx` for public routing.
    - Updates to `client/src/types/career.ts` if needed.
    - `client/src/config.ts` (if not already fully set up).
    - Updated `client/package.json` and `client/package-lock.json`.
- **PR Checklist**:
    - [ ] All specified public pages and components are implemented.
    - [ ] Core user flows (Home -> Search -> Detail) are working smoothly.
    - [ ] Data is correctly fetched from the API and displayed.
    - [ ] Dynamic meta tags are implemented for career detail pages.
    - [ ] Share and Feedback functionalities are integrated.
    - [ ] UI is responsive and follows design guidelines.
    - [ ] Basic accessibility is considered in new components.
    - [ ] Code is well-commented.