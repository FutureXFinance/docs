# 📋 FutureXFinance Testing Guide

> **Comprehensive testing strategy** ensuring 99.9% reliability and enterprise-grade quality for financial operations.

## 🎯 **Testing Philosophy**

### **Quality Gates**
Every code change must pass through multiple quality gates before reaching production:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Unit Tests    │ -> │Integration Tests│ -> │    E2E Tests    │ -> │Security Tests   │
│   >90% Coverage │    │  API & Database │    │  User Journeys  │    │  Vulnerability  │
│   <500ms        │    │  <2s runtime    │    │  <5min runtime  │    │  Zero Critical  │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

### **Testing Pyramid**
```
                    /\
                   /  \
                  /E2E \         <- 10% (Critical user journeys)
                 /      \
                /________\
               /          \
              /Integration \     <- 20% (API contracts, database)
             /              \
            /________________\
           /                  \
          /    Unit Tests      \   <- 70% (Business logic, utilities)
         /                      \
        /________________________\
```

## 🧪 **Unit Testing**

### **Testing Stack**
- **Framework**: Jest + Testing Library
- **Coverage**: 90%+ line coverage required  
- **Mocking**: MSW (Mock Service Worker) for API calls
- **Snapshots**: React component snapshots for UI consistency

### **Unit Test Examples**

#### **Payment Service Tests**
```typescript
// payment.service.test.ts
describe('PaymentService', () => {
  let paymentService: PaymentService;
  let mockLedgerService: jest.Mocked<LedgerService>;
  let mockFraudService: jest.Mocked<FraudService>;

  beforeEach(() => {
    mockLedgerService = createMockLedgerService();
    mockFraudService = createMockFraudService();
    paymentService = new PaymentService(mockLedgerService, mockFraudService);
  });

  describe('processPayment', () => {
    it('should process valid payment successfully', async () => {
      // Arrange
      const paymentRequest = {
        amount: 10000, // $100.00 in cents
        currency: 'USD',
        sender: 'user123',
        recipient: 'user456',
        description: 'Test payment'
      };

      mockFraudService.assessRisk.mockResolvedValue({ score: 25, decision: 'approve' });
      mockLedgerService.reserveFunds.mockResolvedValue(true);
      mockLedgerService.executeTransfer.mockResolvedValue({ transactionId: 'txn_123' });

      // Act
      const result = await paymentService.processPayment(paymentRequest);

      // Assert
      expect(result.success).toBe(true);
      expect(result.transactionId).toBe('txn_123');
      expect(mockFraudService.assessRisk).toHaveBeenCalledWith(paymentRequest);
      expect(mockLedgerService.reserveFunds).toHaveBeenCalled();
      expect(mockLedgerService.executeTransfer).toHaveBeenCalled();
    });

    it('should reject payment with high fraud score', async () => {
      // Arrange
      const paymentRequest = {
        amount: 10000,
        currency: 'USD',
        sender: 'suspicious_user',
        recipient: 'user456'
      };

      mockFraudService.assessRisk.mockResolvedValue({ score: 95, decision: 'decline' });

      // Act
      const result = await paymentService.processPayment(paymentRequest);

      // Assert
      expect(result.success).toBe(false);
      expect(result.reason).toBe('FRAUD_DETECTED');
      expect(mockLedgerService.reserveFunds).not.toHaveBeenCalled();
    });

    it('should handle insufficient funds error', async () => {
      // Arrange
      const paymentRequest = {
        amount: 100000, // $1000.00
        currency: 'USD',
        sender: 'poor_user',
        recipient: 'user456'
      };

      mockFraudService.assessRisk.mockResolvedValue({ score: 10, decision: 'approve' });
      mockLedgerService.reserveFunds.mockResolvedValue(false);

      // Act
      const result = await paymentService.processPayment(paymentRequest);

      // Assert
      expect(result.success).toBe(false);
      expect(result.reason).toBe('INSUFFICIENT_FUNDS');
    });
  });
});
```

