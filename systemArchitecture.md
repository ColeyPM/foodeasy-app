# FoodEasy System Architecture

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [System Architecture Diagram](#system-architecture-diagram)
3. [Technology Stack](#technology-stack)
4. [System Components](#system-components)
5. [Data Models](#data-models)
6. [API Architecture](#api-architecture)
7. [Security & Authentication](#security--authentication)
8. [Deployment Strategy](#deployment-strategy)
9. [Scalability Considerations](#scalability-considerations)

---

## Architecture Overview

FoodEasy follows a **modern 3-tier architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────┐
│                     PRESENTATION LAYER                   │
│        (React.js + Tailwind CSS - Mobile First)         │
└────────────────────┬────────────────────────────────────┘
                     │ HTTPS/REST API
┌────────────────────▼────────────────────────────────────┐
│                    APPLICATION LAYER                     │
│         (Node.js + Express.js + JWT Auth)               │
└────────────────────┬────────────────────────────────────┘
                     │ Mongoose ODM
┌────────────────────▼────────────────────────────────────┐
│                      DATA LAYER                          │
│              (MongoDB + Redis Cache)                     │
└──────────────────────────────────────────────────────────┘
```
## System Architecture Diagram
![System Architecture Diagram](https://image2url.com/images/1762635790848-6b19d0c9-33a4-4876-b6d1-45e2325fa199.png)

### Design Principles
- **Mobile-First**: Optimized for 80% of our target users on smartphones
- **Stateless APIs**: RESTful architecture for horizontal scalability
- **Data-Driven**: Real-time analytics for inventory and delivery optimization
- **Secure by Design**: End-to-end encryption for payments and user data

---

## Technology Stack

### Frontend
| Technology | Purpose | Justification |
|------------|---------|---------------|
| **React.js 18+** | UI Framework | Component reusability, virtual DOM for performance |
| **Tailwind CSS** | Styling | Mobile-first utilities, fast development |
| **React Router** | Navigation | SPA routing with code-splitting |
| **Axios** | HTTP Client | Interceptors for auth tokens, error handling |
| **Redux Toolkit** | State Management | Centralized state for cart, user session |
| **React Query** | Server State | Caching, background refetching for listings |

### Backend
| Technology | Purpose | Justification |
|------------|---------|---------------|
| **Node.js 20+** | Runtime | Non-blocking I/O for handling concurrent orders |
| **Express.js** | Web Framework | Lightweight, extensive middleware ecosystem |
| **MongoDB 7+** | Database | Flexible schema for products, orders, users |
| **Mongoose** | ODM | Schema validation, relationship management |
| **Redis** | Caching | Session storage, cart data, rate limiting |
| **JWT** | Authentication | Stateless auth tokens for scalability |

### Third-Party Services
| Service | Purpose | Integration |
|---------|---------|-------------|
| **Paystack** | Payment Processing | Webhook for payment verification |
| **Twilio** | SMS Notifications | Order updates, OTP verification |
| **Cloudinary** | Image Hosting | Product images, package visuals |
| **Google Maps API** | Delivery Tracking | Real-time location updates |
| **SendGrid** | Email Service | Order confirmations, receipts |

### DevOps
- **Version Control**: Git + GitHub
- **CI/CD**: GitHub Actions
- **Hosting**: Railway (Backend), Vercel (Frontend)
- **Monitoring**: Sentry (Error tracking), Google Analytics
- **Testing**: Jest (Unit), Cypress (E2E)

---

## System Components

### 1. Frontend Application (`/client`)

#### Key Modules

**Authentication Module**
```
/components/Auth
├── LoginForm.jsx
├── SignupForm.jsx
├── PhoneVerification.jsx
└── ProtectedRoute.jsx
```
- Phone/Email login with OTP verification
- JWT token storage in httpOnly cookies
- Protected routes for dashboard access

**Package Selection Module**
```
/components/Packages
├── PackageCard.jsx
├── PackageDetail.jsx
├── PackageComparison.jsx
└── CustomPackageBuilder.jsx
```
- Display 3 pre-curated packages (Basic, Boss, Family)
- Visual breakdown of items, quantities, pricing
- Marketplace for individual item selection

**Cart & Checkout Module**
```
/components/Cart
├── CartSummary.jsx
├── CheckoutForm.jsx
├── DeliveryScheduler.jsx
└── PaymentGateway.jsx
```
- Add/remove items, update quantities
- Delivery address and scheduling
- Paystack payment integration

**User Dashboard Module**
```
/components/Dashboard
├── OrderHistory.jsx
├── DeliveryTracker.jsx
├── ReferralStats.jsx
└── ProfileSettings.jsx
```
- View active and past orders
- Real-time delivery tracking
- Referral points and rewards

#### State Management Strategy
```javascript
// Redux Store Structure
{
  auth: { user, token, isAuthenticated },
  packages: { list, selectedPackage, loading },
  cart: { items, totalPrice, deliveryFee },
  orders: { current, history, tracking }
}
```

---

### 2. Backend Application (`/server`)

#### Directory Structure
```
/server
├── /config         # Database, environment configs
├── /controllers    # Route handlers
├── /models         # Mongoose schemas
├── /routes         # API endpoints
├── /middleware     # Auth, validation, error handling
├── /services       # Business logic (payments, notifications)
├── /utils          # Helpers (JWT, encryption)
└── server.js       # Entry point
```

#### Core Services

**Auth Service**
- JWT generation and verification
- Phone/Email OTP via Twilio
- Password hashing with bcrypt
- Session management with Redis

**Order Processing Service**
```javascript
// Order Workflow
1. Validate cart items and pricing
2. Check inventory availability
3. Calculate delivery fee based on location
4. Create pending order
5. Initiate Paystack payment
6. Confirm payment via webhook
7. Update order status → "Confirmed"
8. Assign to delivery partner
9. Send confirmation SMS/Email
```

**Inventory Management Service**
- Real-time stock tracking
- Auto-reorder triggers for suppliers
- Price update notifications
- Package availability checks

**Notification Service**
- **SMS**: Order confirmations, delivery updates
- **Email**: Receipts, promotional offers
- **Push**: In-app notifications (future)
- Queue system using Bull for async processing

---

## Data Models

### User Schema
```javascript
{
  _id: ObjectId,
  fullName: String,
  phone: { type: String, unique: true, required: true },
  email: { type: String, unique: true },
  password: String (hashed),
  deliveryAddresses: [{
    label: String,
    address: String,
    coordinates: { lat: Number, lng: Number },
    isDefault: Boolean
  }],
  referralCode: String (unique),
  referredBy: ObjectId (ref: User),
  referralPoints: Number,
  role: { type: String, enum: ['customer', 'admin'], default: 'customer' },
  createdAt: Date,
  updatedAt: Date
}
```

### Product Schema
```javascript
{
  _id: ObjectId,
  name: String,
  category: { type: String, enum: ['Staples', 'Proteins', 'Vegetables', 'Dairy', 'Spices'] },
  description: String,
  images: [String], // Cloudinary URLs
  unit: String, // kg, liters, pieces
  pricePerUnit: Number,
  marketPrice: Number, // For comparison
  stock: Number,
  supplier: { type: ObjectId, ref: 'Supplier' },
  isActive: Boolean,
  createdAt: Date
}
```

### Package Schema
```javascript
{
  _id: ObjectId,
  name: { type: String, enum: ['Basic', 'Boss', 'Family'] },
  description: String,
  price: Number,
  items: [{
    product: { type: ObjectId, ref: 'Product' },
    quantity: Number
  }],
  imageUrl: String,
  isActive: Boolean,
  estimatedSavings: Number, // vs market price
  createdAt: Date
}
```

### Order Schema
```javascript
{
  _id: ObjectId,
  orderNumber: String (unique, auto-generated),
  user: { type: ObjectId, ref: 'User' },
  items: [{
    product: { type: ObjectId, ref: 'Product' },
    quantity: Number,
    priceAtPurchase: Number
  }],
  packageId: { type: ObjectId, ref: 'Package' }, // if package order
  totalAmount: Number,
  deliveryFee: Number,
  deliveryAddress: {
    address: String,
    coordinates: { lat: Number, lng: Number }
  },
  deliverySchedule: Date,
  paymentStatus: { type: String, enum: ['pending', 'paid', 'failed'], default: 'pending' },
  paystackReference: String,
  orderStatus: { 
    type: String, 
    enum: ['pending', 'confirmed', 'processing', 'out_for_delivery', 'delivered', 'cancelled'],
    default: 'pending'
  },
  deliveryPartner: { type: ObjectId, ref: 'DeliveryPartner' },
  trackingUpdates: [{
    status: String,
    timestamp: Date,
    location: { lat: Number, lng: Number }
  }],
  createdAt: Date,
  completedAt: Date
}
```

---

## API Architecture

### Base URL
- **Production**: `https://api.foodeasy.app/v1`
- **Development**: `http://localhost:5000/api/v1`

### Key Endpoints

#### Authentication
```
POST   /auth/signup              # Create new user
POST   /auth/login               # Login with phone/email
POST   /auth/verify-otp          # Verify OTP
POST   /auth/logout              # Invalidate token
GET    /auth/me                  # Get current user
```

#### Packages
```
GET    /packages                 # List all active packages
GET    /packages/:id             # Get package details
POST   /packages/compare         # Compare packages
```

#### Products (Marketplace)
```
GET    /products                 # List products (with filters)
GET    /products/:id             # Product details
GET    /products/search?q=rice   # Search products
```

#### Cart
```
GET    /cart                     # Get user's cart
POST   /cart/add                 # Add item to cart
PUT    /cart/update/:itemId      # Update quantity
DELETE /cart/remove/:itemId      # Remove item
```

#### Orders
```
POST   /orders                   # Create new order
GET    /orders                   # User's order history
GET    /orders/:id               # Order details
GET    /orders/:id/track         # Real-time tracking
POST   /orders/:id/cancel        # Cancel order
```

#### Payments
```
POST   /payments/initialize      # Start Paystack payment
POST   /payments/webhook         # Paystack webhook handler
GET    /payments/verify/:ref     # Verify payment status
```

#### Admin (Protected)
```
GET    /admin/orders             # All orders dashboard
PUT    /admin/orders/:id/status  # Update order status
POST   /admin/products           # Add new product
PUT    /admin/packages/:id       # Update package
```

### API Response Format
```javascript
// Success
{
  success: true,
  data: { ... },
  message: "Operation successful"
}

// Error
{
  success: false,
  error: "Error message",
  statusCode: 400
}
```

---

## Security & Authentication

### Authentication Flow
```
1. User submits phone/email
2. Server generates 6-digit OTP → stores in Redis (5min TTL)
3. OTP sent via Twilio SMS
4. User submits OTP
5. Server validates → generates JWT token
6. Token stored in httpOnly cookie (7 days)
7. All subsequent requests include token
```

### JWT Token Structure
```javascript
{
  payload: {
    userId: "user_id",
    role: "customer",
    exp: timestamp
  },
  signed with: process.env.JWT_SECRET
}
```

### Security Measures
- **Password Hashing**: bcrypt with 10 salt rounds
- **HTTPS Only**: All API calls over SSL/TLS
- **Rate Limiting**: 100 requests/15min per IP (express-rate-limit)
- **Input Validation**: Joi schemas for all request bodies
- **SQL Injection Protection**: Mongoose parameterized queries
- **CORS**: Whitelist frontend domain only
- **Helmet.js**: Security headers (CSP, XSS protection)
- **Payment Security**: Paystack handles card details (PCI compliant)

---

## Deployment Strategy

### Frontend (Vercel)
```yaml
Build Command: npm run build
Output Directory: build
Environment Variables:
  - REACT_APP_API_URL
  - REACT_APP_PAYSTACK_PUBLIC_KEY
  - REACT_APP_GOOGLE_MAPS_KEY
```

### Backend (Railway)
```yaml
Start Command: npm start
Health Check: GET /health
Environment Variables:
  - MONGODB_URI
  - REDIS_URL
  - JWT_SECRET
  - PAYSTACK_SECRET_KEY
  - TWILIO_ACCOUNT_SID
  - CLOUDINARY_URL
```

### Database (MongoDB Atlas)
- **Tier**: M10 cluster (production)
- **Region**: eu-west-1 (closest to Nigeria)
- **Backup**: Daily automated snapshots
- **Indexing**: Compound indexes on frequently queried fields

### CI/CD Pipeline (GitHub Actions)
```yaml
Trigger: Push to main branch
Steps:
  1. Run linting (ESLint)
  2. Run unit tests (Jest)
  3. Build frontend
  4. Build backend
  5. Deploy to Railway (backend)
  6. Deploy to Vercel (frontend)
  7. Run E2E tests (Cypress)
  8. Notify team on Slack
```

---

## Scalability Considerations

### Current MVP Architecture (0-10K users)
- Single Node.js server instance
- MongoDB shared cluster
- Redis for sessions
- **Estimated Cost**: $50-100/month

### Phase 2 (10K-100K users)
- **Load Balancing**: Nginx reverse proxy → 3 Node.js instances
- **Database**: MongoDB replica set (3 nodes)
- **Caching**: Redis cluster for cart/session data
- **CDN**: Cloudflare for static assets
- **Estimated Cost**: $300-500/month

### Phase 3 (100K+ users)
- **Microservices**: Separate services for orders, payments, notifications
- **Message Queue**: RabbitMQ for async processing
- **Auto-Scaling**: Kubernetes cluster
- **Database Sharding**: Partition users by region
- **Estimated Cost**: $1000+/month

### Performance Targets
| Metric | Target |
|--------|--------|
| API Response Time | <200ms (p95) |
| Page Load Time | <2 seconds |
| Checkout Completion | <30 seconds |
| Database Queries | <50ms (indexed) |
| Uptime | 99.9% |

---

## Conclusion

FoodEasy's architecture is designed for **rapid MVP launch** with a clear path to scale. The technology choices prioritize:

1. **Developer Productivity**: Modern, well-documented tools
2. **User Experience**: Fast, mobile-optimized interface
3. **Cost Efficiency**: Leverage free tiers and PaaS solutions
4. **Future-Proof**: Easy to extend with new features

---
# Document Version: **1.0**
Last Updated: November 8 2025
Author:Coletta Ezeagba