# Sprint 2: Vault

## 1. Narrative Goal

In this sprint, we're building the "Vault" – the robust data persistence layer for our Career Options Explorer. We'll design and implement a PostgreSQL database schema capable of storing all the structured career information parsed in Sprint 1. By the end of these four days, we'll have a fully functional database with tables for careers (including new fields for quality score, parsing notes, and template versioning), sections, tags (with hierarchy support), and their relationships. Crucially, we'll develop a clean repository layer abstracting database operations, and an `ImportService` to seamlessly transfer parsed DOCX data into this vault. This sprint is about creating a reliable, efficient, and extensible foundation for our career data, ensuring integrity, and preparing for the API and UI layers that will depend on it.

## 2. Back-of-the-Envelope Design

```mermaid
graph TD
    subgraph "Database Schema (PostgreSQL)"
        CareersTbl[CAREER Table (id, title, slug, snapshot, overview, status, quality_score, template_version, source_docx_filename, last_parsed_at, parsing_notes, ...)]
        SectionsTbl[SECTION Table (id, career_id, title, content, display_order, section_type)]
        TagsTbl[TAG Table (id, name, category, parent_tag_id)]
        CareerTagsTbl[CAREER_TAG Table (career_id, tag_id, relevance_score)]
        CareerSlugHistoryTbl[CAREER_SLUG_HISTORY Table (id, career_id, old_slug)]
        UserFeedbackTbl[USER_FEEDBACK Table (id, career_id, page_url, is_helpful, comment)]

        CareersTbl --o{ SectionsTbl
        CareersTbl --o{ CareerTagsTbl
        CareersTbl --o{ CareerSlugHistoryTbl
        CareersTbl --o{ UserFeedbackTbl
        TagsTbl --o{ CareerTagsTbl
        TagsTbl --o| TagsTbl : "parent/child"
    end

    subgraph "Server-Side Layers"
        ParserModule[Parser Module (from Sprint 1)] --> ImportService[ImportService]
        ImportService --> RepositoryLayer[Repository Layer]
        RepositoryLayer --> DBConnection[Database Connection (pg Pool)]
        DBConnection --> PostgreSQL[(PostgreSQL Database)]
    end

    subgraph "Repository Pattern Example"
        direction LR
        CareerCtrl[CareerController (Sprint 3)] --> CareerRepoInterface[ICareerRepository]
        CareerRepoInterface -.-> PostgresCareerRepoImpl[PostgresCareerRepository]
        PostgresCareerRepoImpl --> DBConnection
    end
```

## 3. Task Checklist

### Database Setup & Configuration

- [ ] **Install PostgreSQL Locally (if not done in Sprint 0):**
    *   Follow standard installation for your OS.
    *   Ensure PostgreSQL service is running.
    *   **VS Code Tip:** Use the `ms-ossdata.vscode-postgresql` (or `mtxr.sqltools` with `mtxr.sqltools-driver-pg`) extension for DB interaction within VS Code.

- [ ] **Create Database and User:**
    *   Commands (run as `postgres` user or superuser):
      ```sql
      CREATE USER career_user WITH ENCRYPTED PASSWORD 'career_password'; -- Choose a strong password
      CREATE DATABASE career_explorer OWNER career_user;
      -- Optionally, for local dev, grant all if needed, but be more specific in prod
      -- GRANT ALL PRIVILEGES ON DATABASE career_explorer TO career_user;
      ```
    *   Ensure `career_user` has permissions to create tables, sequences, etc., within the `career_explorer` database.

- [ ] **Update Server `.env` and `.env.example`:**
    *   Ensure `server/.env` and `server/.env.example` have the correct DB connection details (from Sprint 0, verify):
      ```
      DB_HOST=localhost
      DB_PORT=5432
      DB_NAME=career_explorer
      DB_USER=career_user
      DB_PASSWORD=career_password # Your chosen password
      NODE_ENV=development
      ```
    *   `dotenv` package should already be installed from Sprint 0.

### Database Migration System (`db-migrate`)

- [ ] **Install `db-migrate` and PostgreSQL driver** (in `server/` directory):
  ```bash
  npm install db-migrate db-migrate-pg
  # No separate @types for db-migrate usually needed
  ```

- [ ] **Create `database.json` configuration for `db-migrate`** (in `server/` root):
  ```json
  // server/database.json
  {
    "dev": {
      "driver": "pg",
      "host": { "ENV": "DB_HOST" },
      "port": { "ENV": "DB_PORT" },
      "user": { "ENV": "DB_USER" },
      "password": { "ENV": "DB_PASSWORD" },
      "database": { "ENV": "DB_NAME" },
      "ssl": false // Adjust if SSL is needed for local dev (usually not)
    },
    "test": {
      "driver": "pg",
      "host": { "ENV": "TEST_DB_HOST" }, // Separate test DB if needed
      "port": { "ENV": "TEST_DB_PORT" },
      "user": { "ENV": "TEST_DB_USER" },
      "password": { "ENV": "TEST_DB_PASSWORD" },
      "database": { "ENV": "TEST_DB_NAME" },
      "ssl": false
    }
    // Add staging/production configurations later
  }
  ```
  *Note: Using environment variables for `database.json` is best practice.*

