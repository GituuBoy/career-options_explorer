Sprint 0: Bootstrap

1 · Narrative Goal

In this foundational sprint we’re setting up the development environment and project structure that will serve as the launch‑pad for our Career Options Explorer application. By the end of these two days we’ll have a clean, organised code‑base with the basic tool‑chain configured, Git version‑control established with pre‑commit hooks, and an empty but functional React application rendering in the browser, styled with our initial iOS‑inspired theme.

This sprint is about building a solid foundation—ensuring our development tools are properly configured, our TypeScript setup is working correctly, and our project structure follows best practices. While there won’t be much visible functionality yet, this groundwork will enable smooth, efficient development in subsequent sprints.

2 · Back‑of‑the‑Envelope Design

graph TD
    subgraph "Project Structure"
        Root["/"] --> Client["client/"]
        Root --> Server["server/"]
        Root --> Docs["docs/"]
        Client --> ClientSrc["src/"]
        Client --> ClientPublic["public/"]
        Client --> ClientCfg["config files (tailwind, postcss, vite, tsconfig)"]
        Server --> ServerSrc["src/"]
        Server --> ServerCfg["config files (tsconfig, jest)"]
        Server --> ServerEnv[".env (git‑ignored)"]
    end
    subgraph "Core Tech Stack"
        Client -.-> React["React 18"]
        Client -.-> Vite["Vite"]
        Client -.-> TS_Client["TypeScript"]
        Client -.-> Tailwind["TailwindCSS (iOS Theme)"]
        Server -.-> Node["Node 20 LTS"]
        Server -.-> Express["Express"]
        Server -.-> TS_Server["TypeScript"]
        Server -.-> PostgreSQL["PostgreSQL (via pg driver)"]
    end

3 · Task Checklist

3.1 Project Initialisation

# Create project root directory
mkdir -p career-options-explorer
cd career-options-explorer

# Initialise Git repository and create .gitignore
git init
echo "node_modules/\n.DS_Store\n# Environment files\n.env\n.env.*\n!/.env.example\n\n# Build output\ndist/\nbuild/\n\n# VSCode\n.vscode/\n\n# Logs\n*.log\nnpm-debug.log*\nyarn-debug.log*\nyarn-error.log*\n\n# Coverage\ncoverage/" > .gitignore

# Create initial README
cat <<'EOF' > README.md
# Career Options Explorer

A mobile‑first web application for exploring career options, designed with an iOS aesthetic.

## Core Technologies
* **Frontend** : React 18, Vite, TypeScript, TailwindCSS
* **Backend** : Node 20 LTS, Express, TypeScript
* **Database**: PostgreSQL
* **DOCX Parsing** : Mammoth.js, Cheerio
* **Tagging** : Compromise, TF‑IDF
* **Testing** : Jest / Vitest
EOF

# Create docs holder
mkdir -p docs && touch docs/.gitkeep

3.2 Server Setup (Node + Express + TypeScript)

mkdir -p server && cd server
npm init -y

# Runtime dependencies
npm i express cors body-parser pg slugify dotenv

# Dev dependencies
npm i -D typescript ts-node-dev @types/node @types/express \
       @types/cors @types/body-parser @types/pg @types/slugify \
       jest ts-jest @types/jest supertest @types/supertest

# Generate tsconfig
npx tsc --init --rootDir src --outDir dist --esModuleInterop --resolveJsonModule \
        --lib es2020,dom --module commonjs --target es2020 --strict true \
        --skipLibCheck true --forceConsistentCasingInFileNames true

tsconfig.json (minimal):

{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "**/*.test.ts", "src/__tests__", "src/parser/__tests__"]
}

# Directory scaffold
mkdir -p src/__tests__ src/api src/config src/db src/models \
         src/parser src/repositories src/routes src/services src/utils

Create .env.example and .env (remember .env is git‑ignored):

echo "PORT=3001\nDB_HOST=localhost\nDB_PORT=5432\nDB_USER=career_user\nDB_PASSWORD=career_password\nDB_NAME=career_explorer\nNODE_ENV=development" > .env.example
cp .env.example .env

src/server.ts:

import express from 'express';
import cors from 'cors';
import bodyParser from 'body-parser';
import dotenv from 'dotenv';

