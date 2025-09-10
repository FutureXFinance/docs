# 🚀 FutureXFinance Quick Start Guide

> Get FutureXFinance running locally in **under 5 minutes**!

## ⚡ **Prerequisites**

### **Required Software**
```bash
# Node.js 18+ and npm
node --version  # Should be 18.0.0+
npm --version   # Should be 9.0.0+

# PostgreSQL 15+
psql --version  # Should be 15.0+

# Redis 7+
redis-server --version  # Should be 7.0+

# Git
git --version
```

### **Development Tools (Recommended)**
- **VS Code** with extensions: ESLint, Prettier, TypeScript
- **Postman** or **Insomnia** for API testing
- **pgAdmin** or **TablePlus** for database management

## 🏗️ **Quick Setup (5 Minutes)**

### **Step 1: Clone Repositories**
```bash
# Create workspace directory
mkdir futurexfinance-dev
cd futurexfinance-dev

# Clone all repositories
git clone https://github.com/FutureXFinance/frontend.git
git clone https://github.com/FutureXFinance/backend.git
git clone https://github.com/FutureXFinance/infrastructure.git
git clone https://github.com/FutureXFinance/docs.git
```

### **Step 2: Start Backend Services**
```bash
cd backend

# Install dependencies
npm install

# Setup environment variables
cp .env.example .env.local

# Start PostgreSQL and Redis (if not running)
# macOS with Homebrew:
brew services start postgresql
brew services start redis

# Ubuntu/Debian:
sudo systemctl start postgresql
sudo systemctl start redis

# Setup database
npm run db:setup
npm run db:migrate

# Start development server
npm run dev
```

**Backend will be running at**: `http://localhost:3001`

### **Step 3: Start Frontend Application**
```bash
cd ../frontend

# Install dependencies
npm install

# Setup environment variables  
cp .env.example .env.local

# Start development server
npm run dev
```

**Frontend will be running at**: `http://localhost:3000`

### **Step 4: Verify Setup**
```bash
# Test backend health
curl http://localhost:3001/health

# Expected response:
# {"status":"ok","database":"connected","redis":"connected"}

# Test frontend
open http://localhost:3000
# Should show FutureXFinance dashboard
```

## 🔧 **Configuration**

### **Environment Variables**

#### **Backend (.env.local)**
```bash
# Database
DATABASE_URL="postgresql://username:password@localhost:5432/futurexfinance_dev"
REDIS_URL="redis://localhost:6379"

# Authentication  
JWT_SECRET="your-super-secret-jwt-key-here"
JWT_EXPIRES_IN="7d"

# External APIs
STRIPE_SECRET_KEY="sk_test_..."
PLAID_CLIENT_ID="your-plaid-client-id"
PLAID_SECRET="your-plaid-secret"

# Email Service
SENDGRID_API_KEY="SG...."
FROM_EMAIL="noreply@futurexfinance.com"

# Development
NODE_ENV="development"
PORT=3001
LOG_LEVEL="debug"
```

#### **Frontend (.env.local)**
```bash
# API Configuration
NEXT_PUBLIC_API_URL="http://localhost:3001"
NEXT_PUBLIC_WS_URL="ws://localhost:3001"

# Authentication
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="your-nextauth-secret-here"

# Feature Flags
NEXT_PUBLIC_ENABLE_CRYPTO="true"
NEXT_PUBLIC_ENABLE_INTERNATIONAL="true"

# Analytics (optional for development)
NEXT_PUBLIC_GOOGLE_ANALYTICS="G-XXXXXXXXXX"
NEXT_PUBLIC_HOTJAR_ID="12345"

# Development
NODE_ENV="development"
```

## 📊 **Development Workflow**

### **Daily Development Process**

#### **1. Start Development Environment**
```bash
# Terminal 1: Backend
cd backend
npm run dev

# Terminal 2: Frontend  
cd frontend
npm run dev

# Terminal 3: Database monitoring (optional)
cd backend
npm run db:studio
```

#### **2. Code Quality Checks**
```bash
# Run before committing
npm run lint        # ESLint check
npm run type-check  # TypeScript check
npm run test        # Unit tests
npm run format      # Prettier formatting
```

#### **3. Database Operations**
```bash
# Create new migration
npm run db:migrate:create -- --name add_user_preferences

# Apply migrations
npm run db:migrate

# Reset database (development only!)
npm run db:reset

# Seed test data
npm run db:seed
```

