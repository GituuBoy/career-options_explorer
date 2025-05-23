# Sprint 3: Gatekeeper

## 1. Narrative Goal

In this sprint, we construct the "Gatekeeper"—our Express API layer. This vital component will serve as the secure and efficient interface between our rich database (built in Sprint 2) and the upcoming frontend UIs. By the end of these five days, we'll have a comprehensive set of API endpoints for creating, retrieving, updating, and managing (archiving, restoring, permanently deleting) career data. A key feature will be a robust file upload system enabling administrators to submit DOCX files, which are then processed by our `ImportService` and stored. We'll implement thorough input validation, structured error handling, detailed API documentation using Swagger/OpenAPI, and mechanisms for faceted search data. This sprint is about building a well-defined, testable, and secure API that reliably exposes our application's data and functionality.

## 2. Back-of-the-Envelope Design

```mermaid
graph TD
    ClientFE[Client Frontend (React)] --> ExpressRouter{Express Router}
    AdminFE[Admin Frontend (React)] --> ExpressRouter

    subgraph "API Layer (Express.js)"
        direction LR
        ExpressRouter --> AuthMiddleware[Auth Middleware (Admin Routes)]
        AuthMiddleware --> ValidationMiddleware[Validation (express-validator)]
        ValidationMiddleware --> Controllers[API Controllers]
        Controllers --> ServicesLayer[Services (e.g., ImportService)]
        ServicesLayer --> RepositoryLayer[Repositories (from Sprint 2)]
        
        ExpressRouter -.-> LoggingMiddleware[Logging (Logger/Morgan)]
        ExpressRouter -.-> ErrorHandlingMiddleware[Error Handling]
        ExpressRouter -.-> FileUploadMiddleware[File Upload (Multer)]
        FileUploadMiddleware --> Controllers
    end
    
    RepositoryLayer --> DBLayer[(PostgreSQL Database)]

    subgraph "Key API Endpoints"
        direction TB
        PublicRoutes[Public Routes]
        AdminRoutes[Admin Routes (Protected)]

        PublicRoutes --> GetCareers["GET /careers (list, search, facets)"]
        PublicRoutes --> GetSingleCareer["GET /careers/:slug (details, handles old slugs)"]
        PublicRoutes --> GetTags["GET /tags (list, popular, by category)"]
        PublicRoutes --> GetFacets["GET /careers/facets (filter options)"]
        PublicRoutes --> PostFeedback["POST /feedback (submit user feedback)"]

        AdminRoutes --> UploadDocx["POST /admin/upload (parse & import)"]
        AdminRoutes --> PreviewDocx["POST /admin/upload/preview (parse only)"]
        AdminRoutes --> CreateCareer["POST /admin/careers (manual create)"]
        AdminRoutes --> GetAdminCareers["GET /admin/careers (all statuses)"]
        AdminRoutes --> UpdateCareer["PUT /admin/careers/:id (edit all fields)"]
        AdminRoutes --> ArchiveCareer["PATCH /admin/careers/:id/archive (soft delete)"]
        AdminRoutes --> RestoreCareer["PATCH /admin/careers/:id/restore"]
        AdminRoutes --> DeleteCareerPerm["DELETE /admin/careers/:id/permanently (hard delete)"]
        AdminRoutes --> GetAdminFeedback["GET /admin/feedback (review feedback)"]
    end
```

## 3. Task Checklist

### API Structure and Core Setup

- [ ] Create API-specific directory structure in `server/src/api/`:
  ```bash
  # (Assuming already in server directory)
  # mkdir -p src/api/controllers src/api/middleware src/api/routes src/api/validators src/api/docs
  # (Most of this structure should exist from Sprint 0, verify/create `docs` for swagger)
  ```

- [ ] Install API-related dependencies in `server/`:
  ```bash
  npm install express-validator multer morgan swagger-ui-express swagger-jsdoc http-status-codes
  npm install -D @types/multer @types/morgan @types/swagger-ui-express @types/swagger-jsdoc
  ```
  *   `express-validator`: For input validation.
  *   `multer`: For handling `multipart/form-data` (file uploads).
  *   `morgan`: HTTP request logger middleware (can be complemented by Winston).
  *   `swagger-ui-express`, `swagger-jsdoc`: For API documentation.
  *   `http-status-codes`: For using named HTTP status codes.