dotenv.config();

const app = express();
const PORT = process.env.PORT || 3001;

app.use(cors());
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

app.get('/api/health', (_req, res) => {
  res.json({ status: 'ok', message: 'Server is healthy and running!' });
});

app.listen(PORT, () => {
  console.log(`Server listening on http://localhost:${PORT}`);
});

export default app;

Add scripts to server/package.json:

"scripts": {
  "start": "node dist/server.js",
  "dev": "ts-node-dev --respawn --transpile-only src/server.ts",
  "build": "tsc",
  "test": "jest --coverage"
}

jest.config.js:

module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['**/__tests__/**/*.test.ts?(x)'],
  collectCoverage: true,
  coverageDirectory: 'coverage',
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/server.ts',
    '!src/db/index.ts',
    '!src/db/seed.ts',
    '!src/parser/cli.ts',
    '!src/parser/watch.ts'
  ],
  setupFiles: ['dotenv/config']
};

src/__tests__/server.test.ts:

import request from 'supertest';
import app from '../server';

describe('GET /api/health', () => {
  it('returns 200 OK', async () => {
    const res = await request(app).get('/api/health');
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ status: 'ok', message: 'Server is healthy and running!' });
  });
});

3.3 Client Setup (React + Vite + TS + Tailwind)

cd ..   # back to root
npm create vite@latest client -- --template react-ts
cd client && npm install

Install Tailwind:

npm i -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

tailwind.config.js (excerpt):

import colors from 'tailwindcss/colors';

export default {
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx}'],
  theme: {
    extend: {
      colors: {
        primary: colors.blue,
        secondary: colors.gray,
        'ios-blue': '#007AFF',
        'ios-green': '#34C759',
        'ios-red':   '#FF3B30',
        'ios-gray': {
          100:'#F2F2F7',200:'#E5E5EA',300:'#D1D1D6',400:'#C7C7CC',500:'#AEAEB2',600:'#8E8E93',DEFAULT:'#8E8E93'
        },
        'ios-light-background':'#F2F2F7',
        'ios-dark-background':'#000000',
        'ios-light-text':'#000000',
        'ios-dark-text':'#FFFFFF'
      },
      borderRadius:{ ios:'10px' },
      fontFamily:{ sans:['-apple-system','BlinkMacSystemFont','"Segoe UI"','Roboto','"Helvetica Neue"','Arial','sans-serif'] }
    }
  },
  plugins: []
};

src/index.css:

@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  @apply bg-ios-light-background text-ios-light-text;
}

Directory scaffold:

mkdir -p src/{components,pages,assets,hooks,services,types,utils,config}

src/App.tsx:

import './App.css';

function App() {
  return (
    <div className="min-h-screen bg-ios-light-background">
      <header className="bg-white/80 backdrop-blur-md shadow-sm sticky top-0 z-50">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div className="flex items-center justify-between h-16">
            <h1 className="text-xl font-semibold text-ios-blue">Career Options Explorer</h1>
          </div>
        </div>
      </header>

      <main className="py-6">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div className="bg-white shadow-md rounded-ios p-6">
            <p className="text-gray-700">Welcome to the Career Options Explorer. More content coming soon!</p>
          </div>
        </div>
      </main>

      <footer className="py-8 text-center border-t border-ios-gray-200">
        <p className="text-sm text-ios-gray-500">
          © {new Date().getFullYear()} Career Options Explorer
        </p>
      </footer>
    </div>
  );
}

export default App;

vite.config.ts:

import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true
      }
    }
  }
});

Vitest tests (src/App.test.tsx):

import { render, screen } from '@testing-library/react';
import App from './App';

describe('App', () => {
  it('renders heading', () => {
    render(<App />);
    expect(screen.getByText(/Career Options Explorer/i)).toBeInTheDocument();
  });
});

Add scripts and install Vitest:

npm i -D @testing-library/react @testing-library/jest-dom jsdom vitest @vitest/coverage-v8

vitest.config.ts:

import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/setupTests.ts',
    coverage: { reporter: ['text','json','html'] }
  }
});

3.4 Root Monorepo & Developer Experience

Create root package.json and workspaces:

{
  "name": "career-options-explorer-monorepo",
  "version": "0.1.0",
  "private": true,
  "workspaces": ["client","server"],
  "scripts": {
    "install:all": "npm install -ws --if-present && npm run install:server && npm run install:client",
    "install:client": "cd client && npm install",
    "install:server": "cd server && npm install",
    "dev": "concurrently \"npm:dev:server\" \"npm:dev:client\"",
    "dev:client": "cd client && npm run dev",
    "dev:server": "cd server && npm run dev",
    "build:all": "npm run build:client && npm run build:server",
    "build:client": "cd client && npm run build",
    "build:server": "cd server && npm run build",
    "test:all": "npm run test:client && npm run test:server",
    "test:client": "cd client && npm run test",
    "test:server": "cd server && npm run test",
    "lint": "npm run lint:client && npm run lint:server",
    "lint:client": "cd client && npm run lint",
    "lint:server": "cd server && echo \"ESLint for server not yet configured, run manually if needed\"",
    "prepare": "husky install"
  },
  "devDependencies": {
    "concurrently": "^8.2.2",
    "husky": "^8.0.0",
    "lint-staged": "^15.0.0"
  }
}

npm i -D concurrently husky lint-staged
npx husky init
npx husky add .husky/pre-commit "npx lint-staged"

Add lint‑staged in root package.json:

"lint-staged": {
  "client/src/**/*.{js,jsx,ts,tsx}": [
    "cd client && npm run lint -- --fix",
    "cd client && prettier --write"
  ],
  "server/src/**/*.ts": [
    "cd server && npm run lint -- --fix",
    "cd server && prettier --write"
  ],
  "*.{json,md}": ["prettier --write"]
}

3.5 VS Code Workspace

career-options-explorer.code-workspace:

{
  "folders": [
    { "path": "client" },
    { "path": "server" },
    { "path": "." }
  ],
  "settings": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    "editor.codeActionsOnSave": {
      "source.fixAll.eslint": "explicit"
    },
    "typescript.tsdk": "node_modules/typescript/lib",
    "jest.jestCommandLine": "npm test --",
    "vitest.commandLine": "npm run test --",
    "files.associations": { "*.css": "tailwindcss" }
  },
  "extensions": {
    "recommendations": [
      "dbaeumer.vscode-eslint",
      "esbenp.prettier-vscode",
      "bradlc.vscode-tailwindcss",
      "ms-vscode.vscode-typescript-next",
      "orta.vscode-jest",
      "vitest.explorer",
      "mtxr.sqltools",
      "mtxr.sqltools-driver-pg",
      "GitHub.copilot",
      "eamodio.gitlens",
      "Gruntfuggly.todo-tree",
      "wayou.vscode-todo-highlight"
    ]
  }
}

4 · Creative Add‑Ons (Stretch)

The stretch goals from the original prompt have been folded into the core checklist, ensuring a stronger base.

5 · Run & Verify

5.1 Installation & Build

npm run install:all   # install everything
npm run build:all     # build client & server

5.2 Start Development Servers

npm run dev           # runs both servers concurrently

Open http://localhost:3000 → React welcome screen.

Open http://localhost:3001/api/health → {"status":"ok","message":"Server is healthy and running!"}.

5.3 Running Tests

# Server (Jest)
cd server && npm test

# Client (Vitest)
cd client && npm test

5.4 Git Pre‑commit Hook

Introduce an ESLint error in a .ts(x) file.

git add . → git commit -m "test: pre‑commit".

The commit should be blocked/fixed automatically.

5.5 Success Criteria

✅ Dependencies install without error.

✅ npm run build:all succeeds.

✅ npm run dev starts both servers, React loads, health endpoint replies.

✅ All unit tests pass.

✅ Husky + lint‑staged block or fix bad code.

✅ Project structure matches design; VS Code workspace operates smoothly.

6 · Artifacts to Commit

Branch : sprint-0-bootstrap

Mandatory files:

.gitattributes (if applicable)

.gitignore

README.md

package.json / package-lock.json (root, client, server)

career-options-explorer.code-workspace

.husky/ pre‑commit hook

client/ (entire Vite setup)

server/ (entire Express setup)

docs/.gitkeep

PR Checklist:

Project builds & runs.

Health endpoint OK.

Tests green.

Husky hooks active.

README filled.

No secrets or unwanted files committed.