#### **React Component Tests**
```typescript
// SendMoneyForm.test.tsx
describe('SendMoneyForm', () => {
  const mockOnSubmit = jest.fn();

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('should render all form fields', () => {
    render(<SendMoneyForm onSubmit={mockOnSubmit} />);
    
    expect(screen.getByLabelText(/recipient email/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/amount/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/description/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /send money/i })).toBeInTheDocument();
  });

  it('should validate required fields', async () => {
    render(<SendMoneyForm onSubmit={mockOnSubmit} />);
    
    const submitButton = screen.getByRole('button', { name: /send money/i });
    fireEvent.click(submitButton);

    await waitFor(() => {
      expect(screen.getByText(/recipient email is required/i)).toBeInTheDocument();
      expect(screen.getByText(/amount is required/i)).toBeInTheDocument();
    });

    expect(mockOnSubmit).not.toHaveBeenCalled();
  });

  it('should submit form with valid data', async () => {
    render(<SendMoneyForm onSubmit={mockOnSubmit} />);
    
    const emailInput = screen.getByLabelText(/recipient email/i);
    const amountInput = screen.getByLabelText(/amount/i);
    const submitButton = screen.getByRole('button', { name: /send money/i });

    fireEvent.change(emailInput, { target: { value: 'recipient@example.com' } });
    fireEvent.change(amountInput, { target: { value: '100' } });
    fireEvent.click(submitButton);

    await waitFor(() => {
      expect(mockOnSubmit).toHaveBeenCalledWith({
        recipient: 'recipient@example.com',
        amount: 10000, // Converted to cents
        currency: 'USD',
        description: ''
      });
    });
  });
});
```

## 🔗 **Integration Testing**

### **API Integration Tests**
```typescript
// payment.integration.test.ts  
describe('Payment API Integration', () => {
  let app: INestApplication;
  let database: TestDatabase;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = module.createNestApplication();
    database = new TestDatabase();
    
    await app.init();
    await database.seed();
  });

  afterAll(async () => {
    await database.cleanup();
    await app.close();
  });

  describe('POST /api/payments/transfer', () => {
    it('should create transfer successfully', async () => {
      const transferData = {
        amount: 5000, // $50.00
        currency: 'USD',
        recipient: 'recipient@example.com',
        description: 'Integration test transfer'
      };

      const response = await request(app.getHttpServer())
        .post('/api/payments/transfer')
        .set('Authorization', `Bearer ${await getTestUserToken()}`)
        .send(transferData)
        .expect(200);

      expect(response.body.success).toBe(true);
      expect(response.body.data.amount).toBe(5000);
      expect(response.body.data.status).toBe('completed');

      // Verify database state
      const transaction = await database.findTransactionById(response.body.data.id);
      expect(transaction.amount).toBe(5000);
      expect(transaction.status).toBe('completed');
    });

    it('should reject transfer with invalid recipient', async () => {
      const transferData = {
        amount: 5000,
        currency: 'USD',
        recipient: 'nonexistent@example.com'
      };

      await request(app.getHttpServer())
        .post('/api/payments/transfer')
        .set('Authorization', `Bearer ${await getTestUserToken()}`)
        .send(transferData)
        .expect(400);
    });
  });
});
```

### **Database Integration Tests**
```typescript
// database.integration.test.ts
describe('Database Integration', () => {
  let prisma: PrismaClient;

  beforeAll(async () => {
    prisma = new PrismaClient({
      datasources: { db: { url: process.env.TEST_DATABASE_URL } }
    });
  });

  beforeEach(async () => {
    // Clean slate for each test
    await prisma.transaction.deleteMany();
    await prisma.account.deleteMany();
    await prisma.user.deleteMany();
  });

  describe('User and Account Creation', () => {
    it('should create user with account atomically', async () => {
      const userData = {
        email: 'test@example.com',
        encrypted_phone: 'encrypted_phone_data',
        kyc_status: 'pending'
      };

      // This should create user and account in a transaction
      const result = await prisma.$transaction(async (tx) => {
        const user = await tx.user.create({ data: userData });
        const account = await tx.account.create({
          data: {
            user_id: user.id,
            currency: 'USD',
            balance_cents: 0,
            available_cents: 0
          }
        });
        return { user, account };
      });

      expect(result.user.email).toBe('test@example.com');
      expect(result.account.balance_cents).toBe(0);
      expect(result.account.user_id).toBe(result.user.id);
    });
  });
});
```

## 🌐 **End-to-End Testing**