- [ ] **Create `migrations` directory** (in `server/` root, if not already present):
  ```bash
  mkdir -p migrations
  ```

- [ ] **Create Initial Migration for `careers` Table:**
  ```bash
  npx db-migrate create create-careers-table --sql-file
  ```
  *   **`migrations/sqls/*-create-careers-table-up.sql`:**
    ```sql
    CREATE TABLE careers (
        id SERIAL PRIMARY KEY,
        title VARCHAR(255) NOT NULL,
        slug VARCHAR(300) NOT NULL,
        snapshot TEXT,
        overview TEXT,
        status VARCHAR(50) NOT NULL DEFAULT 'draft', -- e.g., draft, published, archived, parsing_issue
        quality_score REAL DEFAULT 0.5, -- 0.0 to 1.0
        template_version VARCHAR(100),
        source_docx_filename VARCHAR(255),
        last_parsed_at TIMESTAMPTZ,
        parsing_notes TEXT,
        created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );

    CREATE UNIQUE INDEX idx_careers_slug ON careers(slug);
    CREATE INDEX idx_careers_status ON careers(status);
    CREATE INDEX idx_careers_title_trgm ON careers USING gin (title gin_trgm_ops); -- For faster LIKE/ILIKE on title
    -- Ensure pg_trgm extension is enabled: CREATE EXTENSION IF NOT EXISTS pg_trgm; (run this once in DB)
    ```
  *   **`migrations/sqls/*-create-careers-table-down.sql`:**
    ```sql
    DROP TABLE careers;
    ```

- [ ] **Create Migration for `sections` Table:**
  ```bash
  npx db-migrate create create-sections-table --sql-file
  ```
  *   **`migrations/sqls/*-create-sections-table-up.sql`:**
    ```sql
    CREATE TABLE sections (
        id SERIAL PRIMARY KEY,
        career_id INTEGER NOT NULL REFERENCES careers(id) ON DELETE CASCADE,
        title VARCHAR(255) NOT NULL,
        content TEXT,
        display_order INTEGER NOT NULL DEFAULT 0,
        section_type VARCHAR(100), -- e.g., 'CAREER_SNAPSHOT', 'MISCELLANEOUS'
        created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );

    CREATE INDEX idx_sections_career_id ON sections(career_id);
    CREATE INDEX idx_sections_career_id_display_order ON sections(career_id, display_order);
    ```
  *   **`migrations/sqls/*-create-sections-table-down.sql`:**
    ```sql
    DROP TABLE sections;
    ```

- [ ] **Create Migration for `tags` Table:**
  ```bash
  npx db-migrate create create-tags-table --sql-file
  ```
  *   **`migrations/sqls/*-create-tags-table-up.sql`:**
    ```sql
    CREATE TABLE tags (
        id SERIAL PRIMARY KEY,
        name VARCHAR(150) NOT NULL,
        category VARCHAR(75) NOT NULL,
        parent_tag_id INTEGER REFERENCES tags(id) ON DELETE SET NULL, -- For hierarchy
        created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );

    CREATE UNIQUE INDEX idx_tags_name_category ON tags(LOWER(name), LOWER(category));
    CREATE INDEX idx_tags_category ON tags(category);
    CREATE INDEX idx_tags_parent_tag_id ON tags(parent_tag_id);
    CREATE INDEX idx_tags_name_trgm ON tags USING gin (name gin_trgm_ops);
    ```
  *   **`migrations/sqls/*-create-tags-table-down.sql`:**
    ```sql
    DROP TABLE tags;
    ```

- [ ] **Create Migration for `career_tags` (Join Table):**
  ```bash
  npx db-migrate create create-career-tags-table --sql-file
  ```
  *   **`migrations/sqls/*-create-career-tags-table-up.sql`:**
    ```sql
    CREATE TABLE career_tags (
        career_id INTEGER NOT NULL REFERENCES careers(id) ON DELETE CASCADE,
        tag_id INTEGER NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
        relevance_score REAL DEFAULT 1.0,
        PRIMARY KEY (career_id, tag_id)
    );

    CREATE INDEX idx_career_tags_tag_id ON career_tags(tag_id);
    -- (career_id is already part of PK, so often indexed)
    ```
  *   **`migrations/sqls/*-create-career-tags-table-down.sql`:**
    ```sql
    DROP TABLE career_tags;
    ```

- [ ] **Create Migration for `career_slug_history` Table:**
  ```bash
  npx db-migrate create create-career-slug-history-table --sql-file
  ```
  *   **`migrations/sqls/*-create-career-slug-history-table-up.sql`:**
    ```sql
    CREATE TABLE career_slug_history (
        id SERIAL PRIMARY KEY,
        career_id INTEGER NOT NULL REFERENCES careers(id) ON DELETE CASCADE,
        old_slug VARCHAR(300) NOT NULL,
        changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );

    CREATE INDEX idx_career_slug_history_old_slug ON career_slug_history(old_slug);
    CREATE INDEX idx_career_slug_history_career_id ON career_slug_history(career_id);
    ```
  *   **`migrations/sqls/*-create-career-slug-history-table-down.sql`:**
    ```sql
    DROP TABLE career_slug_history;
    ```

