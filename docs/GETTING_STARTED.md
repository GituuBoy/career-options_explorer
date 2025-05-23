# Getting Started

> Complete setup guide for the Career Options Explorer development environment

## Prerequisites

Before you begin, ensure you have the following installed on your system:

### Required Software
- **Node.js 20 LTS** - [Download here](https://nodejs.org/)
- **PostgreSQL 14+** - [Download here](https://www.postgresql.org/download/)
- **Git** - [Download here](https://git-scm.com/downloads)

### Recommended Tools
- **VS Code** with extensions:
  - PostgreSQL (ms-ossdata.vscode-postgresql)
  - TypeScript and JavaScript Language Features
  - Tailwind CSS IntelliSense
  - ESLint
  - Prettier

### Verify Installation

```bash
# Check Node.js version (should be 20.x)
node --version

# Check npm version
npm --version

# Check PostgreSQL version
psql --version

# Check Git version
git --version
```

## Project Setup

### 1. Clone the Repository

```bash
# Clone the repository
git clone <repository-url>
cd career-options-explorer

# Create a new branch for your work
git checkout -b feature/your-feature-name
```

### 2. Install Dependencies

```bash
# Install server dependencies
cd server
npm install

# Install client dependencies
cd ../client
npm install

# Return to project root
cd ..
```

### 3. Database Setup

#### Create Database and User

```bash
# Connect to PostgreSQL as superuser
psql -U postgres

# Create database user
CREATE USER career_user WITH ENCRYPTED PASSWORD 'career_password';

# Create database
CREATE DATABASE career_explorer OWNER career_user;

# Grant privileges
GRANT ALL PRIVILEGES ON DATABASE career_explorer TO career_user;

# Exit psql
\q
```

#### Verify Database Connection

```bash
# Test connection
psql -U career_user -d career_explorer -h localhost

# If successful, you should see the PostgreSQL prompt
# Exit with \q
```

### 4. Environment Configuration

#### Server Environment

Create `server/.env` file:

```bash
cd server
cp .env.example .env
```

Edit `server/.env` with your configuration:

```env
# Database Configuration
DB_HOST=localhost
DB_PORT=5432
DB_NAME=career_explorer
DB_USER=career_user
DB_PASSWORD=career_password

# Server Configuration
PORT=3001
NODE_ENV=development
API_VERSION=v1

# CORS Configuration
CORS_ORIGIN=http://localhost:3000

# File Upload Configuration
UPLOAD_DIR=./uploads
MAX_FILE_SIZE_MB=10

# Pagination Configuration
PAGINATION_DEFAULT_LIMIT=12
PAGINATION_MAX_LIMIT=100

# JWT Configuration (for development)
JWT_SECRET=your-super-secret-jwt-key-change-in-production
JWT_EXPIRES_IN=24h

# Logging Configuration
LOG_LEVEL=debug
```

#### Client Environment

Create `client/.env` file:

```bash
cd client
cp .env.example .env
```

Edit `client/.env`:

```env
# API Configuration
VITE_API_BASE_URL=http://localhost:3001/api/v1

# App Configuration
VITE_APP_NAME=Career Options Explorer
VITE_APP_VERSION=1.0.0

# Development Configuration
VITE_DEV_MODE=true
```

### 5. Database Migration

```bash
# Navigate to server directory
cd server

# Run database migrations
npm run migrate:up

# Seed database with sample data (optional)
npm run seed:dev
```

### 6. Create Upload Directory

```bash
# Create uploads directory for file storage
mkdir -p server/uploads
```

## Running the Application

### Development Mode

#### Option 1: Run Both Services Simultaneously

```bash
# From project root
npm run dev
```

This will start:
- Backend server on `http://localhost:3001`
- Frontend development server on `http://localhost:3000`

#### Option 2: Run Services Separately

**Terminal 1 - Backend:**
```bash
cd server
npm run dev
```

**Terminal 2 - Frontend:**
```bash
cd client
npm run dev
```

### Verify Setup

1. **Backend Health Check:**
   ```bash
   curl http://localhost:3001/api/v1/health
   ```
   Should return: `{"status": "ok", "timestamp": "..."}`

2. **Frontend Access:**
   Open `http://localhost:3000` in your browser

3. **Database Connection:**
   Check server logs for successful database connection

## Development Workflow

### Code Quality Tools

```bash
# Run linting
cd server && npm run lint
cd client && npm run lint

# Fix linting issues
cd server && npm run lint:fix
cd client && npm run lint:fix

# Format code
cd server && npm run format
cd client && npm run format
```

### Testing

```bash
# Run server tests
cd server && npm test

# Run client tests
cd client && npm test

# Run tests in watch mode
cd server && npm run test:watch
cd client && npm run test:watch

# Generate coverage report
cd server && npm run test:coverage
cd client && npm run test:coverage
```

### Database Operations

```bash
# Create new migration
cd server
npm run migrate:create migration-name

# Run migrations
npm run migrate:up

# Rollback migrations
npm run migrate:down

# Reset database (caution: destroys all data)
npm run migrate:reset
```

## Common Issues & Solutions

### Database Connection Issues

**Problem:** `ECONNREFUSED` or authentication errors

**Solutions:**
1. Verify PostgreSQL is running:
   ```bash
   # Windows
   net start postgresql-x64-14
   
   # macOS
   brew services start postgresql
   
   # Linux
   sudo systemctl start postgresql
   ```

2. Check database credentials in `.env`
3. Verify database and user exist
4. Check PostgreSQL logs for detailed errors

### Port Already in Use

**Problem:** `EADDRINUSE` error

**Solutions:**
1. Kill process using the port:
   ```bash
   # Find process using port 3001
   lsof -i :3001
   
   # Kill the process
   kill -9 <PID>
   ```

2. Change port in `.env` file

### Node Modules Issues

**Problem:** Module resolution or version conflicts

**Solutions:**
1. Clear npm cache:
   ```bash
   npm cache clean --force
   ```

2. Delete node_modules and reinstall:
   ```bash
   rm -rf node_modules package-lock.json
   npm install
   ```

### File Upload Issues

**Problem:** File upload fails or directory errors

**Solutions:**
1. Ensure upload directory exists:
   ```bash
   mkdir -p server/uploads
   ```

2. Check file permissions:
   ```bash
   chmod 755 server/uploads
   ```

## Development Tips

### VS Code Configuration

Create `.vscode/settings.json`:

```json
{
  "typescript.preferences.importModuleSpecifier": "relative",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "files.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.env": false
  }
}
```

### Debugging

#### Backend Debugging
1. Add breakpoints in VS Code
2. Use the "Node.js" debug configuration
3. Or use `console.log()` and check server logs

#### Frontend Debugging
1. Use browser developer tools
2. React Developer Tools extension
3. Check network tab for API calls

### Hot Reloading

- **Backend**: Uses `ts-node-dev` for automatic restart on file changes
- **Frontend**: Uses Vite's built-in hot module replacement

### API Testing

Use tools like:
- **Postman** - GUI-based API testing
- **curl** - Command-line testing
- **Thunder Client** - VS Code extension

Example API test:
```bash
# Test career listing
curl "http://localhost:3001/api/v1/careers?limit=5"

# Test with authentication (replace <token>)
curl -H "Authorization: Bearer <token>" \
     "http://localhost:3001/api/v1/admin/careers"
```

## Next Steps

1. **Explore the Codebase:**
   - Review [Architecture Documentation](./ARCHITECTURE.md)
   - Check [API Documentation](./API.md)
   - Read [Component Guide](./COMPONENTS.md)

2. **Start Development:**
   - Pick a task from the project board
   - Create a feature branch
   - Follow the [Contributing Guidelines](./CONTRIBUTING.md)

3. **Learn the Domain:**
   - Review sample career documents
   - Understand the parsing pipeline
   - Explore the admin interface

## Getting Help

If you encounter issues:

1. Check the [Troubleshooting Guide](./TROUBLESHOOTING.md)
2. Search existing GitHub issues
3. Ask questions in team chat
4. Create a detailed issue with:
   - Steps to reproduce
   - Expected vs actual behavior
   - Environment details
   - Error messages and logs

---

**Happy coding! 🚀**