### **Testing Your Changes**

#### **Backend API Testing**
```bash
# Unit tests
npm run test

# Integration tests
npm run test:integration

# Test specific endpoint
curl -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'
```

#### **Frontend Testing**
```bash
# Component tests
npm run test

# E2E tests  
npm run test:e2e

# Visual regression tests
npm run test:visual
```

## 🔍 **Common Issues & Solutions**

### **Database Connection Issues**
```bash
# Check PostgreSQL is running
pg_isready

# Check connection
psql -h localhost -p 5432 -U your_username -d futurexfinance_dev

# Reset connection pool
npm run db:pool:reset
```

### **Port Already in Use**
```bash
# Check what's using the port
lsof -i :3000  # or :3001

# Kill the process
kill -9 <PID>

# Or use different ports
PORT=3002 npm run dev
```

### **Node Modules Issues**
```bash
# Clear npm cache
npm cache clean --force

# Delete and reinstall
rm -rf node_modules package-lock.json
npm install

# Use exact versions
npm ci
```

## 🧪 **Test Data & Demo Accounts**

### **Pre-seeded Test Users**
```javascript
// Login credentials for testing
{
  email: "merchant@demo.com",
  password: "Demo123!",
  role: "merchant"
},
{
  email: "admin@demo.com", 
  password: "Admin123!",
  role: "admin"
}
```

### **Test Payment Scenarios**
```bash
# Seed test transactions
npm run db:seed:transactions

# Generate test webhooks
npm run test:webhooks

# Mock external APIs
npm run dev:mock-apis
```

## 📱 **API Development**

### **Creating New Endpoints**

#### **1. Define Route**
```typescript
// src/routes/payments.ts
import { Router } from 'express';
import { PaymentController } from '../controllers/payment.controller';

const router = Router();
const paymentController = new PaymentController();

router.post('/transfer', paymentController.createTransfer);

export default router;
```

#### **2. Implement Controller**
```typescript
// src/controllers/payment.controller.ts
import { Request, Response } from 'express';
import { PaymentService } from '../services/payment.service';

export class PaymentController {
  async createTransfer(req: Request, res: Response) {
    try {
      const transfer = await PaymentService.createTransfer(req.body);
      res.json({ success: true, data: transfer });
    } catch (error) {
      res.status(400).json({ success: false, error: error.message });
    }
  }
}
```

#### **3. Add Tests**
```typescript
// src/__tests__/payment.controller.test.ts
describe('PaymentController', () => {
  it('should create transfer successfully', async () => {
    const response = await request(app)
      .post('/api/payments/transfer')
      .send({
        amount: 1000,
        currency: 'USD',
        recipient: 'user@example.com'
      })
      .expect(200);
    
    expect(response.body.success).toBe(true);
  });
});
```

## 🎯 **Next Steps**

### **After Setup is Complete:**

1. **📖 Read Architecture Docs**: [technical/ARCHITECTURE.md](../technical/ARCHITECTURE.md)
2. **🔐 Setup Security**: [development/SECURITY_SETUP.md](SECURITY_SETUP.md)
3. **🧪 Write Tests**: [development/TESTING.md](TESTING.md)
4. **🚀 Deploy**: [operations/DEPLOYMENT.md](../operations/DEPLOYMENT.md)

### **Development Resources**
- **API Documentation**: `http://localhost:3001/api/docs` (Swagger UI)
- **Database Admin**: `http://localhost:3001/db/studio` (Prisma Studio)
- **Email Testing**: `http://localhost:3001/mail/preview` (Email previews)
- **Redis Admin**: `http://localhost:3001/redis/insight` (Redis insights)

## 🤝 **Getting Help**

### **Documentation**
- **API Reference**: [technical/API_REFERENCE.md](../technical/API_REFERENCE.md)
- **Troubleshooting**: [development/TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- **Code Style**: [development/CODE_STYLE.md](CODE_STYLE.md)

### **Community**
- **Slack**: `#dev-support` channel
- **GitHub Discussions**: Ask questions and share ideas
- **Office Hours**: Tuesdays 2-4 PM PT for live help

---

**🎉 You're all set!** Your FutureXFinance development environment is ready. Happy coding! 🚀