- [ ] Create/Refine API Configuration (`server/src/api/config.ts` or use general server config):
  ```typescript
  // server/src/config/api.config.ts (or merge into a general server config)
  import dotenv from 'dotenv';
  dotenv.config(); // Ensure .env is loaded

  export const API_CONFIG = {
    PORT: parseInt(process.env.PORT || '3001', 10),
    NODE_ENV: process.env.NODE_ENV || 'development',
    API_VERSION: 'v1',
    API_PREFIX: `/api/${process.env.API_VERSION || 'v1'}`, // Consistent prefix

    CORS_ORIGIN: process.env.CORS_ORIGIN || 'http://localhost:3000', // Client URL

    UPLOAD_DIR: process.env.UPLOAD_DIR || './uploads', // Relative to server root
    MAX_FILE_SIZE_MB: parseInt(process.env.MAX_FILE_SIZE_MB || '10', 10), // In MB

    PAGINATION_DEFAULT_LIMIT: parseInt(process.env.PAGINATION_DEFAULT_LIMIT || '12', 10),
    PAGINATION_MAX_LIMIT: parseInt(process.env.PAGINATION_MAX_LIMIT || '100', 10),
    
    // JWT settings (placeholders for now, full auth is a separate concern)
    JWT_SECRET: process.env.JWT_SECRET || 'THIS_IS_A_DEV_SECRET_REPLACE_ME!',
    // Add other relevant API configs
  };
  ```
  *Ensure `UPLOAD_DIR` exists or is created on startup.*
  ```bash
  mkdir -p uploads # In server root
  touch uploads/.gitkeep
  ```

### Core Middleware Implementation (`server/src/api/middleware/`)

- [ ] **Error Handling Middleware (`errorHandler.ts`):**
    *   Refine from Sprint 0.
    *   Use `http-status-codes` for clarity.
    *   Log errors using the structured logger (from Suggestion 14).
    *   Distinguish between `ApiError` (custom controlled errors) and unexpected server errors.
      ```typescript
      // server/src/api/middleware/errorHandler.ts
      import { Request, Response, NextFunction } from 'express';
      import { StatusCodes, getReasonPhrase } from 'http-status-codes';
      import logger from '../../utils/logger';
      import { API_CONFIG } from '../../config/api.config'; // Adjust path as needed

      export class ApiError extends Error {
        public readonly statusCode: StatusCodes;
        public readonly isOperational: boolean; // To distinguish programmer errors from operational errors

        constructor(message: string, statusCode: StatusCodes, isOperational: boolean = true, cause?: Error) {
          super(message);
          this.name = this.constructor.name;
          this.statusCode = statusCode;
          this.isOperational = isOperational;
          if (cause) this.cause = cause;
          Error.captureStackTrace(this, this.constructor);
        }
      }
      
      // eslint-disable-next-line @typescript-eslint/no-unused-vars
      export const errorHandler = (err: Error | ApiError, req: Request, res: Response, next: NextFunction) => {
        const errorTimestamp = new Date().toISOString();
        let statusCode = StatusCodes.INTERNAL_SERVER_ERROR;
        let message = getReasonPhrase(StatusCodes.INTERNAL_SERVER_ERROR);
        let errorsArray: any[] | undefined = undefined;

        if (err instanceof ApiError && err.isOperational) {
          statusCode = err.statusCode;
          message = err.message;
        } else if (err.name === 'ValidationError' && (err as any).errors) { // express-validator
          statusCode = StatusCodes.UNPROCESSABLE_ENTITY; // Or BAD_REQUEST
          message = 'Validation failed';
          errorsArray = (err as any).errors;
        } else {
          // Log unexpected errors with more detail
          logger.error('Unhandled API Error', {
            timestamp: errorTimestamp,
            requestId: (req as any).id, // From requestIdMiddleware
            path: req.path,
            method: req.method,
            errorName: err.name,
            errorMessage: err.message,
            stack: err.stack,
            cause: (err as any).cause,
          });
        }
        
        // In development, send more details
        const responseBody: Record<string, any> = {
            status: 'error',
            statusCode,
            message,
            timestamp: errorTimestamp,
            requestId: (req as any).id,
        };

        if (errorsArray) {
            responseBody.errors = errorsArray;
        }

        if (API_CONFIG.NODE_ENV === 'development' && !(err instanceof ApiError && err.isOperational)) {
          responseBody.stack = err.stack;
          if ((err as any).cause) responseBody.cause = (err as any).cause.toString();
        }
        
        res.status(statusCode).json(responseBody);
      };

      export const notFoundHandler = (req: Request, res: Response, next: NextFunction) => {
        next(new ApiError(`Resource not found at ${req.originalUrl}`, StatusCodes.NOT_FOUND));
      };

      // Wrapper for async route handlers to catch errors and pass to error middleware
      export const asyncHandler = (fn: (req: Request, res: Response, next: NextFunction) => Promise<any>) => 
        (req: Request, res: Response, next: NextFunction) => {
          Promise.resolve(fn(req, res, next)).catch(next);
      };
      ```

