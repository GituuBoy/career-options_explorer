# Career Options Explorer

> A mobile-first web application for exploring career options, designed with an iOS aesthetic.

## 🚀 Quick Start

### Prerequisites
- Node.js 20 LTS
- PostgreSQL 14+
- Git

### Setup
```bash
# Clone the repository
git clone <repository-url>
cd career-options-explorer

# Install dependencies
cd server && npm install
cd ../client && npm install

# Setup database
# See DATABASE.md for detailed setup instructions

# Start development servers
npm run dev  # Starts both client and server
```

## 📋 Project Overview

The Career Options Explorer is a comprehensive web application that helps users discover and explore various career paths. The application features:

- **DOCX Parser**: Intelligent parsing of career description documents
- **Admin Interface**: Content management system for administrators
- **Public Interface**: User-friendly career exploration and search
- **Faceted Search**: Advanced filtering and categorization
- **Mobile-First Design**: iOS-inspired responsive interface

## 🏗️ Architecture

### Tech Stack
- **Frontend**: React 18, TypeScript, TailwindCSS, Vite
- **Backend**: Node.js, Express, TypeScript
- **Database**: PostgreSQL
- **Parsing**: Mammoth.js, Cheerio, Compromise NLP
- **Testing**: Jest, Vitest

### Project Structure
```
career-options-explorer/
├── client/                 # React frontend application
│   ├── src/
│   │   ├── components/     # Reusable UI components
│   │   ├── pages/         # Page components
│   │   ├── hooks/         # Custom React hooks
│   │   └── utils/         # Utility functions
│   └── public/            # Static assets
├── server/                # Express backend application
│   ├── src/
│   │   ├── api/           # API routes and controllers
│   │   ├── parser/        # DOCX parsing module
│   │   ├── repositories/  # Data access layer
│   │   └── services/      # Business logic
│   └── migrations/        # Database migrations
└── docs/                  # Documentation
```

## 📚 Documentation

### Development
- [Getting Started](./docs/GETTING_STARTED.md) - Detailed setup instructions
- [Architecture](./docs/ARCHITECTURE.md) - System design and diagrams
- [API Reference](./docs/API.md) - Complete API documentation
- [Database Schema](./docs/DATABASE.md) - Database design and setup
- [Components Guide](./docs/COMPONENTS.md) - Frontend component documentation
- [Testing Strategy](./docs/TESTING.md) - Testing approaches and guidelines

### Deployment
- [Deployment Guide](./docs/DEPLOYMENT.md) - Production deployment instructions
- [Security Guidelines](./docs/SECURITY.md) - Security best practices
- [Performance Guide](./docs/PERFORMANCE.md) - Optimization strategies

### User Guides
- [User Guide](./docs/USER_GUIDE.md) - End-user documentation
- [Admin Guide](./docs/ADMIN_GUIDE.md) - Administrator documentation
- [Troubleshooting](./docs/TROUBLESHOOTING.md) - Common issues and solutions

### Project Management
- [Contributing](./docs/CONTRIBUTING.md) - Development workflow and guidelines
- [Sprint Documentation](./Development%20documentation/) - Detailed sprint plans
- [Changelog](./CHANGELOG.md) - Version history

## 🎯 Development Sprints

The project is organized into 8 development sprints:

1. **[Sprint 0: Bootstrap](./Development%20documentation/sprint0_bootstrap.md)** - Project setup and toolchain
2. **[Sprint 1: Word Wizard](./Development%20documentation/sprint1_word_wizard.md)** - DOCX parser implementation
3. **[Sprint 2: Vault](./Development%20documentation/sprint2_vault.md)** - Database schema and repository layer
4. **[Sprint 3: Gatekeeper](./Development%20documentation/sprint3_gatekeeper.md)** - Express API development
5. **[Sprint 4: Scribe](./Development%20documentation/sprint4_scribe.md)** - Admin UI implementation
6. **[Sprint 5: Explorer](./Development%20documentation/sprint5_explorer.md)** - Public UI development
7. **[Sprint 6: Facet](./Development%20documentation/sprint6_facet.md)** - Advanced search and performance
8. **[Sprint 7: Polish](./Development%20documentation/sprint7_polish.md)** - Accessibility and production readiness

## 🚦 Current Status

- [x] Project documentation and architecture design
- [ ] Sprint 0: Development environment setup
- [ ] Sprint 1: DOCX parser module
- [ ] Sprint 2: Database implementation
- [ ] Sprint 3: API development
- [ ] Sprint 4: Admin interface
- [ ] Sprint 5: Public interface
- [ ] Sprint 6: Advanced features
- [ ] Sprint 7: Production polish

## 🤝 Contributing

Please read [CONTRIBUTING.md](./docs/CONTRIBUTING.md) for details on our code of conduct and the process for submitting pull requests.

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🆘 Support

If you encounter any issues:
1. Check the [Troubleshooting Guide](./docs/TROUBLESHOOTING.md)
2. Search existing [GitHub Issues](../../issues)
3. Create a new issue with detailed information

## 🔗 Quick Links

- [Live Demo](https://career-explorer-demo.com) *(Coming Soon)*
- [API Documentation](./docs/API.md)
- [Component Storybook](https://storybook.career-explorer.com) *(Coming Soon)*
- [Project Roadmap](../../projects)

---

**Built with ❤️ for career exploration and discovery**