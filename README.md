# My App - Full Stack Application

A comprehensive full-stack application with a React frontend and Deno backend, featuring PWA capabilities, authentication, and business management.

## 🏗️ Project Structure

```
My App/
├── mercury-app/          # Frontend (React + TypeScript + Vite)
├── Backend/             # Backend (Deno + Supabase)
├── .gitmodules          # Git submodule configuration
└── README.md            # This file
```

## 🚀 Quick Start

### Prerequisites
- Node.js 18+ (for frontend)
- Deno 1.40+ (for backend)
- Git with submodule support

### Initial Setup

1. **Clone the repository with submodules:**
   ```bash
   git clone --recursive https://github.com/your-username/my-app.git
   cd my-app
   ```

2. **If you already cloned without submodules:**
   ```bash
   git submodule update --init --recursive
   ```

### Frontend Setup (mercury-app)

```bash
cd mercury-app
npm install
npm run dev
```

**Available Scripts:**
- `npm run dev` - Start development server
- `npm run build` - Build for production
- `npm run preview` - Preview production build
- `npm run test` - Run tests

### Backend Setup (Backend)

```bash
cd Backend
deno task start
```

**Available Scripts:**
- `deno task start` - Start development server
- `deno task test` - Run tests
- `deno task migrate` - Run database migrations

## 🔧 Development Workflow

### Working with Submodules

**Update all submodules to latest:**
```bash
git submodule update --remote
```

**Work on a specific component:**
```bash
# Frontend development
cd mercury-app
git checkout main
git pull origin main
# Make your changes
git add .
git commit -m "Your changes"
git push origin main

# Update main project
cd ..
git add mercury-app
git commit -m "Update mercury-app with latest changes"
```

**Clone for new team members:**
```bash
git clone --recursive <repository-url>
```

## 📱 Features

### Frontend (mercury-app)
- **Framework**: React 18 + TypeScript
- **Build Tool**: Vite
- **UI**: Tailwind CSS + Radix UI
- **PWA**: Service Worker + Offline Support
- **Authentication**: Supabase Auth
- **State Management**: TanStack Query
- **Testing**: Vitest + Testing Library

### Backend (Backend)
- **Runtime**: Deno
- **Database**: Supabase (PostgreSQL)
- **Authentication**: Supabase Auth
- **API**: RESTful endpoints
- **Security**: CSRF protection, XSS prevention
- **Monitoring**: Built-in logging and metrics

## 🛠️ Technology Stack

### Frontend
- React 18 with TypeScript
- Vite for build tooling
- Tailwind CSS for styling
- Radix UI for components
- TanStack Query for data fetching
- Supabase for authentication
- PWA capabilities

### Backend
- Deno runtime
- Supabase for database and auth
- TypeScript
- SQL migrations
- RESTful API design
- Security middleware

## 📦 Deployment

### Frontend Deployment
The frontend is configured for Vercel deployment with:
- Automatic builds on push
- Environment variable configuration
- PWA optimization
- CDN distribution

### Backend Deployment
The backend supports multiple deployment options:
- Deno Deploy
- Docker containers
- Traditional VPS deployment

## 🔐 Environment Variables

### Frontend (.env)
```env
VITE_SUPABASE_URL=your_supabase_url
VITE_SUPABASE_ANON_KEY=your_supabase_anon_key
VITE_BACKEND_URL=your_backend_url
```

### Backend (.env)
```env
SUPABASE_URL=your_supabase_url
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
DATABASE_URL=your_database_url
```

## 🧪 Testing

### Frontend Testing
```bash
cd mercury-app
npm run test
```

### Backend Testing
```bash
cd Backend
deno task test
```

## 📚 Documentation

- [Frontend Documentation](./mercury-app/README.md)
- [Backend Documentation](./Backend/README.md)
- [PWA Guide](./mercury-app/docs/PWA_README.md)
- [Security Guide](./Backend/docs/SECURITY_GUIDE.md)

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

### Development Guidelines
- Follow TypeScript best practices
- Write tests for new features
- Update documentation
- Follow conventional commits

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 🆘 Support

For support and questions:
- Check the documentation in each component
- Review existing issues
- Create a new issue with detailed information

## 🔄 Submodule Management

### Common Commands

**Update all submodules:**
```bash
git submodule update --remote --merge
```

**Update specific submodule:**
```bash
git submodule update --remote mercury-app
```

**Check submodule status:**
```bash
git submodule status
```

**Remove a submodule:**
```bash
git submodule deinit mercury-app
git rm mercury-app
```

## 🚀 Getting Help

1. **Check component READMEs** for specific setup instructions
2. **Review the documentation** in the `docs/` folders
3. **Check existing issues** on GitHub
4. **Create a new issue** with detailed error information

---

**Happy coding! 🎉**