- [ ] **Request ID Middleware (`requestId.ts`):** (From Suggestion 14)
    *   Generate unique ID per request for tracing.
      ```typescript
      // server/src/api/middleware/requestId.ts
      import { Request, Response, NextFunction } from 'express';
      import { v4 as uuidv4 } from 'uuid';

      // Extend Express Request type
      declare global {
        namespace Express {
          interface Request {
            id?: string;
          }
        }
      }

      export const requestIdMiddleware = (req: Request, res: Response, next: NextFunction) => {
        const requestId = req.headers['x-request-id'] as string || uuidv4();
        req.id = requestId;
        res.setHeader('X-Request-Id', requestId);
        next();
      };
      ```

- [ ] **Request Logging Middleware (`requestLogger.ts`):** (Refine from Sprint 0 or integrate with structured logger)
    *   Use Winston (or similar) for structured request/response logging, including request ID, duration, status.
      ```typescript
      // server/src/api/middleware/requestLogger.ts
      import { Request, Response, NextFunction } from 'express';
      import morgan from 'morgan'; // Can still use morgan for simple console logs
      import logger from '../../utils/logger';
      import { API_CONFIG } from '../../config/api.config';

      // Morgan stream to Winston
      const stream = {
        write: (message: string) => logger.http(message.trim()),
      };
      
      // Skip morgan in test environment or if winston is handling it fully
      const skip = () => API_CONFIG.NODE_ENV === 'test';

      // Configure morgan
      const morganMiddleware = morgan(
        // Use a format that includes request ID if available
        (tokens, req: Request, res: Response) => {
          return [
            req.id, // From requestIdMiddleware
            tokens.method(req, res),
            tokens.url(req, res),
            tokens.status(req, res),
            tokens.res(req, res, 'content-length'), '-',
            tokens['response-time'](req, res), 'ms'
          ].join(' ');
        },
        { stream, skip }
      );
      
      // More detailed logging middleware using Winston directly (optional, can replace or augment Morgan)
      export const detailedRequestLogger = (req: Request, res: Response, next: NextFunction) => {
        const start = process.hrtime();
        const { method, originalUrl, ip, headers } = req;
      
        logger.info('Request received', {
          requestId: req.id,
          method,
          url: originalUrl,
          ip,
          userAgent: headers['user-agent'],
        });
      
        res.on('finish', () => {
          const diff = process.hrtime(start);
          const duration = (diff[0] * 1e3) + (diff[1] * 1e-6); // milliseconds
          logger.info('Request finished', {
            requestId: req.id,
            method,
            url: originalUrl,
            statusCode: res.statusCode,
            durationMs: parseFloat(duration.toFixed(2)),
          });
        });
        next();
      };

      export default API_CONFIG.NODE_ENV === 'production' ? detailedRequestLogger : morganMiddleware;
      ```

