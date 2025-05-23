# Sprint 4: Scribe

## 1. Narrative Goal

In this "Scribe" sprint, we empower our administrators by building the initial version of the Admin User Interface. Our primary goal is to create a clean, intuitive, and mobile-first interface (with an iOS design aesthetic) where administrators can efficiently manage career content. By the end of these five days, admins will be able to:
1.  Upload new career DOCX files via a drag-and-drop interface.
2.  Preview the parsed content (title, sections, tags, quality score, parsing notes) before final import.
3.  View a list of all careers, including their status, quality score, and parsing provenance.
4.  Access a dedicated view to triage careers flagged with parsing issues.
5.  Perform basic editing of career details (title, slug, snapshot, overview, status).
6.  Archive and restore careers.
This sprint focuses on the core content ingestion and management workflows, transforming the backend capabilities into tangible tools for our "Scribes."

## 2. Back-of-the-Envelope Design

```mermaid
graph TD
    subgraph "Admin UI - Core Pages & Layout (React + TailwindCSS)"
        AdminApp[Admin App Root (/admin)] --> AdminRouter[Admin Router]
        AdminRouter --> AdminLayoutComp[AdminLayout Component]
        
        AdminLayoutComp --> AdminHeader[Admin Header (Navigation)]
        AdminLayoutComp --> AdminMainOutlet[Main Content Area (Outlet)]
        
        AdminMainOutlet --> UploadPage[Upload & Preview Page (/admin/upload)]
        AdminMainOutlet --> CareerListPage[Manage Careers Page (/admin/careers)]
        AdminMainOutlet --> CareerEditorPage[Career Editor Page (/admin/careers/edit/:id)]
        AdminMainOutlet --> ParsingIssuesPage[Parsing Issues Triage Page (/admin/parsing-issues)]
        AdminMainOutlet --> FeedbackReviewPage[Feedback Review Page (/admin/feedback) - Stretch]
    end

    subgraph "Upload Page Components"
        UploadPage --> FileDropzoneComp[FileDropzone Component]
        UploadPage --> PreviewPanelComp[PreviewPanel Component (displays ParsedCareer)]
        UploadPage --> UploadConfirmButton[Confirm & Upload Button]
    end

    subgraph "Manage Careers Page Components"
        CareerListPage --> CareerListTable[CareerList Component (Table)]
        CareerListTable --> CareerListRow[CareerListRow (with Edit, Archive/Restore buttons)]
        CareerListPage --> StatusFilters[Status/Quality Filters]
    end

    subgraph "Career Editor Page Components"
        CareerEditorPage --> CareerEditFormComp[CareerEditForm Component]
        CareerEditFormComp --> CoreInfoEditor[Core Info Fields (Title, Slug, Snapshot, Overview, Status)]
        CareerEditFormComp --> SectionsEditor[Sections Editor (Add, Edit, Delete, Reorder)]
        CareerEditFormComp --> TagsEditor[Tags Editor (Add, Remove, Edit Relevance)]
    end
    
    subgraph "Client-API Interaction (Admin UI)"
        FileDropzoneComp -->|File Data| API_Preview["POST /api/v1/admin/upload/preview"]
        UploadConfirmButton -->|File Data| API_Upload["POST /api/v1/admin/upload"]
        
        CareerListTable -->|Fetch Data| API_GetAdminCareers["GET /api/v1/admin/careers (with filters)"]
        CareerListRow -- Edit --> CareerEditorPage
        CareerListRow -- Archive/Restore --> API_UpdateStatus["PATCH /api/v1/admin/careers/:id/archive | /restore"]
        
        CareerEditFormComp -->|Save Data| API_UpdateCareer["PUT /api/v1/admin/careers/:id"]
        CareerEditFormComp -->|Fetch Data| API_GetCareerDetails["GET /api/v1/careers/:id (or admin equivalent)"]
        
        ParsingIssuesPage -->|Fetch Data| API_GetParsingIssues["GET /api/v1/admin/careers?status=parsing_issue"]
    end
    
    ReactAdminState[Admin UI State (Zustand/Context)]
    ReactAdminState <--> UploadPage
    ReactAdminState <--> CareerListPage
    ReactAdminState <--> CareerEditorPage
```

## 3. Task Checklist

### Client-Side Setup & Admin Routing (Extending Sprint 0)

- [ ] **Install Client-Side Dependencies** (in `client/` directory):
  ```bash
  npm install react-router-dom react-dropzone zustand # Or other state manager
  npm install -D @types/react-router-dom @types/react-dropzone
  # Consider a form library for the editor: npm install react-hook-form
  # For icons: npm install lucide-react (or similar icon library)
  ```