### **E2E Testing with Playwright**
```typescript
// e2e/payment-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Payment Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login as test user
    await page.goto('/login');
    await page.fill('[data-testid="email"]', 'testuser@example.com');
    await page.fill('[data-testid="password"]', 'TestPassword123!');
    await page.click('[data-testid="login-button"]');
    await expect(page).toHaveURL('/dashboard');
  });

  test('should complete full payment flow', async ({ page }) => {
    // Navigate to send money page
    await page.click('[data-testid="send-money-button"]');
    await expect(page).toHaveURL('/send');

    // Fill payment form
    await page.fill('[data-testid="recipient-email"]', 'recipient@example.com');
    await page.fill('[data-testid="amount"]', '25.00');
    await page.fill('[data-testid="description"]', 'E2E test payment');

    // Submit payment
    await page.click('[data-testid="send-payment-button"]');

    // Verify success message
    await expect(page.locator('[data-testid="success-message"]')).toContainText('Payment sent successfully');

    // Check transaction appears in history
    await page.goto('/transactions');
    await expect(page.locator('[data-testid="transaction-row"]').first()).toContainText('recipient@example.com');
    await expect(page.locator('[data-testid="transaction-row"]').first()).toContainText('$25.00');
  });

  test('should handle insufficient funds gracefully', async ({ page }) => {
    // Try to send more money than available
    await page.goto('/send');
    await page.fill('[data-testid="recipient-email"]', 'recipient@example.com');
    await page.fill('[data-testid="amount"]', '10000.00'); // $10,000
    
    await page.click('[data-testid="send-payment-button"]');
    
    // Verify error message
    await expect(page.locator('[data-testid="error-message"]')).toContainText('Insufficient funds');
  });

  test('should prevent fraud attempts', async ({ page }) => {
    // Simulate suspicious activity pattern
    for (let i = 0; i < 10; i++) {
      await page.goto('/send');
      await page.fill('[data-testid="recipient-email"]', `victim${i}@example.com`);
      await page.fill('[data-testid="amount"]', '999.99');
      await page.click('[data-testid="send-payment-button"]');
    }

    // Should eventually get blocked
    await expect(page.locator('[data-testid="fraud-warning"]')).toBeVisible();
  });
});
```

### **Mobile E2E Tests**
```typescript
// e2e/mobile-payment.spec.ts
import { test, expect, devices } from '@playwright/test';

// Configure for mobile testing
test.use({ ...devices['iPhone 13'] });

test.describe('Mobile Payment Experience', () => {
  test('should work on mobile devices', async ({ page }) => {
    await page.goto('/');
    
    // Test responsive design
    const sendButton = page.locator('[data-testid="mobile-send-button"]');
    await expect(sendButton).toBeVisible();
    
    // Test touch interactions
    await sendButton.tap();
    await expect(page).toHaveURL('/send');
  });
});
```

## 🔒 **Security Testing**

### **Automated Security Tests**
```typescript
// security/auth.security.test.ts
describe('Authentication Security', () => {
  describe('JWT Security', () => {
    it('should reject expired tokens', async () => {
      const expiredToken = generateExpiredJWT();
      
      const response = await request(app.getHttpServer())
        .get('/api/user/profile')
        .set('Authorization', `Bearer ${expiredToken}`)
        .expect(401);

      expect(response.body.message).toBe('Token expired');
    });

    it('should reject tampered tokens', async () => {
      const validToken = await generateValidJWT();
      const tamperedToken = validToken.slice(0, -10) + 'tampered123';
      
      await request(app.getHttpServer())
        .get('/api/user/profile')  
        .set('Authorization', `Bearer ${tamperedToken}`)
        .expect(401);
    });
  });

  describe('Input Validation', () => {
    it('should prevent SQL injection', async () => {
      const maliciousInput = "'; DROP TABLE users; --";
      
      await request(app.getHttpServer())
        .post('/api/auth/login')
        .send({
          email: maliciousInput,
          password: 'password123'
        })
        .expect(400);

      // Verify users table still exists
      const userCount = await database.user.count();
      expect(userCount).toBeGreaterThanOrEqual(0);
    });

    it('should prevent XSS attacks', async () => {
      const xssPayload = '<script>alert("xss")</script>';
      
      const response = await request(app.getHttpServer())
        .post('/api/user/profile')
        .set('Authorization', `Bearer ${await getTestUserToken()}`)
        .send({ name: xssPayload })
        .expect(400);

      expect(response.body.message).toContain('Invalid input');
    });
  });
});
```

## 📊 **Performance Testing**

### **Load Testing with Artillery**
```yaml
# artillery-config.yml
config:
  target: 'https://api.futurexfinance.com'
  phases:
    - duration: 300  # 5 minutes
      arrivalRate: 10  # 10 requests per second
    - duration: 600  # 10 minutes  
      arrivalRate: 50  # 50 requests per second
    - duration: 300  # 5 minutes
      arrivalRate: 100 # 100 requests per second (peak load)

scenarios:
  - name: "Payment API Load Test"
    weight: 70
    flow:
      - post:
          url: "/api/auth/login"
          json:
            email: "loadtest@example.com"
            password: "LoadTest123!"
          capture:
            - json: "$.token"
              as: "authToken"
      - post:
          url: "/api/payments/transfer"
          headers:
            Authorization: "Bearer {{ authToken }}"
          json:
            amount: 1000
            currency: "USD"
            recipient: "recipient@example.com"
            
  - name: "User Registration Load Test"
    weight: 30
    flow:
      - post:
          url: "/api/auth/register"
          json:
            email: "user{{ $randomString() }}@example.com"
            password: "TestPassword123!"
            phone: "+1555{{ $randomInt(1000000, 9999999) }}"
```