- [ ] **File Upload Middleware (`fileUpload.ts`):**
    *   Configure `multer` for DOCX uploads, setting limits and storage destination.
    *   Error handling for `MulterError` (file size, type).
      ```typescript
      // server/src/api/middleware/fileUpload.ts
      import multer, { MulterError } from 'multer';
      import path from 'path';
      import fs from 'fs';
      import { Request, Response, NextFunction } from 'express';
      import { API_CONFIG } from '../../config/api.config';
      import { ApiError } from './errorHandler';
      import { StatusCodes } from 'http-status-codes';

      const uploadDir = path.resolve(process.cwd(), API_CONFIG.UPLOAD_DIR);
      if (!fs.existsSync(uploadDir)) {
        fs.mkdirSync(uploadDir, { recursive: true });
      }

      const storage = multer.diskStorage({
        destination: (req, file, cb) => cb(null, uploadDir),
        filename: (req, file, cb) => {
          const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
          const extension = path.extname(file.originalname);
          cb(null, `${file.fieldname}-${uniqueSuffix}${extension}`);
        },
      });

      const fileFilter = (req: Request, file: Express.Multer.File, cb: multer.FileFilterCallback) => {
        if (file.mimetype === 'application/vnd.openxmlformats-officedocument.wordprocessingml.document') {
          cb(null, true);
        } else {
          cb(new ApiError('Invalid file type. Only .docx files are allowed.', StatusCodes.UNSUPPORTED_MEDIA_TYPE) as any);
        }
      };

      export const uploadMiddleware = multer({
        storage,
        fileFilter,
        limits: {
          fileSize: API_CONFIG.MAX_FILE_SIZE_MB * 1024 * 1024, // Convert MB to Bytes
        },
      });
      
      // Specific error handler for multer, to be used after upload.single() or upload.array()
      export const handleMulterError = (err: any, req: Request, res: Response, next: NextFunction) => {
        if (err instanceof MulterError) {
          if (err.code === 'LIMIT_FILE_SIZE') {
            return next(new ApiError(`File too large. Max size is ${API_CONFIG.MAX_FILE_SIZE_MB}MB.`, StatusCodes.REQUEST_TOO_LONG));
          }
          return next(new ApiError(err.message, StatusCodes.BAD_REQUEST));
        }
        // If not a MulterError, pass it to the general error handler
        next(err);
      };
      ```
- [ ] **Authentication/Authorization Middleware Placeholder (`authMiddleware.ts`):**
    *   Create a placeholder for admin routes. Actual JWT logic can be a future enhancement.
      ```typescript
      // server/src/api/middleware/authMiddleware.ts
      import { Request, Response, NextFunction } from 'express';
      import { StatusCodes } from 'http-status-codes';
      import { ApiError } from './errorHandler';
      // import jwt from 'jsonwebtoken'; // npm install jsonwebtoken @types/jsonwebtoken
      // import { API_CONFIG } from '../../config/api.config';

      // Placeholder: In a real app, this would verify a JWT
      export const requireAdminAuth = (req: Request, res: Response, next: NextFunction) => {
        // const authHeader = req.headers.authorization;
        // if (authHeader && authHeader.startsWith('Bearer ')) {
        //   const token = authHeader.substring(7);
        //   try {
        //     const decoded = jwt.verify(token, API_CONFIG.JWT_SECRET);
        //     (req as any).user = decoded; // Attach user info to request
        //     // Add role checks if necessary: if ((req as any).user.role !== 'admin') throw ...
        //     return next();
        //   } catch (err) {
        //     return next(new ApiError('Invalid or expired token.', StatusCodes.UNAUTHORIZED));
        //   }
        // }
        // return next(new ApiError('Authorization header missing or malformed.', StatusCodes.UNAUTHORIZED));
        
        // For now, bypass auth in development, or use a simple API key check
        if (API_CONFIG.NODE_ENV !== 'production' && req.headers['x-dev-bypass-auth'] === 'true') {
            (req as any).user = { id: 'dev-admin', role: 'admin' }; // Mock user
            return next();
        }
        // In a real scenario, you'd enable the JWT logic above.
        // For this project without full auth, we might just allow admin routes for now.
        // Consider this a placeholder for actual security.
        logger.warn('Auth middleware is currently bypassed or using placeholder logic for admin routes.');
        next(); 
      };
      ```

### API Validators (`server/src/api/validators/`)