- [ ] **Create Migration for `user_feedback` Table:**
  ```bash
  npx db-migrate create create-user-feedback-table --sql-file
  ```
  *   **`migrations/sqls/*-create-user-feedback-table-up.sql`:**
    ```sql
    CREATE TABLE user_feedback (
        id SERIAL PRIMARY KEY,
        career_id INTEGER REFERENCES careers(id) ON DELETE SET NULL,
        page_url VARCHAR(500) NOT NULL, -- The URL where feedback was given
        is_helpful BOOLEAN,
        rating INTEGER CHECK (rating >= 1 AND rating <= 5), -- e.g., 1-5 stars
        comment TEXT,
        user_ip_hash VARCHAR(64), -- Optional: Hashed IP for basic rate limiting/uniqueness check, consider privacy
        submitted_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );

    CREATE INDEX idx_user_feedback_career_id ON user_feedback(career_id);
    CREATE INDEX idx_user_feedback_page_url ON user_feedback(page_url);
    ```
  *   **`migrations/sqls/*-create-user-feedback-table-down.sql`:**
    ```sql
    DROP TABLE user_feedback;
    ```

- [ ] **Add migration scripts to `server/package.json`:**
  ```json
  "scripts": {
    // ... other scripts from Sprint 0 & 1
    "db:migrate": "db-migrate up -e dev",
    "db:migrate:down": "db-migrate down -e dev -c 1", // Rollback one migration
    "db:migrate:reset": "db-migrate reset -e dev", // Rollback all migrations
    "db:create-migration": "db-migrate create" // Helper for creating new migrations
  }
  ```
  *Note: Added `-e dev` to specify environment from `database.json`.*

- [ ] **Enable `pg_trgm` Extension (Run once manually in PSQL for the `career_explorer` database):**
  ```sql
  CREATE EXTENSION IF NOT EXISTS pg_trgm;
  ```
  *This is for efficient `LIKE` / `ILIKE` queries.*

### Database Connection Module (`server/src/db/index.ts`)

- [ ] **Implement/Verify Database Connection Pool:** (Refine from Sprint 0 if basic version exists)
  ```typescript
  // server/src/db/index.ts
  import { Pool, PoolClient, QueryResultRow } from 'pg';
  import dotenv from 'dotenv';
  import logger from '../utils/logger'; // Assuming logger from Suggestion 14

  dotenv.config();

  const dbConfig = {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '5432', 10),
    database: process.env.DB_NAME || 'career_explorer',
    user: process.env.DB_USER || 'career_user',
    password: process.env.DB_PASSWORD || 'career_password',
    max: 20, // Max number of clients in the pool
    idleTimeoutMillis: 30000, // How long a client is allowed to remain idle before being closed
    connectionTimeoutMillis: 5000, // How long to wait for a connection if all are busy
  };

  const pool = new Pool(dbConfig);

  pool.on('connect', (client: PoolClient) => {
    logger.info('New client connected to PostgreSQL database');
    // You can set session variables here if needed, e.g., client.query('SET TIME ZONE "UTC";')
  });

  pool.on('error', (err: Error, client: PoolClient) => {
    logger.error('Unexpected error on idle PostgreSQL client', { error: err, clientInfo: client ? 'Client was provided' : 'Client was not provided' });
    // Depending on error, might want to process.exit(-1) or implement more robust recovery
  });
  
  logger.info('PostgreSQL Pool configured', { host: dbConfig.host, database: dbConfig.database });

  // Generic query function
  export const query = async <T extends QueryResultRow = any>(
    text: string,
    params?: any[]
  ): Promise<import('pg').QueryResult<T>> => {
    const start = Date.now();
    try {
      const res = await pool.query<T>(text, params);
      const duration = Date.now() - start;
      logger.debug('Executed query', { text, duration, rowCount: res.rowCount, params: params ? params.slice(0,5) : [] }); // Log first 5 params for brevity
      return res;
    } catch (error) {
      logger.error('Error executing query', { text, params, error });
      throw error; // Re-throw to be handled by the caller
    }
  };

  // Function to get a client for transactions
  export const getClient = async (): Promise<PoolClient> => {
    const client = await pool.connect();
    logger.debug('Client acquired from pool');
    return client;
  };

  // Helper for transactions
  export const transaction = async <T>(
    callback: (client: PoolClient) => Promise<T>
  ): Promise<T> => {
    const client = await getClient();
    try {
      await client.query('BEGIN');
      logger.debug('Transaction BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      logger.debug('Transaction COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      logger.error('Transaction ROLLBACK due to error', { error });
      throw error;
    } finally {
      client.release();
      logger.debug('Client released back to pool');
    }
  };

  // Test connection on startup (optional, but good for immediate feedback)
  (async () => {
    try {
      const client = await pool.connect();
      logger.info('Successfully connected to PostgreSQL database on startup.');
      client.release();
    } catch (error) {
      logger.error('Failed to connect to PostgreSQL database on startup.', { error });
      // process.exit(1); // Consider if fatal on startup
    }
  })();

  export default pool; // Export pool for specific uses if necessary
  ```

--- END OF FILE sprint2_vault.md (Part 1 of 2) ---

This first part covers the database setup, extensive migrations for all the new tables and fields we discussed, and a refined database connection module with better logging.

