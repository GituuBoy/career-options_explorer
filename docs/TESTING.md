# Testing Strategy

> Comprehensive testing approach for the Career Options Explorer application

## Overview

Our testing strategy ensures code quality, reliability, and maintainability across all layers of the application. We employ a multi-layered testing approach with different types of tests for different purposes.

## Testing Pyramid

```
    /\     E2E Tests (Few)
   /  \    - Critical user journeys
  /____\   - Cross-browser testing
 /      \  Integration Tests (Some)
/        \ - API endpoint testing
\        / - Database integration
 \______/  Unit Tests (Many)
          - Pure functions
          - Component logic
          - Business rules
```

## Testing Tools & Frameworks

### Backend Testing
- **Unit Tests**: Jest
- **Integration Tests**: Jest + Supertest
- **Database Testing**: Jest + Test Database
- **Mocking**: Jest mocks
- **Coverage**: Jest coverage reports

### Frontend Testing
- **Unit Tests**: Vitest
- **Component Tests**: React Testing Library
- **Integration Tests**: Vitest + MSW (Mock Service Worker)
- **E2E Tests**: Playwright (future implementation)
- **Visual Testing**: Storybook (future implementation)

### API Testing
- **Manual Testing**: Postman/Thunder Client
- **Automated Testing**: Jest + Supertest
- **Documentation Testing**: Swagger validation

## Backend Testing Strategy

### Unit Tests

#### Parser Module Tests

**Location**: `server/src/parser/__tests__/`

```typescript
// Example: server/src/parser/__tests__/docxToHtml.test.ts
import { convertDocxToHtml } from '../docxToHtml';
import * as fs from 'fs/promises';

describe('DOCX to HTML Conversion', () => {
  test('should convert valid DOCX to HTML', async () => {
    const docxBuffer = await fs.readFile('./fixtures/sample-career.docx');
    const result = await convertDocxToHtml(docxBuffer);
    
    expect(result.html).toContain('<h1>');
    expect(result.messages).toHaveLength(0);
  });

  test('should handle corrupted DOCX files', async () => {
    const invalidBuffer = Buffer.from('invalid docx content');
    
    await expect(convertDocxToHtml(invalidBuffer))
      .rejects.toThrow('Invalid DOCX file');
  });
});
```

#### Repository Tests

**Location**: `server/src/repositories/__tests__/`

```typescript
// Example: server/src/repositories/__tests__/careerRepository.test.ts
import { CareerRepository } from '../careerRepository';
import { setupTestDb, cleanupTestDb } from '../../__tests__/helpers/database';

describe('CareerRepository', () => {
  let repository: CareerRepository;
  
  beforeAll(async () => {
    await setupTestDb();
    repository = new CareerRepository();
  });
  
  afterAll(async () => {
    await cleanupTestDb();
  });
  
  test('should create career with sections', async () => {
    const careerData = {
      title: 'Test Career',
      slug: 'test-career',
      snapshot: 'Test snapshot',
      overview: 'Test overview',
      sections: [
        {
          title: 'Skills',
          content: 'Required skills',
          displayOrder: 1
        }
      ]
    };
    
    const career = await repository.create(careerData);
    
    expect(career.id).toBeDefined();
    expect(career.sections).toHaveLength(1);
  });
});
```

### Integration Tests

#### API Endpoint Tests

**Location**: `server/src/api/__tests__/`

```typescript
// Example: server/src/api/__tests__/careers.test.ts
import request from 'supertest';
import { app } from '../../app';
import { setupTestDb, cleanupTestDb, seedTestData } from '../helpers/database';

describe('Careers API', () => {
  beforeAll(async () => {
    await setupTestDb();
    await seedTestData();
  });
  
  afterAll(async () => {
    await cleanupTestDb();
  });
  
  describe('GET /api/v1/careers', () => {
    test('should return paginated careers', async () => {
      const response = await request(app)
        .get('/api/v1/careers')
        .expect(200);
      
      expect(response.body.success).toBe(true);
      expect(response.body.data.careers).toBeInstanceOf(Array);
      expect(response.body.pagination).toBeDefined();
    });
    
    test('should filter by category', async () => {
      const response = await request(app)
        .get('/api/v1/careers?category=Technology')
        .expect(200);
      
      const careers = response.body.data.careers;
      careers.forEach(career => {
        expect(career.tags.some(tag => tag.category === 'Technology')).toBe(true);
      });
    });
  });
  
  describe('POST /api/v1/admin/upload', () => {
    test('should upload and parse DOCX file', async () => {
      const response = await request(app)
        .post('/api/v1/admin/upload')
        .attach('docxFile', './fixtures/sample-career.docx')
        .set('Authorization', 'Bearer valid-jwt-token')
        .expect(201);
      
      expect(response.body.success).toBe(true);
      expect(response.body.data.career.title).toBeDefined();
    });
  });
});
```