- [ ] **Create/Update `careerValidators.ts`:**
    *   Define validation rules using `express-validator` for creating/updating careers, including new fields like `status`, `quality_score`, etc.
    *   Validate pagination params (`limit`, `page`).
    *   Validate search query params.
      ```typescript
      // server/src/api/validators/careerValidators.ts
      import { body, param, query, ValidationChain } from 'express-validator';
      import { API_CONFIG } from '../../config/api.config'; // Adjust path

      const validCareerStatuses = ['draft', 'published', 'archived', 'parsing_issue', 'needs_review'];

      export const careerValidationRules = {
        // For POST /admin/careers and PUT /admin/careers/:id (parts of it)
        upsertCareer: [
          body('title').notEmpty().withMessage('Title is required').isString().trim().isLength({ min: 3, max: 255 }),
          body('slug').optional().isString().trim().isLength({ max: 300 }).matches(/^[a-z0-9]+(?:-[a-z0-9]+)*$/).withMessage('Slug must be lowercase alphanumeric with hyphens'),
          body('snapshot').optional({ nullable: true }).isString().trim(),
          body('overview').optional({ nullable: true }).isString().trim(),
          body('status').optional().isIn(validCareerStatuses).withMessage(`Status must be one of: ${validCareerStatuses.join(', ')}`),
          body('quality_score').optional({ nullable: true }).isFloat({ min: 0, max: 1 }).withMessage('Quality score must be a float between 0 and 1'),
          body('template_version').optional({ nullable: true }).isString().trim().isLength({ max: 100 }),
          body('source_docx_filename').optional({ nullable: true }).isString().trim().isLength({ max: 255 }),
          body('parsing_notes').optional({ nullable: true }).isString(), // Could be JSON string

          // Sections validation (example)
          body('sections').optional().isArray().withMessage('Sections must be an array'),
          body('sections.*.title').notEmpty().isString().trim().isLength({ max: 255 }),
          body('sections.*.content').optional({ nullable: true }).isString(),
          body('sections.*.display_order').isInt({ min: 0 }),
          body('sections.*.section_type').optional({nullable: true}).isString().isLength({max:100}),


          // Tags validation (example - assumes tags are provided as objects)
          body('tags').optional().isArray().withMessage('Tags must be an array'),
          body('tags.*.name').notEmpty().isString().trim().isLength({ max: 150 }),
          body('tags.*.category').notEmpty().isString().trim().isLength({ max: 75 }),
          body('tags.*.relevance_score').optional().isFloat({ min: 0, max: 1 }),
        ],
        // For GET /careers, GET /careers/search
        listCareers: [
          query('limit').optional().isInt({ min: 1, max: API_CONFIG.PAGINATION_MAX_LIMIT }).toInt().withMessage(`Limit must be between 1 and ${API_CONFIG.PAGINATION_MAX_LIMIT}`),
          query('page').optional().isInt({ min: 1 }).toInt().withMessage('Page must be a positive integer'),
          query('status').optional().isIn(validCareerStatuses).withMessage('Invalid status filter'),
          query('q').optional().isString().trim(),
          query('tag').optional().isString().trim(), // Comma-separated IDs or names
          query('category').optional().isString().trim(), // Comma-separated category names for filtering
        ],
        // For GET /careers/:slug or /careers/:id
        getCareerByIdentifier: [
          param('identifier').notEmpty().isString().trim().withMessage('Career identifier (slug or ID) is required'),
        ],
        // For PUT /admin/careers/:id and DELETE operations
        careerIdParam: [
          param('id').isInt({ min: 1 }).toInt().withMessage('Valid Career ID parameter is required'),
        ],
      };
      ```

- [ ] **Create `feedbackValidators.ts`:**
    ```typescript
    // server/src/api/validators/feedbackValidators.ts
    import { body } from 'express-validator';
    
    export const feedbackValidationRules = {
      submitFeedback: [
        body('page_url').notEmpty().isURL().withMessage('Valid page_url is required'),
        body('is_helpful').optional({nullable: true}).isBoolean().withMessage('is_helpful must be true or false'),
        body('rating').optional({nullable: true}).isInt({ min: 1, max: 5 }).withMessage('Rating must be an integer between 1 and 5'),
        body('comment').optional({nullable: true}).isString().trim().isLength({ max: 2000 }),
        // Note: user_ip_hash would be set server-side
      ],
    };
    ```

### API Controllers (`server/src/api/controllers/`)

- [ ] **Update `careerController.ts`:**
    *   Handle CRUD operations for careers, including sections and tags.
    *   Implement logic for fetching careers by slug (handling old slugs via `CareerSlugHistory` - Suggestion 17).
    *   Implement search with pagination and filtering (by status for admin, only published for public).
    *   Implement facet data retrieval.
    *   Implement archive, restore, and permanent delete actions.
    *   Use `validationResult` from `express-validator`.