Please review this part. When you're ready, I'll provide Part 2, which will cover Data Models, Repository Interfaces, Repository Implementations, the Import Service, Seed Script, and Tests.

### Data Models (`server/src/models/`)

- [ ] Create/Update data model interfaces to reflect the database schema.
    *   **`server/src/models/Career.ts`:**
      ```typescript
      // server/src/models/Career.ts
      import { Section } from './Section';
      import { Tag } from './Tag';

      export type CareerStatus = 'draft' | 'published' | 'archived' | 'parsing_issue' | 'needs_review';

      export interface Career {
        id: number;
        title: string;
        slug: string;
        snapshot: string | null;
        overview: string | null;
        status: CareerStatus;
        quality_score: number | null;
        template_version: string | null;
        source_docx_filename: string | null;
        last_parsed_at: Date | null;
        parsing_notes: string | null; // Could be JSON string or plain text
        created_at: Date;
        updated_at: Date;

        // Optional relations, populated by specific repository methods
        sections?: Section[];
        tags?: Tag[];
      }

      // For creating a new career (some fields are optional or auto-generated)
      export interface CreateCareerInput {
        title: string;
        slug: string; // Can be auto-generated if not provided
        snapshot?: string;
        overview?: string;
        status?: CareerStatus; // Defaults to 'draft' or 'parsing_issue'
        quality_score?: number;
        template_version?: string;
        source_docx_filename?: string;
        last_parsed_at?: Date;
        parsing_notes?: string;
        // sections and tags are handled separately after career creation
      }

      // For updating an existing career
      export type UpdateCareerInput = Partial<Omit<CreateCareerInput, 'slug'>> & { slug?: string }; // slug can be updated carefully
      ```

    *   **`server/src/models/Section.ts`:**
      ```typescript
      // server/src/models/Section.ts
      export interface Section {
        id: number;
        career_id: number;
        title: string;
        content: string | null;
        display_order: number;
        section_type: string | null; // e.g., 'CAREER_SNAPSHOT', 'MISCELLANEOUS'
        created_at: Date;
        updated_at: Date;
      }

      export interface CreateSectionInput {
        career_id: number;
        title: string;
        content?: string;
        display_order?: number;
        section_type?: string;
      }
      export type UpdateSectionInput = Partial<CreateSectionInput>;
      ```

    *   **`server/src/models/Tag.ts`:**
      ```typescript
      // server/src/models/Tag.ts
      export interface Tag {
        id: number;
        name: string;
        category: string;
        parent_tag_id: number | null;
        created_at: Date;
        updated_at: Date;

        // For joined queries or aggregations
        relevance_score?: number; // When joined with career_tags
        career_count?: number;    // For popular tags
      }

      export interface CreateTagInput {
        name: string;
        category: string;
        parent_tag_id?: number;
      }
      export type UpdateTagInput = Partial<CreateTagInput>;
      ```
    *   **`server/src/models/CareerSlugHistory.ts`:**
        ```typescript
        // server/src/models/CareerSlugHistory.ts
        export interface CareerSlugHistory {
            id: number;
            career_id: number;
            old_slug: string;
            changed_at: Date;
        }
        ```
    *   **`server/src/models/UserFeedback.ts`:**
        ```typescript
        // server/src/models/UserFeedback.ts
        export interface UserFeedback {
            id: number;
            career_id: number | null;
            page_url: string;
            is_helpful: boolean | null;
            rating: number | null;
            comment: string | null;
            user_ip_hash: string | null;
            submitted_at: Date;
        }
        // Input types as needed
        ```

### Repository Layer (`server/src/repositories/`)