- [ ] **Setup Client Configuration** (`client/src/config.ts`, if not already done):
  ```typescript
  // client/src/config.ts
  export const CLIENT_CONFIG = {
    API_BASE_URL: import.meta.env.VITE_API_BASE_URL || '/api/v1', // Use Vite env vars
    // Add other client-specific configs
  };
  ```
  *Create `client/.env` and `client/.env.example` with `VITE_API_BASE_URL=http://localhost:3001/api/v1` (for dev when not using proxy) or just `/api/v1` (if proxy is set up in `vite.config.ts`).*

- [ ] **Create Admin Layout Component** (`client/src/components/admin/AdminLayout.tsx`):
    *   Includes header with navigation (Upload, Manage Careers, Parsing Issues).
    *   Uses `<Outlet />` for nested routes.
    *   Styled with TailwindCSS, iOS aesthetic.

- [ ] **Implement Admin Pages Shells:**
    *   `client/src/pages/admin/AdminDashboardPage.tsx` (optional, could be redirect or simple overview)
    *   `client/src/pages/admin/UploadPage.tsx`
    *   `client/src/pages/admin/CareerListPage.tsx`
    *   `client/src/pages/admin/CareerEditorPage.tsx`
    *   `client/src/pages/admin/ParsingIssuesPage.tsx`

- [ ] **Configure Admin Routing in `client/src/App.tsx`:**
    *   Use `react-router-dom` to set up routes under `/admin`.
    *   Protect admin routes (placeholder for now, actual auth is future).
      ```typescript
      // client/src/App.tsx (snippet)
      // ... imports
      // const AdminDashboardPage = lazy(() => import('./pages/admin/AdminDashboardPage'));
      const UploadPage = lazy(() => import('./pages/admin/UploadPage'));
      const CareerListPage = lazy(() => import('./pages/admin/CareerListPage'));
      const CareerEditorPage = lazy(() => import('./pages/admin/CareerEditorPage'));
      const ParsingIssuesPage = lazy(() => import('./pages/admin/ParsingIssuesPage'));
      const AdminLayout = lazy(() => import('./components/admin/AdminLayout'));
      
      // ... inside <Routes>
      <Route path="/admin" element={<AdminLayout />}>
        <Route index element={<Navigate to="upload" replace />} /> {/* Default to upload */}
        <Route path="upload" element={<UploadPage />} />
        <Route path="careers" element={<CareerListPage />} />
        <Route path="careers/edit/:careerId" element={<CareerEditorPage />} />
        <Route path="careers/new" element={<CareerEditorPage />} /> {/* For creating new manually */}
        <Route path="parsing-issues" element={<ParsingIssuesPage />} />
        {/* <Route path="feedback" element={<FeedbackReviewPage />} /> Stretch */}
      </Route>
      ```

### File Upload & Preview Functionality (`UploadPage.tsx`)

- [ ] **Create `FileDropzone.tsx` Component** (`client/src/components/admin/FileDropzone.tsx`):
    *   Uses `react-dropzone`.
    *   Accepts only `.docx` files.
    *   Handles file size limits.
    *   Provides visual feedback (drag state, selected file, errors).
    *   Callback `onFileAccepted(file: File)`.

- [ ] **Create `PreviewPanel.tsx` Component** (`client/src/components/admin/PreviewPanel.tsx`):
    *   Accepts `parsedCareer: ParsedCareer | null` and `isLoading: boolean` props.
    *   Displays: `title`, `slug`, `snapshot`, `overview`.
    *   Lists `sections` with their titles and content.
    *   Lists `tags` with name, category, and relevance.
    *   Displays `quality_score` and `parsing_notes`.
    *   Shows a loading state.

- [ ] **Integrate Upload and Preview in `UploadPage.tsx`:**
    *   State management for `selectedFile`, `previewData`, `isLoadingPreview`, `isLoadingUpload`, `uploadStatus`, `error`.
    *   `handleFileAccepted`: Sets `selectedFile`, calls `handlePreview`.
    *   `handlePreview(file: File)`:
        *   Sends file to `POST /api/v1/admin/upload/preview`.
        *   Updates `previewData`, `isLoadingPreview`, `error`.
    *   `handleUpload()`:
        *   Sends `selectedFile` to `POST /api/v1/admin/upload`.
        *   Updates `uploadStatus`, `isLoadingUpload`, `error`.
        *   On success, clears `selectedFile` and `previewData`, maybe navigates to career list or editor.
    *   UI: FileDropzone, PreviewPanel, "Confirm & Upload" button (enabled only if preview is successful).