- [ ] **Update `uploadController.ts`:**
    *   Use `ImportService` to process uploaded DOCX files.
    *   Handle both full import and preview-only parsing.
    *   Return structured `ParsedCareer` for preview, and full `Career` for import.

- [ ] **Create `tagController.ts`:**
    *   Endpoints to list all tags, popular tags, tags by category.
    *   Potentially an endpoint to get tag hierarchy.

- [ ] **Create `feedbackController.ts`:**
    *   Endpoint to receive and store user feedback.

### API Routes (`server/src/api/routes/`)

- [ ] **Update `careerRoutes.ts`:**
    *   Public: `GET /`, `GET /search`, `GET /:identifier`, `GET /facets`.
    *   Admin (protected): `POST /`, `PUT /:id`, `PATCH /:id/archive`, `PATCH /:id/restore`, `DELETE /:id/permanently`.

- [ ] **Update `uploadRoutes.ts`:** (Admin, protected)
    *   `POST /` (for full import)
    *   `POST /preview`

- [ ] **Update `tagRoutes.ts`:** (Public)
    *   `GET /`, `GET /popular`, `GET /category/:categoryName`

- [ ] **Create `feedbackRoutes.ts`:** (Public)
    *   `POST /`

- [ ] **Update Main Router (`server/src/api/routes/index.ts`):**
    *   Mount all specific route handlers under the `API_CONFIG.API_PREFIX`.
    *   Include the health check route.

### API Documentation (Swagger/OpenAPI)