- [ ] **Define Repository Interfaces** (`server/src/repositories/interfaces/index.ts`):
    *   Define `ICareerRepository`, `ISectionRepository`, `ITagRepository`, etc.
    *   These interfaces will outline CRUD operations and any specific query methods.
      ```typescript
      // server/src/repositories/interfaces/index.ts
      import { Career, CreateCareerInput, UpdateCareerInput, CareerStatus } from '../../models/Career';
      import { Section, CreateSectionInput, UpdateSectionInput } from '../../models/Section';
      import { Tag, CreateTagInput, UpdateTagInput } from '../../models/Tag';
      // ... other model imports

      // Generic find options for pagination, filtering, sorting
      export interface FindAllOptions {
        limit?: number;
        offset?: number;
        sortBy?: string;
        sortOrder?: 'ASC' | 'DESC';
        status?: CareerStatus; // For filtering careers by status
        // Add other generic filter options if needed
      }
      
      export interface ICareerRepository {
        create(data: CreateCareerInput): Promise<Career>;
        findById(id: number): Promise<Career | null>;
        findBySlug(slug: string): Promise<Career | null>;
        // findAll to accept options for pagination, filtering by status, etc.
        findAll(options?: FindAllOptions): Promise<{ careers: Career[]; total: number }>;
        update(id: number, data: UpdateCareerInput): Promise<Career | null>;
        archive(id: number): Promise<boolean>; // Sets status to 'archived'
        restore(id: number, newStatus?: CareerStatus): Promise<boolean>; // Sets status to 'draft' or 'published'
        deletePermanently(id: number): Promise<boolean>; // Hard delete

        // Search and filter methods
        search(query: string, options?: FindAllOptions): Promise<{ careers: Career[]; total: number }>;
        findByTagId(tagId: number, options?: FindAllOptions): Promise<{ careers: Career[]; total: number }>;
        findByFilters(filters: { tagIds?: number[], categories?: string[], query?: string }, options?: FindAllOptions): Promise<{ careers: Career[]; total: number }>;
        countByFilters(filters: { tagIds?: number[], categories?: string[], query?: string }): Promise<number>;


        // Methods for managing slug history
        addSlugToHistory(careerId: number, oldSlug: string): Promise<void>;
        findCareerByOldSlug(oldSlug: string): Promise<{ currentSlug: string; careerId: number } | null>;
        
        // Methods for related data
        findCompleteById(id: number): Promise<Career | null>; // Includes sections and tags

        // Facet counting methods
        getFacetCounts(filters: { tagIds?: number[], categories?: string[], query?: string }): Promise<any>; // Define facet result type
        getFacetCountsForSearch(query: string): Promise<any>;
        getAllFacetCounts(): Promise<any>;
        countSearch(query: string): Promise<number>;
        count(): Promise<number>;
      }

      export interface ISectionRepository {
        create(data: CreateSectionInput): Promise<Section>;
        createMany(data: CreateSectionInput[]): Promise<Section[]>;
        findByCareerId(careerId: number): Promise<Section[]>;
        update(id: number, data: UpdateSectionInput): Promise<Section | null>;
        delete(id: number): Promise<boolean>;
        deleteByCareerId(careerId: number): Promise<boolean>;
      }

      export interface ITagRepository {
        create(data: CreateTagInput): Promise<Tag>;
        findOrCreate(name: string, category: string, parentTagId?: number): Promise<Tag>;
        findById(id: number): Promise<Tag | null>;
        findByNameAndCategory(name: string, category: string): Promise<Tag | null>;
        findAll(options?: { category?: string }): Promise<Tag[]>;
        findPopular(limit?: number): Promise<Tag[]>; // Returns tags with career_count
        findByCareerId(careerId: number): Promise<Tag[]>; // Returns tags with relevance_score
        update(id: number, data: UpdateTagInput): Promise<Tag | null>;
        delete(id: number): Promise<boolean>; // Consider if tags should be hard deleted or just unlinked

        // Career-Tag link management
        linkToCareer(careerId: number, tagId: number, relevanceScore?: number): Promise<void>;
        unlinkFromCareer(careerId: number, tagId: number): Promise<void>;
        updateLinkRelevance(careerId: number, tagId: number, relevanceScore: number): Promise<void>;
        unlinkAllFromCareer(careerId: number): Promise<void>;
      }
      
      export interface IUserFeedbackRepository {
        // Define methods like create, findByCareerId, findAll, etc.
      }
      ```

- [ ] **Implement PostgreSQL Repositories** (`server/src/repositories/implementations/`):
    *   Create `PostgresCareerRepository.ts`, `PostgresSectionRepository.ts`, `PostgresTagRepository.ts`.
    *   Implement all methods defined in the interfaces using the `query` and `transaction` helpers from `src/db/index.ts`.
    *   **Example snippet for `PostgresCareerRepository.ts` `create` method:**
      ```typescript
      // server/src/repositories/implementations/PostgresCareerRepository.ts
      import { query } from '../../db';
      import { Career, CreateCareerInput } from '../../models/Career';
      import { ICareerRepository } from '../interfaces'; // Adjust path as needed

      export class PostgresCareerRepository implements ICareerRepository {
        async create(data: CreateCareerInput): Promise<Career> {
          const { title, slug, snapshot, overview, status = 'draft', ...otherData } = data;
          const sql = `
            INSERT INTO careers (title, slug, snapshot, overview, status, quality_score, template_version, source_docx_filename, last_parsed_at, parsing_notes)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
            RETURNING *;
          `;
          const params = [
            title, slug, snapshot, overview, status,
            otherData.quality_score, otherData.template_version,
            otherData.source_docx_filename, otherData.last_parsed_at,
            otherData.parsing_notes ? JSON.stringify(otherData.parsing_notes) : null // Assuming parsing_notes is stored as JSON text
          ];
          const result = await query<Career>(sql, params);
          return result.rows[0];
        }
        // ... other methods
      }
      ```
    *   **Slug Uniqueness in `findBySlug` and `create/update`:** Ensure logic to handle potential slug conflicts or auto-incrementing if desired (Suggestion 17).
    *   **Soft Delete Logic:** The `archive` method should update `status`. `deletePermanently` performs actual deletion. Public-facing `find` methods should filter by `status = 'published'`.

- [ ] **Create Repository Factory** (`server/src/repositories/index.ts`):
  ```typescript
  // server/src/repositories/index.ts
  import { ICareerRepository, ISectionRepository, ITagRepository } from './interfaces';
  import { PostgresCareerRepository } from './implementations/PostgresCareerRepository';
  import { PostgresSectionRepository } from './implementations/PostgresSectionRepository';
  import { PostgresTagRepository } from './implementations/PostgresTagRepository';
  // Import other repository implementations

  export const careerRepository: ICareerRepository = new PostgresCareerRepository();
  export const sectionRepository: ISectionRepository = new PostgresSectionRepository();
  export const tagRepository: ITagRepository = new PostgresTagRepository();
  // Export other repository instances
  ```