### Career List & Management (`CareerListPage.tsx`)

- [ ] **Create `CareerListAdmin.tsx` Component** (`client/src/components/admin/CareerListAdmin.tsx`):
    *   Fetches careers from `GET /api/v1/admin/careers` (can include `status` filter).
    *   Displays careers in a table: `Title`, `Slug`, `Status`, `Quality Score`, `Last Parsed At`, `Source Filename`, `Updated At`, `Actions`.
    *   Actions column: "Edit", "Archive"/"Restore" buttons.
    *   Handles loading and error states.
    *   Basic client-side pagination or triggers server-side pagination.

- [ ] **Implement Filter Controls on `CareerListPage.tsx`:**
    *   Dropdowns/buttons to filter by `status` (Draft, Published, Archived, Parsing Issue).
    *   (Stretch) Slider or input to filter by `quality_score` range.

- [ ] **Implement "Archive" and "Restore" Functionality:**
    *   Buttons in `CareerListAdmin.tsx` call respective API endpoints (`PATCH /admin/careers/:id/archive` or `/restore`).
    *   Refresh list or update item state on success.

### Parsing Issues Triage Page (`ParsingIssuesPage.tsx`)

- [ ] **Develop `ParsingIssuesPage.tsx`:**
    *   Fetches careers with `status=parsing_issue` from `GET /api/v1/admin/careers?status=parsing_issue`.
    *   Uses a similar list component to `CareerListAdmin.tsx` but tailored for this view.
    *   Prominently displays `parsing_notes`, `source_docx_filename`, `last_parsed_at`.
    *   Each row has an "Edit" button linking to `CareerEditorPage.tsx`.
    *   (Stretch) Option to "Mark as Resolved" (changes status to `draft`).
    *   (Stretch) Option to re-parse a specific DOCX if the source file is still available or can be re-uploaded.

### Career Editor Functionality (`CareerEditorPage.tsx`)

- [ ] **Develop `CareerEditorPage.tsx`:**
    *   Fetches career data if `careerId` is present in URL params (for editing), or initializes an empty form (for new).
    *   Uses a form library like `react-hook-form` for managing form state and validation.

- [ ] **Create `CareerEditForm.tsx` Component** (`client/src/components/admin/CareerEditForm.tsx`):
    *   **Core Fields:** Inputs for `title`, `slug` (with on-the-fly uniqueness check via API debounce?), `snapshot` (textarea), `overview` (textarea).
    *   Dropdown for `status`.
    *   Display (read-only or editable) for `quality_score`, `template_version`, `source_docx_filename`, `last_parsed_at`, `parsing_notes`.
    *   Save button calls `POST /api/v1/admin/careers` (new) or `PUT /api/v1/admin/careers/:id` (edit).

- [ ] **Create `SectionEditor.tsx` Component** (`client/src/components/admin/SectionEditor.tsx`):
    *   Manages a list of sections for the current career.
    *   Allows adding new sections (title, content, type).
    *   Allows editing existing section titles and content (textareas).
    *   Allows reordering sections (drag-and-drop or up/down buttons).
    *   Allows deleting sections.

- [ ] **Create `TagEditorAdmin.tsx` Component** (`client/src/components/admin/TagEditorAdmin.tsx`):
    *   Displays current tags associated with the career.
    *   Input to add new tags (name, category, relevance score). Could use an autocomplete for existing tags in the system.
    *   Ability to remove tags from the career.
    *   Ability to edit relevance score for existing tags on this career.

## 4. Creative Add-Ons (Stretch)

- [ ] **Rich Text Editor for Section Content:** Instead of plain textareas, use a lightweight WYSIWYG editor (e.g., TipTap, Quill, Slate.js) for editing section content in `CareerEditorPage.tsx`.
- [ ] **Bulk Actions in `CareerListAdmin.tsx`:** Select multiple careers to archive, publish, or delete.
- [ ] **Visual Diff for Preview:** When re-uploading a DOCX for an existing career, show a diff between the current DB content and the newly parsed content in the `PreviewPanel.tsx`.
- [ ] **Admin Dashboard Overview:** A simple `AdminDashboardPage.tsx` showing stats: total careers, careers by status, recent uploads, careers needing review.
- [ ] **Slug Uniqueness Live Check:** In `CareerEditForm.tsx`, as the admin types a slug, make a debounced API call to check if it's unique.