- [ ] **Create `swaggerDefinition.ts`** (or similar in `server/src/api/docs/`)
    *   Define OpenAPI spec (info, servers, components like schemas for Career, Tag, Section, Error, Pagination, etc.).
    *   Reference JSDoc comments in route files for endpoint definitions.
      ```typescript
      // server/src/api/docs/swaggerDefinition.ts
      import swaggerJsdoc from 'swagger-jsdoc';
      import { API_CONFIG } from '../../config/api.config'; // Adjust path

      const swaggerOptions = {
        definition: {
          openapi: '3.0.0',
          info: {
            title: 'Career Options Explorer API',
            version: API_CONFIG.API_VERSION || '1.0.0',
            description: 'API for managing and exploring career options.',
            contact: { name: 'Support', email: 'dev@example.com' },
          },
          servers: [
            { url: `http://localhost:${API_CONFIG.PORT}${API_CONFIG.API_PREFIX}`, description: 'Development server' },
            // Add production server URL later
          ],
          components: {
            // Define reusable schemas (Career, Section, Tag, ErrorResponse, Pagination, etc.)
            // These should match your data models
            schemas: {
              Career: { /* ... detailed schema ... */ },
              Section: { /* ... detailed schema ... */ },
              Tag: { /* ... detailed schema ... */ },
              ErrorResponse: {
                type: 'object',
                properties: {
                  status: { type: 'string', example: 'error' },
                  statusCode: { type: 'integer' },
                  message: { type: 'string' },
                  timestamp: { type: 'string', format: 'date-time' },
                  requestId: { type: 'string', format: 'uuid' },
                  errors: { type: 'array', items: { type: 'object' } }
                }
              },
              // ... more schemas
            },
            securitySchemes: { // Placeholder for JWT auth
                bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' }
            }
          },
          // security: [{ bearerAuth: [] }] // Global security if all/most routes are protected
        },
        // Path to the API docs (JSDoc comments in route files)
        apis: ['./src/api/routes/*.ts', './src/api/controllers/*.ts'], // Adjust paths to where your JSDoc comments are
      };
      export const swaggerSpec = swaggerJsdoc(swaggerOptions);
      ```

- [ ] **Add JSDoc comments** to controller methods/route definitions for Swagger.

### Main API Server Setup (`server/src/server.ts` & `server/src/app.ts` or `api/index.ts`)

- [ ] **Create `app.ts` (or `api/index.ts`)** to configure Express app, middleware, routes, Swagger UI.
  ```typescript
  // server/src/app.ts (or api/index.ts)
  import express, { Express } from 'express';
  import cors from 'cors';
  import bodyParser from 'body-parser';
  import swaggerUi from 'swagger-ui-express';
  import { API_CONFIG } from './config/api.config'; // Adjust path
  import { errorHandler, notFoundHandler, requestIdMiddleware } from './api/middleware/errorHandler'; // Adjust path
  import requestLoggerMiddleware from './api/middleware/requestLogger'; // Adjust path
  import mainApiRouter from './api/routes'; // Adjust path (index.ts from routes)
  import { swaggerSpec } from './api/docs/swaggerDefinition'; // Adjust path
  import logger from './utils/logger'; // Adjust path

  const app: Express = express();

  // Core Middleware
  app.use(requestIdMiddleware); // Must be early
  app.use(cors({ origin: API_CONFIG.CORS_ORIGIN }));
  app.use(bodyParser.json({ limit: '5mb' })); // For career create/update with sections
  app.use(bodyParser.urlencoded({ extended: true, limit: '5mb' }));
  app.use(requestLoggerMiddleware); // Structured request logger

  // API Routes
  app.use(API_CONFIG.API_PREFIX, mainApiRouter);

  // Swagger UI
  if (API_CONFIG.NODE_ENV !== 'production') {
    app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec));
    logger.info(`Swagger UI available at /api-docs`);
  }
  
  // Static serving of uploads (for dev access, usually handled by Nginx/CDN in prod)
  app.use('/uploads', express.static(path.resolve(process.cwd(), API_CONFIG.UPLOAD_DIR)));


  // Error Handling Middleware (must be last)
  app.use(notFoundHandler);
  app.use(errorHandler);

  export default app;
  ```

- [ ] **Update `server.ts`** to import and start the configured `app`.
  ```typescript
  // server/src/server.ts
  import app from './app'; // Assuming app.ts is in src/
  import { API_CONFIG } from './config/api.config'; // Adjust path
  import logger from './utils/logger'; // Adjust path

  const server = app.listen(API_CONFIG.PORT, () => {
    logger.info(`Server is live and listening on http://localhost:${API_CONFIG.PORT}`);
    logger.info(`API Prefix: ${API_CONFIG.API_PREFIX}`);
    if (API_CONFIG.NODE_ENV !== 'production') {
      logger.info(`Swagger Docs available at http://localhost:${API_CONFIG.PORT}/api-docs`);
    }
  });

  // Graceful shutdown
  const signals = ['SIGINT', 'SIGTERM', 'SIGQUIT'];
  signals.forEach(signal => {
    process.on(signal, () => {
      logger.info(`Received ${signal}, shutting down gracefully...`);
      server.close(() => {
        logger.info('HTTP server closed.');
        // Close database pool, other resources
        // require('./db').default.end(() => { // Assuming pool export from db/index.ts
        //   logger.info('Database pool closed.');
        //   process.exit(0);
        // });
        process.exit(0); // Simpler exit for now
      });
    });
  });
  ```

### API Tests (`server/src/api/__tests__/`)

- [ ] **Write integration tests for API endpoints** using `supertest`.
    *   Test successful responses (200, 201, 204).
    *   Test error responses (400, 401, 403, 404, 422, 500).
    *   Test input validation.
    *   Test file uploads and parsing results.
    *   Test pagination and filtering logic.
    *   Ensure test database is used and reset between tests/suites.

## 4. Creative Add-Ons (Stretch)

- [ ] **API Rate Limiting:** Implement middleware (e.g., `express-rate-limit`) for public and admin endpoints to prevent abuse.
- [ ] **Response Caching Strategy:** Implement caching for frequently accessed, rarely changing GET requests (e.g., using `apicache` middleware or custom Redis/Memcached integration). Consider cache invalidation strategies.
- [ ] **API Versioning in URL vs. Headers:** Current plan uses URL versioning (`/api/v1`). Consider pros/cons of header-based versioning for future flexibility.
- [ ] **Advanced Search Capabilities:** Integrate more advanced search like Elasticsearch if PostgreSQL full-text search becomes insufficient.
- [ ] **Webhook for Content Updates:** If external systems need to be notified of career updates, implement a simple webhook system.

## 5. Run & Verify

### Setup & Build

1.  Ensure all new dependencies are installed in `server/`: `npm install`
2.  Build the server: `npm run build` (in `server/`)

### Start Server

1.  Start the API server: `npm run dev` (in `server/`)
2.  **Observe Logs:** Check console for startup messages, including Swagger UI path.

### API Testing (Manual & Automated)

1.  **Swagger UI:**
    *   Open browser to `http://localhost:PORT/api-docs` (e.g., `http://localhost:3001/api-docs`).
    *   Explore available endpoints.
    *   Use "Try it out" feature to test endpoints manually (e.g., GET careers, upload a DOCX).