### Data Import Service (`server/src/services/`)

- [ ] **Create `ImportService.ts`:**
  ```typescript
  // server/src/services/ImportService.ts
  import { CareerParser, ParsedCareer, CareerSection as ParsedSection, CareerTag as ParsedTag } from '../parser';
  import { careerRepository, sectionRepository, tagRepository } from '../repositories';
  import { transaction } from '../db';
  import { CreateCareerInput, CareerStatus, Career } from '../models/Career';
  import { CreateSectionInput } from '../models/Section';
  import logger from '../utils/logger';

  export class ImportService {
    private parser: CareerParser;

    constructor() {
      this.parser = new CareerParser({
        // Default parser options, can be overridden
        extractRawHtml: false,
        defaultSectionTitle: 'Additional Information',
      });
    }

    private async findOrCreateTags(parsedTags: ParsedTag[]): Promise<Map<string, number>> {
      const tagMap = new Map<string, number>(); // name:category -> id
      for (const pTag of parsedTags) {
        const dbTag = await tagRepository.findOrCreate(pTag.name, pTag.category);
        tagMap.set(`${pTag.name}:${pTag.category}`, dbTag.id);
      }
      return tagMap;
    }

    public async importCareerFromParsedData(
        parsedCareerData: ParsedCareer,
        sourceFilename?: string // Original DOCX filename
    ): Promise<Career> {
      return transaction(async (client) => { // Use the client from transaction for all DB ops in this scope
        // Use repositories with the transaction client if they support it,
        // otherwise, ensure repository methods can run in a transaction started by this service.
        // For simplicity here, assuming repositories use the global pool, which is fine if transaction wraps them.

        let career: Career | null = await careerRepository.findBySlug(parsedCareerData.slug);
        const careerInput: CreateCareerInput | Partial<Career> = {
          title: parsedCareerData.title,
          slug: parsedCareerData.slug,
          snapshot: parsedCareerData.snapshot,
          overview: parsedCareerData.overview,
          status: parsedCareerData.quality_score && parsedCareerData.quality_score < 0.6 ? 'parsing_issue' : 'draft',
          quality_score: parsedCareerData.quality_score,
          template_version: parsedCareerData.template_version,
          source_docx_filename: sourceFilename,
          last_parsed_at: new Date(),
          parsing_notes: parsedCareerData.parsing_notes ? JSON.stringify(parsedCareerData.parsing_notes) : null,
        };

        if (career) {
          // Career exists, update it
          logger.info(`Updating existing career: ${career.title} (ID: ${career.id})`);
          const updated = await careerRepository.update(career.id, careerInput as Partial<Career>);
          if (!updated) throw new Error(`Failed to update career ${career.id}`);
          career = updated;
          // Clear existing sections and tags for this career before re-adding
          await sectionRepository.deleteByCareerId(career.id);
          await tagRepository.unlinkAllFromCareer(career.id);
        } else {
          // Career does not exist, create it
          logger.info(`Creating new career: ${parsedCareerData.title}`);
          career = await careerRepository.create(careerInput as CreateCareerInput);
        }
        if (!career) throw new Error('Failed to create or update career');

        // Create sections
        if (parsedCareerData.sections && parsedCareerData.sections.length > 0) {
          const sectionInputs: CreateSectionInput[] = parsedCareerData.sections.map(s => ({
            career_id: career!.id,
            title: s.title,
            content: s.content,
            display_order: s.displayOrder,
            section_type: s.section_type,
          }));
          await sectionRepository.createMany(sectionInputs); // Assuming createMany exists
        }

        // Create/Link tags
        if (parsedCareerData.tags && parsedCareerData.tags.length > 0) {
          const tagIdMap = await this.findOrCreateTags(parsedCareerData.tags);
          for (const pTag of parsedCareerData.tags) {
            const tagId = tagIdMap.get(`${pTag.name}:${pTag.category}`);
            if (tagId) {
              await tagRepository.linkToCareer(career!.id, tagId, pTag.relevanceScore);
            }
          }
        }
        
        // Fetch the complete career to return
        const completeCareer = await careerRepository.findCompleteById(career.id);
        if(!completeCareer) throw new Error (`Failed to fetch complete career data for ${career.id} after import.`);
        return completeCareer;
      });
    }

    public async importCareerFromFile(filePath: string): Promise<Career> {
      const parsedCareerData = await this.parser.parseFile(filePath);
      const filename = path.basename(filePath);
      return this.importCareerFromParsedData(parsedCareerData, filename);
    }

    public async importCareerFromBuffer(buffer: Buffer, originalFilename?: string): Promise<Career> {
      const parsedCareerData = await this.parser.parseBuffer(buffer);
      return this.importCareerFromParsedData(parsedCareerData, originalFilename);
    }
  }

  export const importService = new ImportService();
  ```

### Database Seed Script (`server/src/db/seed.ts`)