## 5. Run & Verify

### Setup & Start

1.  Ensure client dependencies (routing, dropzone, state management) are installed.
2.  Ensure backend server (Sprint 3) is running and accessible.
3.  Start the client development server: `cd client && npm run dev`.

### Manual Browser Checks & Admin Workflows

1.  **Navigation:**
    *   Access `/admin`. Should redirect to `/admin/upload` or show a dashboard.
    *   Navigate between Upload, Manage Careers, Parsing Issues pages using admin header links.
2.  **Upload & Preview Workflow (`/admin/upload`):**
    *   Drag & drop (or select) a valid `.docx` file.
        *   Verify preview panel populates correctly with title, slug, sections, tags, quality score, parsing notes.
    *   Try uploading an invalid file type (e.g., `.txt`, `.pdf`). Verify error message.
    *   Try uploading a file exceeding size limits. Verify error message.
    *   Click "Confirm & Upload." Verify success message and that the career appears in the "Manage Careers" list.
    *   Test with a poorly formatted DOCX to see if it gets flagged for parsing issues.
3.  **Manage Careers Workflow (`/admin/careers`):**
    *   Verify career list displays data correctly (title, slug, status, quality, timestamps).
    *   Test status filters.
    *   Test "Archive" action. Career should move to an "Archived" filter view or be visually distinct.
    *   Test "Restore" action for an archived career.
4.  **Parsing Issues Triage Workflow (`/admin/parsing-issues`):**
    *   If any careers were flagged with `parsing_issue` status during upload, verify they appear here.
    *   Check if `parsing_notes` are displayed.
    *   Click "Edit" for a problematic career.
5.  **Career Editor Workflow (`/admin/careers/edit/:id` or `/admin/careers/new`):**
    *   Access editor for an existing career. Verify fields are pre-populated.
    *   Modify title, snapshot, overview. Save and verify changes in the list and on the public site (if published).
    *   Edit a section's content. Save and verify.
    *   Add a new section. Save and verify.
    *   Delete a section. Save and verify.
    *   Add a new tag to the career. Save and verify.
    *   Remove a tag from the career. Save and verify.
    *   Change career status (e.g., from `draft` to `published`). Verify on public site.

### Success Criteria

- ✅ Admin UI is accessible at `/admin` and sub-routes.
- ✅ FileDropzone correctly handles DOCX uploads, type/size validation.
- ✅ PreviewPanel accurately displays parsed career data, including quality score and parsing notes.
- ✅ Admins can successfully upload DOCX files, which are then imported into the database.
- ✅ Career List page displays all careers with relevant admin-specific information (status, quality, provenance).
- ✅ Admins can filter careers by status.
- ✅ Admins can archive and restore careers.
- ✅ Parsing Issues page correctly lists careers flagged during import.
- ✅ Admins can navigate to an editor to correct/update career details (core fields, sections, tags, status).
- ✅ Basic editing functionality is implemented and persists changes to the backend.
- ✅ UI follows mobile-first principles and iOS design aesthetics.
- ✅ Admin actions provide clear feedback (success/error messages).

## 6. Artifacts to Commit

- **Branch name**: `sprint-4-scribe`
- **Mandatory files**:
    - `client/src/components/admin/` (AdminLayout, FileDropzone, PreviewPanel, CareerListAdmin, CareerEditForm, SectionEditor, TagEditorAdmin, etc.)
    - `client/src/pages/admin/` (UploadPage, CareerListPage, CareerEditorPage, ParsingIssuesPage)
    - Updates to `client/src/App.tsx` for admin routing.
    - Updates to `client/src/types/career.ts` if new client-specific admin types are needed.
    - `client/src/config.ts` (if created/updated).
    - Updated `client/package.json` and `client/package-lock.json`.
- **PR Checklist**:
    - [ ] Admin can upload and preview DOCX files.
    - [ ] Admin can confirm upload, importing data to DB.
    - [ ] Admin can view a list of all careers with status, quality, and provenance information.
    - [ ] Admin can filter careers (at least by status).
    - [ ] Admin can archive and restore careers.
    - [ ] Admin can access a dedicated view for careers with parsing issues.
    - [ ] Admin can perform basic edits on career information (core fields, status; section/tag editing is primary goal).
    - [ ] UI is responsive and adheres to iOS styling.
    - [ ] All admin actions provide user feedback.
    - [ ] Code is well-commented.