2.  **cURL / Postman / Insomnia:**
    *   Test various endpoints with different inputs and headers.
    *   Verify correct status codes, response bodies, and error handling.
    *   Test file upload: `curl -X POST -F "file=@/path/to/your/Sample.docx" http://localhost:3001/api/v1/admin/upload`
3.  **Automated Tests (Jest + Supertest):**
    *   Run API tests from `server/` directory: `npm test -- --testPathPattern=api`
    *   **In VS Code:** Use Test Explorer for `server/src/api/__tests__/`.

### Specific Checks:

-   **File Upload:**
    *   Upload a valid DOCX: check for 201 (or 200) and correct career data in response. Verify data in DB.
    *   Upload an invalid file type: check for 415 (Unsupported Media Type) or 400.
    *   Upload a file too large: check for 413 (Request Entity Too Large) or 400.
-   **Career CRUD:**
    *   Create, read, update, archive, restore, and (if implemented) permanently delete careers. Verify DB changes.
    *   Test slug uniqueness and old slug redirection (if implemented).
-   **Search & Filter:**
    *   Test `GET /careers/search` with various `q`, `tag`, `category`, `limit`, `page` parameters.
    *   Verify `GET /careers/facets` returns correct filter options.
-   **Error Handling:**
    *   Send invalid data to trigger validation errors (422).
    *   Request non-existent resources (404).
    *   (If possible) Simulate server errors to check 500 responses.

### Success Criteria

- ✅ API server starts without errors, and Swagger documentation is accessible and accurate.
- ✅ All defined API endpoints are implemented and functional as per specifications.
- ✅ Input validation effectively rejects invalid requests with appropriate error codes.
- ✅ Error handling middleware catches and formats errors consistently.
- ✅ DOCX file upload via `/api/admin/upload` successfully parses and imports career data using `ImportService`.
- ✅ Preview endpoint `/api/admin/upload/preview` returns parsed data without saving.
- ✅ Public career endpoints (`/careers`, `/careers/:slug`, `/careers/search`) return only "published" careers and correctly handle pagination/filtering.
- ✅ Admin career endpoints allow management of careers with different statuses (draft, published, archived, parsing_issue).
- ✅ Facet data endpoint (`/careers/facets`) provides correct counts for filter options.
- ✅ Slug management (uniqueness, history, redirection) works as intended.
- ✅ User feedback endpoint successfully stores feedback.
- ✅ All API integration tests pass with ≥ 85% coverage for controllers and routes.
- ✅ Logging provides useful information for requests, responses, and errors.

## 6. Artifacts to Commit

- **Branch name**: `sprint-3-gatekeeper`
- **Mandatory files**:
    - `server/src/app.ts` (or `server/src/api/index.ts` - the main Express app setup).
    - `server/src/api/` directory with all subdirectories and files (controllers, middleware, routes, validators, docs).
    - `server/uploads/.gitkeep` (to ensure the directory is tracked).
    - Updated `server/src/server.ts`.
    - API test files in `server/src/api/__tests__/`.
    - Updated `server/package.json` and `server/package-lock.json`.
- **PR Checklist**:
    - [ ] All API endpoints listed in the design are implemented.
    - [ ] File upload and preview functionality is working and integrated with `ImportService`.
    - [ ] Input validation is implemented for all relevant endpoints.
    - [ ] Error handling is robust and provides clear, structured error responses.
    - [ ] API documentation (Swagger/OpenAPI) is generated, accurate, and accessible.
    - [ ] Public endpoints correctly filter by "published" status.
    - [ ] Admin endpoints allow management of career lifecycle (status, archive, delete).
    - [ ] API integration tests achieve target coverage and pass.
    - [ ] Logging is implemented for requests, responses, and errors.
    - [ ] Security considerations (like auth middleware placeholder) are noted/implemented.

--- END OF FILE sprint3_gatekeeper.md ---