- [ ] **Implement Seed Script** to populate initial data using the `ImportService`.
    *   It should iterate over `.docx` files in a specific seed directory (e.g., `docs/seed_examples/` from Suggestion 13) and use `importService.importCareerFromFile()`.
  ```typescript
  // server/src/db/seed.ts
  import * as fs from 'fs/promises';
  import * as path from 'path';
  import { importService } from '../services/ImportService';
  import logger from '../utils/logger';
  import pool from './index'; // To close the pool after seeding

  const SEED_FILES_DIR = path.resolve(__dirname, '../../../docs/seed_examples');

  async function seedDatabase() {
    logger.info('Starting database seed process...');
    try {
      const files = await fs.readdir(SEED_FILES_DIR);
      const docxFiles = files.filter(file => file.toLowerCase().endsWith('.docx'));

      if (docxFiles.length === 0) {
        logger.info(`No .docx seed files found in ${SEED_FILES_DIR}. Skipping seed.`);
        return;
      }

      logger.info(`Found ${docxFiles.length} DOCX files to seed: ${docxFiles.join(', ')}`);

      for (const docxFile of docxFiles) {
        const filePath = path.join(SEED_FILES_DIR, docxFile);
        try {
          logger.info(`Seeding from file: ${docxFile}`);
          const importedCareer = await importService.importCareerFromFile(filePath);
          // Optionally update status of seeded careers to 'published'
          if (importedCareer.status !== 'published') {
            await careerRepository.update(importedCareer.id, { status: 'published' });
            logger.info(`  Set status to 'published' for seeded career: ${importedCareer.title}`);
          }
          logger.info(`  Successfully seeded career: ${importedCareer.title} (ID: ${importedCareer.id})`);
        } catch (fileError) {
          logger.error(`  Error seeding file ${docxFile}:`, { error: fileError });
        }
      }
      logger.info('Database seed process completed.');
    } catch (error) {
      logger.error('Fatal error during database seed process:', { error });
      if ((error as NodeJS.ErrnoException).code === 'ENOENT' && (error as NodeJS.ErrnoException).path === SEED_FILES_DIR) {
        logger.warn(`Seed directory not found: ${SEED_FILES_DIR}. Ensure it exists with example DOCX files.`);
      }
    } finally {
      await pool.end(); // Close the database pool
      logger.info('Database pool closed after seeding.');
    }
  }

  if (require.main === module) {
    seedDatabase().catch(err => {
      logger.error('Unhandled error in seed script execution:', { error: err });
      pool.end(); // Ensure pool is closed on unhandled error
      process.exit(1);
    });
  }
  ```
- [ ] Add/update `db:seed` script in `server/package.json`:
  ```json
  "scripts": {
    // ...
    "db:seed": "ts-node src/db/seed.ts"
  }
  ```
  *Note: Create `docs/seed_examples/` directory at the project root and place 1-2 sample DOCX files there for testing the seed.*

### Unit Tests for Repositories and Services

- [ ] **Write tests for each repository implementation** (`server/src/repositories/__tests__/`):
    *   Focus on testing each method: `create`, `findById`, `findBySlug`, `findAll`, `update`, `delete`, specific query methods.
    *   Use a separate test database or ensure data cleanup (`beforeAll`, `afterEach`, `afterAll`).
    *   Mock the `db/index.ts` `query` function or use an in-memory DB/Dockerized test DB for more isolated tests.
    *   **Example for `PostgresCareerRepository.test.ts`:**
      ```typescript
      // server/src/repositories/__tests__/PostgresCareerRepository.test.ts
      import { PostgresCareerRepository } from '../implementations/PostgresCareerRepository';
      import { careerRepository, sectionRepository, tagRepository } from '../index'; // Using exported instances
      import pool, { query as dbQuery } from '../../db'; // For direct DB manipulation/cleanup
      import { CreateCareerInput } from '../../models/Career';

      describe('PostgresCareerRepository', () => {
        let testCareerId: number;
        const repo = new PostgresCareerRepository(); // Test the implementation directly

        beforeAll(async () => {
          // Clean up relevant tables or use a transaction rollback strategy for each test
          await dbQuery('DELETE FROM career_tags;');
          await dbQuery('DELETE FROM sections;');
          await dbQuery('DELETE FROM career_slug_history;');
          await dbQuery('DELETE FROM careers;');
          await dbQuery('DELETE FROM tags;');
        });

        afterAll(async () => {
          await pool.end();
        });

        it('should create a new career', async () => {
          const careerData: CreateCareerInput = {
            title: 'Test Software Engineer',
            slug: 'test-software-engineer',
            snapshot: 'Builds software.',
            overview: 'Detailed overview.',
            status: 'draft',
          };
          const createdCareer = await repo.create(careerData);
          expect(createdCareer).toBeDefined();
          expect(createdCareer.id).toBeGreaterThan(0);
          expect(createdCareer.title).toBe(careerData.title);
          testCareerId = createdCareer.id;
        });

        it('should find a career by ID', async () => {
          const foundCareer = await repo.findById(testCareerId);
          expect(foundCareer).not.toBeNull();
          expect(foundCareer?.id).toBe(testCareerId);
        });
        
        // ... more tests for findBySlug, update, archive, deletePermanently, etc.
        
        it('should add slug to history and find by old slug', async () => {
            const originalSlug = 'test-software-engineer';
            const newSlug = 'updated-test-software-engineer';

            // Update slug
            await repo.update(testCareerId, { slug: newSlug });
            // Manually add to history (or test this as part of update if integrated)
            await repo.addSlugToHistory(testCareerId, originalSlug);

            const result = await repo.findCareerByOldSlug(originalSlug);
            expect(result).not.toBeNull();
            expect(result?.careerId).toBe(testCareerId);
            expect(result?.currentSlug).toBe(newSlug);
        });
      });
      ```
