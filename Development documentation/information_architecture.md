# Information Architecture

This document contains the key architectural diagrams for the Career Options Explorer application.

## 1. Site-Map Diagram

graph TD
    Home[Home Page] --> Search[Search Results]
    Home --> Categories[Categories Page]
    Home --> CareerDetail[/careers/:slug]
    Home --> About[About Page]
    Home --> FAQ[FAQ Page]
    Home --> Contact[Contact Page]

    Categories --> Search
    Search --> CareerDetail
    CareerDetail --> Search

    subgraph Admin
        direction LR
        AdminLogin[Admin Login] --> AdminDashboard[Admin Dashboard /admin]
        AdminDashboard --> UploadPage[Upload & Preview /admin/upload]
        AdminDashboard --> CareerMgmt[Manage Careers /admin/careers]
        AdminDashboard --> ParsingIssues[Parsing Issues /admin/parsing-issues]
        AdminDashboard --> FeedbackReview[Feedback Review /admin/feedback]

        CareerMgmt --> CareerEditor[Edit Career /admin/careers/edit/:id]
        ParsingIssues --> CareerEditor
    end

## 2. Data-Flow Diagram

graph LR
    User[User] --> React[React Frontend]
    AdminUser[Admin User] --> ReactAdmin[React Admin UI]

    React --> API[Express API]
    ReactAdmin --> API

    API --> DB[(PostgreSQL)]
    DB --> API

    subgraph "Content Ingestion"
      direction LR
      DOCX[DOCX Files] --> Parser[DOCX Parser Module]
      Parser --> StructuredJSON[Structured JSON]
      StructuredJSON --> ImportService[Import Service]
      ImportService --> DB
    end

    subgraph "Parser Pipeline"
        direction TB
        DOCX1[DOCX File] --> Mammoth[Mammoth.js]
        Mammoth --> HTML[HTML Stream]
        HTML --> Cheerio[Cheerio Walker]
        Cheerio --> RawJSON[Raw JSON Structure]
        RawJSON --> QualityScore[Content Quality Scoring]
        RawJSON --> TFIDF_Compromise[TF-IDF & Compromise NLP]
        TFIDF_Compromise --> TagsEntities[Keywords & Entities]
        TagsEntities --> TaxonomyMapping[Taxonomy Mapping & Synonym Expansion]
        TaxonomyMapping --> EnrichedJSON[Enriched JSON with Tags]
        EnrichedJSON --> DBInsert[DB Insert via Import Service]
    end

## 3. ER Diagram

erDiagram
    CAREER {
        int id PK
        string title
        string slug UK "Unique, indexed"
        text snapshot
        text overview
        string status "e.g., draft, published, archived, parsing_issue"
        float quality_score "0.0-1.0, for admin review"
        string template_version "Nullable, version of DOCX template used"
        string source_docx_filename "Nullable, original filename"
        timestamp last_parsed_at "Nullable"
        text parsing_notes "Nullable, notes from parser on issues"
        timestamp created_at
        timestamp updated_at
    }

    SECTION {
        int id PK
        int career_id FK
        string title
        text content
        int display_order
        string section_type "Nullable, e.g., 'MISCELLANEOUS' if unmatched"
    }

    TAG {
        int id PK
        string name
        string category
        int parent_tag_id FK "Nullable, self-referencing for hierarchy"
        timestamp created_at
        timestamp updated_at
        UNIQUE(name, category)
    }

    CAREER_TAG {
        int career_id FK
        int tag_id FK
        float relevance_score
        PRIMARY KEY (career_id, tag_id)
    }

    CAREER_SLUG_HISTORY {
        int id PK
        int career_id FK
        string old_slug UK "Old slug, indexed"
        timestamp changed_at
    }

    USER_FEEDBACK {
        int id PK
        int career_id FK "Nullable, if feedback is for a specific career"
        string page_url "URL feedback was submitted from"
        boolean is_helpful "Nullable, for Yes/No type feedback"
        int rating "Nullable, for star rating type feedback"
        text comment "Nullable"
        timestamp submitted_at
    }

    CAREER ||--o{ SECTION : "has"
    CAREER ||--o{ CAREER_TAG : "has"
    CAREER ||--o{ CAREER_SLUG_HISTORY : "had_slug"
    CAREER ||--o{ USER_FEEDBACK : "receives_feedback_on"
    TAG ||--o{ CAREER_TAG : "belongs_to"
    TAG ||--o| TAG : "is_child_of"

## 4. Component Tree

graph TD
    App --> Router
    Router --> PublicLayoutRoute[Public Layout Route]
    Router --> AdminLayoutRoute[Admin Layout Route]

    PublicLayoutRoute --> PublicLayout[PublicLayout Component]
    PublicLayout --> Header[Public Header]
    PublicLayout --> MainContent[MainContent (Outlet)]
    PublicLayout --> Footer[Public Footer]
    PublicLayout --> SkipLink[SkipLink (Accessibility)]

    MainContent --> HomePage
    MainContent --> SearchPage
    MainContent --> DetailPage
    MainContent --> AboutPage
    MainContent --> FAQPage
    MainContent --> ContactPage

    HomePage --> HeroBanner
    HomePage --> SearchBar
    HomePage --> CategoryList
    HomePage --> FeaturedCareers
    HomePage --> RecentlyViewedCareers
    HomePage --> InteractivePathExplorer[Interactive Career Path (Mini)]

    SearchPage --> Breadcrumbs
    SearchPage --> SearchBar
    SearchPage --> FilterPanel
    SearchPage --> SearchResults
    SearchPage --> PageSizeSelector
    SearchPage --> Pagination
    SearchResults --> CareerCard

    DetailPage --> Breadcrumbs
    DetailPage --> CareerHeader
    DetailPage --> CareerDetailNav[In-Page Nav (Tabs/Sticky)]
    CareerDetailNav --> CareerSectionDisplay[Dynamic Section Display]
    DetailPage --> CareerSections[Career Sections Content]
    DetailPage --> RelatedCareers
    DetailPage --> TagCloudDisplay[Tag Cloud]
    DetailPage --> ShareWidget[Share Career Widget]
    DetailPage --> FeedbackWidget[Feedback Widget]

    AdminLayoutRoute --> AdminLayout[AdminLayout Component]
    AdminLayout --> AdminHeader
    AdminLayout --> AdminMainContent[Admin MainContent (Outlet)]

    AdminMainContent --> AdminUploadPage[Upload & Preview Page]
    AdminMainContent --> AdminCareerListPage[Manage Careers Page]
    AdminMainContent --> AdminParsingIssuesPage[Parsing Issues Page]
    AdminMainContent --> AdminFeedbackPage[Feedback Review Page]
    AdminMainContent --> AdminCareerEditorPage[Career Editor Page]

    AdminUploadPage --> FileDropzone
    AdminUploadPage --> ParsePreviewPanel[Parse Preview Panel]

    AdminCareerListPage --> CareerListAdmin[Admin Career List]
    AdminCareerListPage --> AdminFilters[Admin Filters (status, quality)]
    CareerListAdmin --> CareerRowAdmin[Admin Career Row (with Edit/Archive buttons)]

    AdminCareerEditorPage --> CareerEditForm[Career Edit Form]
    CareerEditForm --> SectionEditorList[Section Editor List]
    CareerEditForm --> TagEditorAdmin[Admin Tag Editor]
    SectionEditorList --> SectionEditorItem[Individual Section Editor]