### Test Database Setup

**Location**: `server/src/__tests__/helpers/database.ts`

```typescript
import { Pool } from 'pg';
import { migrate } from 'db-migrate';

let testDb: Pool;

export async function setupTestDb() {
  // Create test database connection
  testDb = new Pool({
    host: process.env.TEST_DB_HOST || 'localhost',
    port: parseInt(process.env.TEST_DB_PORT || '5432'),
    database: process.env.TEST_DB_NAME || 'career_explorer_test',
    user: process.env.TEST_DB_USER || 'career_user',
    password: process.env.TEST_DB_PASSWORD || 'career_password'
  });
  
  // Run migrations
  await migrate.up();
}

export async function cleanupTestDb() {
  // Clean all tables
  await testDb.query('TRUNCATE TABLE careers, sections, tags, career_tags, user_feedback RESTART IDENTITY CASCADE');
}

export async function seedTestData() {
  // Insert test data
  const careerResult = await testDb.query(
    'INSERT INTO careers (title, slug, snapshot, overview, status) VALUES ($1, $2, $3, $4, $5) RETURNING id',
    ['Test Career', 'test-career', 'Test snapshot', 'Test overview', 'published']
  );
  
  // Add more seed data as needed
}
```

## Frontend Testing Strategy

### Component Unit Tests

**Location**: `client/src/components/__tests__/`

```typescript
// Example: client/src/components/__tests__/CareerCard.test.tsx
import { render, screen } from '@testing-library/react';
import { CareerCard } from '../CareerCard';
import { BrowserRouter } from 'react-router-dom';

const mockCareer = {
  id: 1,
  title: 'Data Scientist',
  slug: 'data-scientist',
  snapshot: 'Analyze complex datasets...',
  tags: [
    { name: 'Python', category: 'skill', relevanceScore: 0.9 }
  ]
};

const renderWithRouter = (component: React.ReactElement) => {
  return render(
    <BrowserRouter>
      {component}
    </BrowserRouter>
  );
};

describe('CareerCard', () => {
  test('renders career information correctly', () => {
    renderWithRouter(<CareerCard career={mockCareer} />);
    
    expect(screen.getByText('Data Scientist')).toBeInTheDocument();
    expect(screen.getByText('Analyze complex datasets...')).toBeInTheDocument();
    expect(screen.getByText('Python')).toBeInTheDocument();
  });
  
  test('navigates to career detail on click', () => {
    renderWithRouter(<CareerCard career={mockCareer} />);
    
    const link = screen.getByRole('link');
    expect(link).toHaveAttribute('href', '/careers/data-scientist');
  });
});
```

### Hook Tests

**Location**: `client/src/hooks/__tests__/`

```typescript
// Example: client/src/hooks/__tests__/useCareerSearch.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { useCareerSearch } from '../useCareerSearch';
import { server } from '../../__tests__/mocks/server';
import { rest } from 'msw';

describe('useCareerSearch', () => {
  test('should fetch careers successfully', async () => {
    const { result } = renderHook(() => useCareerSearch({ query: 'data' }));
    
    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });
    
    expect(result.current.careers).toHaveLength(2);
    expect(result.current.error).toBeNull();
  });
  
  test('should handle API errors', async () => {
    server.use(
      rest.get('/api/v1/careers', (req, res, ctx) => {
        return res(ctx.status(500), ctx.json({ error: 'Server error' }));
      })
    );
    
    const { result } = renderHook(() => useCareerSearch({ query: 'data' }));
    
    await waitFor(() => {
      expect(result.current.error).toBeTruthy();
    });
  });
});
```

### Integration Tests with MSW

**Location**: `client/src/__tests__/mocks/`