- [ ] **Write tests for `ImportService.ts`** (`server/src/services/__tests__/ImportService.test.ts`):
    *   Mock the `CareerParser` to return predefined `ParsedCareer` data.
    *   Mock repository methods to verify they are called correctly by the service.
    *   Test creation of new careers and updates to existing careers.
    *   Test correct handling of sections and tags during import.
    *   Test transactionality (e.g., if section creation fails, career creation is rolled back).

## 4. Creative Add-Ons (Stretch)

*   [ ] **Database Indexing Review & Optimization (Post-Schema):** After defining tables, review query patterns from repositories and add more specific indexes if needed (e.g., GIN/GIST indexes for full-text search on `overview` or `section.content`, composite indexes for common filter combinations). This was touched upon in Suggestion 2's migrations.
*   [ ] **Automated Slug Generation Trigger (DB Level):** Implement the PostgreSQL trigger for automatic slug generation (from Suggestion 2) if robust application-level generation is not preferred. Ensure it handles uniqueness.
*   [ ] **Generic Repository/Service Layer:** Abstract common CRUD operations into a generic base repository/service class to reduce boilerplate if many more models are anticipated.
*   [ ] **Connection Read Replicas (Future Scaling):** For read-heavy loads, configure the DB connection pool to differentiate between write master and read replicas (far future, but good to keep in mind for `pg` pool options).

## 5. Run & Verify

### Database & Migrations

1.  Ensure PostgreSQL is running and accessible with configured credentials.
2.  Run migrations from the `server/` directory:
    ```bash
    npm run db:migrate
    ```
3.  **Verify in PSQL or VS Code SQLTools:**
    *   Connect to `career_explorer` database.
    *   Check that all tables (`careers`, `sections`, `tags`, `career_tags`, `career_slug_history`, `user_feedback`) are created with correct columns and indexes:
      ```sql
      \dt -- list tables
      \d careers -- describe careers table
      -- etc. for other tables
      ```
    *   Check if `pg_trgm` extension is enabled: `\dx pg_trgm`

### Seed Script

1.  Create `docs/seed_examples/` directory at the project root.
2.  Add 1-2 well-structured `.docx` files (e.g., `DataScientist.docx`, `UXDesigner.docx`) to `docs/seed_examples/`.
3.  Run the seed script from the `server/` directory:
    ```bash
    npm run db:seed
    ```
4.  **Verify Data in PSQL:**
    ```sql
    SELECT id, title, slug, status, quality_score FROM careers;
    SELECT career_id, title FROM sections LIMIT 5;
    SELECT name, category FROM tags LIMIT 5;
    SELECT * FROM career_tags LIMIT 5;
    ```
    *   Confirm careers, sections, and tags from your seed files are present.

### Unit Tests

1.  Run repository and service tests from the `server/` directory:
    ```bash
    npm test -- --testPathPattern=repositories # For repository tests
    npm test -- --testPathPattern=services   # For service tests
    # Or run all server tests:
    # npm test
    ```
    *   **In VS Code:** Use Test Explorer to run tests.

### Success Criteria

- ✅ All database migrations run successfully without errors.
- ✅ Database schema matches the ER diagram (including new fields and tables).
- ✅ `pg_trgm` extension is enabled in the database.
- ✅ Database connection pool (`db/index.ts`) connects successfully on startup.
- ✅ Repository interfaces and implementations cover required CRUD and query operations.
- ✅ `ImportService` correctly parses data (mocked parser output) and saves careers, sections, and tags to the database, handling both new and existing careers.
- ✅ `db:seed` script successfully populates the database with initial career data from example DOCX files.
- ✅ All repository and service unit tests pass with ≥ 85% line coverage.
- ✅ Data integrity is maintained (e.g., foreign key constraints, uniqueness).

## 6. Artifacts to Commit

- **Branch name**: `sprint-2-vault`
- **Mandatory files**:
    - `server/database.json`
    - `server/migrations/` directory with all SQL migration files.
    - `server/src/db/index.ts` (updated connection pool).
    - `server/src/models/` directory with all data model files.
    - `server/src/repositories/` directory (interfaces, implementations, index).
    - `server/src/services/ImportService.ts`.
    - `server/src/db/seed.ts`.
    - Test files in `server/src/repositories/__tests__/` and `server/src/services/__tests__/`.
    - Updated `server/package.json` and `server/package-lock.json`.
    - Example DOCX files in `docs/seed_examples/`.
- **PR Checklist**:
    - [ ] Database schema implemented as designed, including all new fields and tables.
    - [ ] Migrations are reversible (up and down scripts work).
    - [ ] Repository layer correctly abstracts database operations.
    - [ ] `ImportService` successfully populates data from (mocked) parser output.
    - [ ] Seed script populates initial data.
    - [ ] All unit tests for this layer pass with required coverage.
    - [ ] Sensitive information (like passwords) is handled via environment variables.
    - [ ] Code is well-commented, especially repository queries and service logic.