### **Performance Test Assertions**
```typescript
// performance/api-performance.test.ts
describe('API Performance', () => {
  it('should respond to payment requests within SLA', async () => {
    const startTime = Date.now();
    
    const response = await request(app.getHttpServer())
      .post('/api/payments/transfer')
      .set('Authorization', `Bearer ${await getTestUserToken()}`)
      .send({
        amount: 5000,
        currency: 'USD',
        recipient: 'recipient@example.com'
      })
      .expect(200);

    const responseTime = Date.now() - startTime;
    
    // SLA: API responses must be under 500ms
    expect(responseTime).toBeLessThan(500);
  });

  it('should handle concurrent requests efficiently', async () => {
    const concurrentRequests = 50;
    const requests = Array(concurrentRequests).fill(null).map(() =>
      request(app.getHttpServer())
        .get('/api/user/balance')
        .set('Authorization', `Bearer ${await getTestUserToken()}`)
    );

    const startTime = Date.now();
    const responses = await Promise.all(requests);
    const totalTime = Date.now() - startTime;

    // All requests should succeed
    responses.forEach(response => {
      expect(response.status).toBe(200);
    });

    // Average response time should be reasonable under load
    const avgResponseTime = totalTime / concurrentRequests;
    expect(avgResponseTime).toBeLessThan(1000); // 1 second average
  });
});
```

## 🎯 **Test Data Management**

### **Test Data Factory**
```typescript
// test-utils/factories.ts
export class TestDataFactory {
  static createUser(overrides: Partial<User> = {}): User {
    return {
      id: faker.string.uuid(),
      email: faker.internet.email(),
      encrypted_phone: faker.string.alphanumeric(32),
      kyc_status: 'approved',
      created_at: new Date(),
      updated_at: new Date(),
      ...overrides
    };
  }

  static createAccount(userId: string, overrides: Partial<Account> = {}): Account {
    return {
      id: faker.string.uuid(),
      user_id: userId,
      currency: 'USD',
      balance_cents: 100000, // $1,000 default balance
      available_cents: 100000,
      created_at: new Date(),
      updated_at: new Date(),
      ...overrides
    };
  }

  static createTransaction(overrides: Partial<Transaction> = {}): Transaction {
    return {
      id: faker.string.uuid(),
      amount_cents: faker.number.int({ min: 100, max: 100000 }),
      currency: 'USD',
      status: 'completed',
      sender_id: faker.string.uuid(),
      recipient_id: faker.string.uuid(),
      description: faker.lorem.sentence(),
      created_at: new Date(),
      ...overrides
    };
  }
}
```

## 🔧 **Testing Configuration**

### **Jest Configuration**
```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/test'],
  testMatch: [
    '**/__tests__/**/*.+(ts|tsx|js)',
    '**/*.(test|spec).+(ts|tsx|js)'
  ],
  transform: {
    '^.+\\.(ts|tsx)$': 'ts-jest'
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/main.ts',
    '!src/**/*.interface.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90
    }
  },
  setupFilesAfterEnv: ['<rootDir>/test/setup.ts'],
  testTimeout: 10000
};
```

### **Playwright Configuration**
```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    video: 'retain-on-failure',
    screenshot: 'only-on-failure'
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox', 
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] }
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] }
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] }
    }
  ],

  webServer: {
    command: 'npm run dev',
    port: 3000,
    reuseExistingServer: !process.env.CI
  }
});
```

---

## 📋 **Testing Checklist**

### **Pre-Commit Checklist**
- [ ] Unit tests passing (>90% coverage)
- [ ] Integration tests passing
- [ ] Linting and formatting checks
- [ ] Type checking passing
- [ ] Security tests passing

### **Pre-Deployment Checklist**
- [ ] All test suites passing
- [ ] E2E tests passing on staging
- [ ] Performance tests within SLA
- [ ] Security scans clean
- [ ] Load testing completed

### **Quality Metrics**
- **Unit Test Coverage**: >90%
- **Integration Test Coverage**: >80%
- **E2E Test Coverage**: Critical user journeys (100%)
- **Performance**: <500ms API response time
- **Security**: Zero critical vulnerabilities

---

**🧪 This comprehensive testing strategy ensures FutureXFinance maintains enterprise-grade quality with 99.9% reliability for critical financial operations.**