```typescript
// client/src/__tests__/mocks/handlers.ts
import { rest } from 'msw';

export const handlers = [
  rest.get('/api/v1/careers', (req, res, ctx) => {
    const query = req.url.searchParams.get('q');
    
    return res(
      ctx.json({
        success: true,
        data: {
          careers: [
            {
              id: 1,
              title: 'Data Scientist',
              slug: 'data-scientist',
              snapshot: 'Analyze data...',
              tags: []
            }
          ]
        },
        pagination: {
          currentPage: 1,
          totalPages: 1,
          totalItems: 1
        }
      })
    );
  }),
  
  rest.get('/api/v1/careers/:slug', (req, res, ctx) => {
    return res(
      ctx.json({
        success: true,
        data: {
          career: {
            id: 1,
            title: 'Data Scientist',
            slug: 'data-scientist',
            overview: 'Detailed overview...',
            sections: [],
            tags: []
          }
        }
      })
    );
  })
];
```

## Test Configuration

### Jest Configuration (Backend)

**File**: `server/jest.config.js`

```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/__tests__/**/*.test.ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/__tests__/**/*',
    '!src/migrations/**/*'
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  setupFilesAfterEnv: ['<rootDir>/src/__tests__/setup.ts'],
  testTimeout: 10000
};
```

### Vitest Configuration (Frontend)

**File**: `client/vitest.config.ts`

```typescript
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./src/__tests__/setup.ts'],
    globals: true,
    coverage: {
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/__tests__/',
        '**/*.d.ts'
      ]
    }
  }
});
```

## Test Data Management

### Fixtures

**Location**: `server/src/__tests__/fixtures/`
- `sample-career.docx` - Valid DOCX file for testing
- `invalid-file.txt` - Invalid file for error testing
- `corrupted.docx` - Corrupted DOCX for error handling

### Test Data Factories

```typescript
// server/src/__tests__/factories/careerFactory.ts
export const createCareerData = (overrides = {}) => ({
  title: 'Test Career',
  slug: 'test-career',
  snapshot: 'Test career snapshot',
  overview: 'Test career overview',
  status: 'published',
  sections: [
    {
      title: 'Skills',
      content: 'Required skills',
      displayOrder: 1
    }
  ],
  tags: [
    {
      name: 'JavaScript',
      category: 'skill',
      relevanceScore: 0.8
    }
  ],
  ...overrides
});
```

## Continuous Integration

### GitHub Actions Workflow

**File**: `.github/workflows/test.yml`

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: career_explorer_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: |
          cd server && npm ci
          cd ../client && npm ci
      
      - name: Run server tests
        run: cd server && npm test
        env:
          TEST_DB_HOST: localhost
          TEST_DB_PORT: 5432
          TEST_DB_NAME: career_explorer_test
          TEST_DB_USER: postgres
          TEST_DB_PASSWORD: postgres
      
      - name: Run client tests
        run: cd client && npm test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

## Testing Best Practices

### General Principles
1. **Test Behavior, Not Implementation** - Focus on what the code does, not how
2. **Arrange, Act, Assert** - Structure tests clearly
3. **One Assertion Per Test** - Keep tests focused
4. **Descriptive Test Names** - Make intent clear
5. **Independent Tests** - Tests should not depend on each other

### Backend Testing
1. **Mock External Dependencies** - Database, APIs, file system
2. **Test Error Conditions** - Handle edge cases and failures
3. **Validate Input/Output** - Ensure data integrity
4. **Test Business Logic** - Core functionality thoroughly

### Frontend Testing
1. **Test User Interactions** - Clicks, form submissions, navigation
2. **Test Accessibility** - Screen reader compatibility
3. **Test Responsive Behavior** - Different screen sizes
4. **Mock API Calls** - Use MSW for consistent testing

## Coverage Goals

- **Unit Tests**: 80%+ coverage
- **Integration Tests**: Critical paths covered
- **E2E Tests**: Main user journeys covered

### Coverage Reports

```bash
# Generate coverage reports
cd server && npm run test:coverage
cd client && npm run test:coverage

# View HTML reports
open server/coverage/lcov-report/index.html
open client/coverage/index.html
```

## Running Tests

### Development

```bash
# Run all tests
npm run test

# Run tests in watch mode
npm run test:watch

# Run specific test file
npm test -- careerRepository.test.ts

# Run tests with coverage
npm run test:coverage
```

### CI/CD

```bash
# Run tests in CI mode (no watch, exit on completion)
npm run test:ci
```

## Debugging Tests

### VS Code Configuration

**File**: `.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Jest Tests",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/server/node_modules/.bin/jest",
      "args": ["--runInBand"],
      "cwd": "${workspaceFolder}/server",
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    }
  ]
}
```

---

*This testing strategy ensures comprehensive coverage and maintains code quality throughout the development lifecycle.*