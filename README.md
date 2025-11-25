# CarbonCheck Wiki

> **Making sustainable shopping choices accessible, one product at a time**

## Table of Contents

1. [Product Overview](#product-overview)
2. [System Architecture](#system-architecture)
3. [Data Methodology](#data-methodology)
4. [Component Overview](#component-overview)
5. [Data Flow](#data-flow)
6. [Quick Start](#quick-start)
7. [Technology Stack](#technology-stack)

---

## Product Overview

**CarbonCheck** is a browser extension that helps eco-conscious consumers understand and reduce the carbon footprint of their online purchases. When shopping on Amazon, CarbonCheck provides real-time CO2 equivalent (CO2e) emissions estimates and recommends sustainable alternatives.

### Key Features

```
┌─────────────────────────────────────────────────────────────┐
│                    CarbonCheck Features                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  🌍 Carbon Footprint Calculation                             │
│     • Instant CO2e estimates for Amazon products             │
│     • Emissions displayed in kg/lbs with driving equivalents │
│     • Context: "X lbs CO2e = Y miles driven"                 │
│                                                               │
│  ⭐ Environmental Ratings                                     │
│     • Excellent / Good / Fair / Poor ratings                 │
│     • Based on product category CO2e percentiles             │
│     • Visual circular progress indicators                    │
│                                                               │
│  🔍 Smart Recommendations                                     │
│     • AI-powered similar product suggestions                 │
│     • Lower carbon footprint alternatives                    │
│     • Semantic similarity matching via OpenAI                │
│                                                               │
│  ✅ Sustainability Attributes                                 │
│     • Certification detection (organic, vegan, etc.)         │
│     • Climate Pledge Friendly badge recognition             │
│     • Category-specific attribute evaluation                 │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Target Audience

- **Eco-conscious consumers** who want to reduce their environmental impact
- **Amazon shoppers** looking for sustainable product alternatives
- **Climate-aware individuals** seeking data-driven purchasing decisions

### Current Status

**Early Beta / Prototype** - Active development phase

---

## System Architecture

CarbonCheck is built as a distributed system with four main components:

```
┌──────────────────────────────────────────────────────────────────────┐
│                        CARBONCHECK ARCHITECTURE                       │
└──────────────────────────────────────────────────────────────────────┘

        ┌─────────────────────────────────────────────────┐
        │         USER BROWSING AMAZON                    │
        │         www.amazon.com/product                  │
        └───────────────────┬─────────────────────────────┘
                            │
                            │ [1] Page Load
                            ↓
        ┌─────────────────────────────────────────────────┐
        │      BROWSER EXTENSION (Chrome)                 │
        │  ┌──────────────────────────────────────────┐   │
        │  │  • Extracts ASIN, title, price           │   │
        │  │  • Renders inline widget on page         │   │
        │  │  • Shadow DOM for style isolation        │   │
        │  └──────────────────────────────────────────┘   │
        └───────────────────┬─────────────────────────────┘
                            │
                            │ [2] POST /products/co2e_estimate
                            │     { asin, title, price, category }
                            ↓
        ┌─────────────────────────────────────────────────┐
        │    PRODUCT-RATINGS API (Go / ECS Fargate)       │
        │  ┌──────────────────────────────────────────┐   │
        │  │  • Calculates CO2e from category data    │   │
        │  │  • Assigns environmental rating          │   │
        │  │  • Generates recommendations             │   │
        │  │  • Evaluates product attributes          │   │
        │  └──────────────────────────────────────────┘   │
        └────────┬──────────────────────┬─────────────────┘
                 │                      │
   [3] Query    │                      │ [4] OpenAI API
   Categories & │                      │     • Embeddings (Ada v2)
   CO2e Data    │                      │     • GPT-3.5 Turbo
                 ↓                      ↓
        ┌──────────────────┐   ┌──────────────────────┐
        │   POSTGRESQL     │   │    OPENAI API        │
        │   (RDS)          │   │  ┌────────────────┐  │
        │  ┌────────────┐  │   │  │ Similarity     │  │
        │  │ products   │  │   │  │ Ranking        │  │
        │  │ categories │  │   │  └────────────────┘  │
        │  │ co2e_data  │  │   │  ┌────────────────┐  │
        │  └────────────┘  │   │  │ Certification  │  │
        │                  │   │  │ Detection      │  │
        └──────────────────┘   │  └────────────────┘  │
                               └──────────────────────┘

        ┌─────────────────────────────────────────────────┐
        │       EEIPO_API_V1 (Data Pipeline)              │
        │  ┌──────────────────────────────────────────┐   │
        │  │  • Preprocesses CO2e data                │   │
        │  │  • Maps products to USEEIO sectors       │   │
        │  │  • Generates CSV for database import     │   │
        │  └──────────────────────────────────────────┘   │
        └─────────────────────────────────────────────────┘
                            │
                            │ [5] CSV Export
                            ↓
                   [Imported into PostgreSQL]
```

### Component Responsibilities

| Component | Role | Technology |
|-----------|------|------------|
| **browser-extension** | User interface on Amazon.com | React, TypeScript, Chrome Extension API |
| **product-ratings** | Core API for ratings & recommendations | Go, Chi router, PostgreSQL |
| **eeipo_api_v1** | Data preprocessing & CO2e calculation pipeline | Python, FastAPI, USEEIO model |
| **infra** | Cloud infrastructure provisioning | AWS CDK, ECS Fargate, RDS |

---

## Data Methodology

### CO2 Equivalent (CO2e) Calculation

CarbonCheck uses **Environmental Economic Input-Output (EEIO) analysis** to estimate product carbon footprints:

```
┌────────────────────────────────────────────────────────────────┐
│              CO2e CALCULATION METHODOLOGY                       │
└────────────────────────────────────────────────────────────────┘

STEP 1: Product Classification
├─ Extract product category from Amazon breadcrumb
├─ Map to NAICS (North American Industry Classification)
└─ Fallback to BEA (Bureau of Economic Analysis) sector

STEP 2: USEEIO Model Lookup
├─ Use USEEIO v2.0.1 (411 economic sectors)
├─ Retrieve GHG coefficient per sector
└─ Units: kg CO2e per dollar of economic output

STEP 3: Price Normalization
├─ Adjust product price to 2021 dollars
│  price_2021 = product_price / 1.12
└─ Account for inflation baseline

STEP 4: Emission Calculation
├─ CO2e = price_2021 × sector_coefficient
├─ Apply 35% reduction for Climate Pledge Friendly
└─ Store in database with category clustering

STEP 5: Rating Assignment
├─ Calculate category-level CO2e percentiles (P25, P50, P75)
├─ Rate based on product position in distribution:
│  • Excellent: < P25 (lowest 25%)
│  • Good:     P25 - P50
│  • Fair:     P50 - P75
│  • Poor:     > P75 (highest 25%)
└─ Downgrade if not Climate Pledge Friendly
```

### Data Sources

1. **USEEIO v2.0.1** - US Environmental Economic Input-Output Model
   - 411 economic sectors with GHG coefficients
   - Published by US EPA
   - Captures supply chain emissions

2. **Amazon Product Data**
   - ASIN, title, price, category
   - Climate Pledge Friendly badge
   - Product descriptions and certifications

3. **OpenAI Embeddings**
   - Ada v2 model for semantic similarity
   - GPT-3.5 Turbo for attribute extraction

### Rating Formula

```
Rating Logic (Pseudocode):
─────────────────────────────────────────
IF co2e < category_p25:
    rating = "Excellent"
ELSE IF co2e < category_p50:
    rating = "Good"
ELSE IF co2e < category_p75:
    rating = "Fair"
ELSE:
    rating = "Poor"

IF NOT climate_pledge_friendly:
    downgrade_rating()  // Good → Fair, etc.
```

---

## Component Overview

### 1. Browser Extension

**Location:** `/browser-extension`

Chrome extension that integrates directly into Amazon product pages.

**Key Technologies:**
- React 18.2.0 + TypeScript
- Tailwind CSS for styling
- Shadow DOM for isolation
- Webpack 5 build system

**Features:**
- Inline widget embedded on Amazon pages
- Circular progress indicator for CO2e rating
- Full-screen recommendation modal
- Social sharing (Facebook, Twitter, LinkedIn)
- Plausible analytics tracking

**Entry Point:** `content_script.tsx` → injected on `*://www.amazon.com/*`

---

### 2. Product Ratings API

**Location:** `/product-ratings`

Go microservice providing CO2e estimates, ratings, and recommendations.

**Key Technologies:**
- Go 1.20
- Chi router
- PostgreSQL with pq driver
- OpenAI API (go-openai)
- Sentry error tracking

**API Endpoints:**
```
GET  /status                              → Health check
GET  /products/{id}                       → Fetch product by ID
GET  /products/asin/{asin}                → Fetch by ASIN
POST /products/co2e_estimate              → Calculate emissions
POST /products/evaluate_attributes        → Identify certifications
POST /products/recommendations            → Get alternatives
```

**Database Schema (Key Tables):**
- `products` - Product catalog with CO2e data
- `categories` - Category taxonomy
- `category_clusters_co2e_metrics` - Statistical CO2e metrics per cluster
- `product_attributes` - Certifications and attributes
- `mv_top_products_per_cluster` - Materialized view for recommendations

---

### 3. EEIPO API v1 (Data Pipeline)

**Location:** `/eeipo_api_v1`

Python-based data processing pipeline for CO2e calculation and database seeding.

**Key Technologies:**
- FastAPI 0.97.0
- Pandas for data processing
- Sentence Transformers for NLP
- USEEIO v2.0.1 environmental data

**Process Flow:**
1. Load USEEIO sector data (411 sectors)
2. Generate embeddings for NAICS codes
3. Map products to sectors using semantic similarity
4. Calculate CO2e using sector coefficients
5. Export CSV for PostgreSQL import

**Output:** CSV files ingested by product-ratings database

---

### 4. Infrastructure (AWS)

**Location:** `/infra`

Infrastructure as Code using AWS CDK.

**Deployed Resources:**
```
┌──────────────────────────────────────┐
│      AWS INFRASTRUCTURE              │
├──────────────────────────────────────┤
│                                      │
│  VPC (2 Availability Zones)          │
│    ├─ Public Subnets                 │
│    └─ Private Subnets                │
│                                      │
│  ECS Fargate Cluster                 │
│    ├─ Task Definition                │
│    ├─ Service (2 tasks)              │
│    └─ Application Load Balancer      │
│                                      │
│  RDS PostgreSQL 14                   │
│    ├─ Instance: db.t4g.medium        │
│    └─ Database: products_store       │
│                                      │
│  ECR (Container Registry)            │
│    └─ product-ratings-api:latest     │
│                                      │
│  IAM Roles & Policies                │
│  Secrets Manager                     │
│                                      │
└──────────────────────────────────────┘
```

**Region:** us-west-2

---

## Data Flow

### User Journey: Viewing Product Emissions

```
┌───────────────────────────────────────────────────────────────────┐
│ SEQUENCE: User Views Amazon Product                               │
└───────────────────────────────────────────────────────────────────┘

[1] User navigates to Amazon product page
    │
    ├─→ Extension content script injects
    │
    └─→ Extracts: ASIN, title, price, category, Climate Pledge badge

[2] Extension calls product-ratings API
    │
    POST /products/co2e_estimate
    {
      "asin": "B08XYZ123",
      "title": "Organic Cotton T-Shirt",
      "price": 24.99,
      "categoryNodeId": "1234567",
      "climatePledgeFriendly": true
    }

[3] Product-ratings API processes request
    │
    ├─→ Queries PostgreSQL for category CO2e metrics
    ├─→ Calculates: co2e = (price / 1.12) × category_coefficient
    ├─→ Applies 35% reduction for Climate Pledge
    └─→ Assigns rating based on percentiles

[4] API returns response
    │
    {
      "co2e": 3.2,
      "co2e_unit": "kg CO2e",
      "rating": "Good",
      "equivalentMiles": 7.9
    }

[5] Extension renders widget on page
    │
    └─→ Displays: circular rating indicator, CO2e value, context
```

### User Journey: Getting Recommendations

```
┌───────────────────────────────────────────────────────────────────┐
│ SEQUENCE: User Requests Recommendations                           │
└───────────────────────────────────────────────────────────────────┘

[1] User clicks "Show Recommendations" button
    │
    └─→ Extension opens full-screen modal

[2] Extension calls recommendations endpoint
    │
    POST /products/recommendations
    {
      "asin": "B08XYZ123",
      "title": "Organic Cotton T-Shirt",
      "categoryNodeId": "1234567"
    }

[3] Product-ratings API generates recommendations
    │
    ├─→ Queries database for similar products in category
    ├─→ Filters by better CO2e rating (Excellent > Good > Fair)
    ├─→ Calls OpenAI Ada v2 for title embeddings
    ├─→ Calculates cosine similarity scores
    ├─→ Ranks by: similarity (90%) + Climate Pledge (10%)
    └─→ Returns top 10 diverse alternatives

[4] API response with alternatives
    │
    [
      {
        "asin": "B09ABC789",
        "title": "100% Organic Fair Trade Tee",
        "co2e": 2.1,
        "rating": "Excellent",
        "price": 22.99,
        "manufacturer": "EcoWear"
      },
      ...
    ]

[5] Extension displays product grid
    │
    └─→ 2-column layout with product cards, images, CO2e comparison
```

---

## Quick Start

### Prerequisites

- Node.js 18.12.0+
- Go 1.20+
- Python 3.10+
- Docker & Docker Compose
- AWS CLI (for infrastructure deployment)
- PostgreSQL 14+

### Local Development Setup

#### 1. Browser Extension

```bash
cd browser-extension
npm install
npm run watch          # Development mode with hot reload
npm run build          # Production build

# Load extension in Chrome:
# 1. Navigate to chrome://extensions
# 2. Enable "Developer mode"
# 3. Click "Load unpacked"
# 4. Select browser-extension/dist directory
```

#### 2. Product Ratings API

```bash
cd product-ratings

# Set environment variables
export DB_CONNECTION_STRING="postgres://user:pass@localhost:5432/products_store"
export OPENAI_API_KEY="sk-..."
export APP_ENV="development"

# Run the service
go run cmd/main.go

# Service runs on http://localhost:8080
```

#### 3. EEIPO Data Pipeline

```bash
cd eeipo_api_v1

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Run the API (if needed)
uvicorn main:app --reload

# Generate CO2e data CSV
python main.py --generate-csv
```

#### 4. Deploy Infrastructure

```bash
cd infra
npm install

# Bootstrap CDK (first time only)
cdk bootstrap aws://ACCOUNT-ID/us-west-2

# Deploy to AWS
cdk deploy
```

---

## Technology Stack

### Frontend
- **React 18.2.0** - UI framework
- **TypeScript** - Type safety
- **Tailwind CSS** - Styling
- **Webpack 5** - Bundling

### Backend
- **Go 1.20** - High-performance API service
- **FastAPI** - Python data pipeline
- **Chi Router** - Lightweight HTTP routing
- **PostgreSQL 14** - Relational database

### AI/ML
- **OpenAI Ada v2** - Semantic embeddings
- **OpenAI GPT-3.5 Turbo** - Attribute extraction
- **Sentence Transformers** - NLP embeddings
- **USEEIO v2.0.1** - Environmental modeling

### Infrastructure
- **AWS ECS Fargate** - Serverless containers
- **AWS RDS** - Managed PostgreSQL
- **AWS CDK** - Infrastructure as Code
- **Docker** - Containerization

### Observability
- **Sentry** - Error tracking
- **Plausible Analytics** - Privacy-focused analytics

---

## Data Privacy & Sustainability

### Privacy Commitments

- **No personal data collection** - CarbonCheck does not store user browsing history
- **Privacy-focused analytics** - Uses Plausible (GDPR compliant)
- **Local processing** - Product analysis happens client-side
- **Opt-out support** - Analytics can be disabled via localStorage

### Environmental Impact Data Sources

All CO2e calculations are based on:
- **USEEIO v2.0.1** - Peer-reviewed EPA environmental model
- **Amazon Climate Pledge Friendly** - Verified sustainability certifications
- **Category-level aggregation** - Statistical CO2e distributions

### Limitations

- Estimates are based on economic sector averages, not product-specific LCA
- Does not include end-of-life disposal or usage phase emissions
- Focused on US-based products and supply chains
- Beta accuracy varies by product category

---

## Repository Structure

```
carboncheck/
├── browser-extension/     # Chrome extension (React/TypeScript)
├── product-ratings/       # Go API service
├── eeipo_api_v1/         # Python data pipeline
├── infra/                # AWS CDK infrastructure
└── CARBONCHECK_WIKI.md  # This document
```

---

## Future Roadmap

- Multi-retailer support (Walmart, Target, etc.)
- Mobile app for in-store scanning
- Carbon offset marketplace integration
- Personalized carbon tracking dashboard
- B2B procurement sustainability analytics

---

**Built with care for the planet** 🌍
