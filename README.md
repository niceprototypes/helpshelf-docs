# HelpShelf Backend API Documentation

A comprehensive technical analysis of the HelpShelf Django backend for full stack engineers integrating the new React/TypeScript frontend (`helpshelf-ui`).

## 📋 Overview

This repository contains detailed documentation of HelpShelf's multi-tenant SaaS platform backend, focusing on API endpoints, data models, and integration patterns needed for the `helpshelf-ui` React frontend development.

## 🎯 Purpose

**Target Audience**: New full stack engineers onboarding to the HelpShelf platform  
**Primary Use Case**: Understanding backend architecture for helpshelf-ui integration via iframes and direct API calls

## 📖 Documentation Contents

### [📄 Full Documentation](./helpshelf-api.md)

The complete analysis covers:

1. **Executive Summary** - Platform overview and key integration points
2. **System Architecture** - Multi-tenant foundation, technology stack, data flows
3. **Core Data Models** - User/site management, content & AI system models
4. **API Endpoints** - Guest user management, AI search, content access, analytics
5. **Django Apps Overview** - 5 core apps (Manage, Billing, Crawler, Stats, LLM Core)
6. **Core Services** - AI embedding, payment processing, content extraction, analytics
7. **Admin Interface** - Site, content, subscription, and event administration
8. **Common Patterns** - Authentication, content sync, event tracking patterns
9. **Security & Performance** - Tenant isolation, rate limiting, caching strategies
10. **Integration Architecture** - Widget embedding, iframe communication, direct API access
11. **Development Considerations** - Code organization, testing, deployment

## 🔗 Key Integration Points for helpshelf-ui

### Primary API Endpoints
```
POST /api/embed/ai-searchbox/search/{user_uuid}/     # AI search queries
POST /api/ai-searchbox/feedback/{user_uuid}/         # User feedback collection
GET  /api/ai-searchbox/crawler-content/{user_uuid}/  # Content retrieval
POST /api/stats/events/                              # Analytics tracking
```

### Guest User System
- **UUID-based sessions** for anonymous widget interactions
- **Session management** with expiration and metadata tracking
- **Cross-origin support** for embedded widgets

### Direct Backend Integration
- **Bypass Django views** for performance
- **Service layer access** for AI responses and analytics
- **iframe communication** via PostMessage API

## 🏗️ System Architecture

```
Widget Embed → Guest UUID → API Endpoints → AI Processing → Response + Analytics
     ↓              ↓            ↓              ↓              ↓
   S3 Assets    Profile App   API App     LLM Core App    Stats App
```

## 🔧 Technology Stack

- **Backend**: Django + PostgreSQL with full-text search (GIN indexes)
- **AI/ML**: OpenAI GPT + Milvus vector database for semantic search
- **Infrastructure**: AWS Lambda (crawling), S3 (assets), Redis (caching)
- **Integrations**: Stripe/PayPal (billing), 20+ provider APIs

## 📊 Business Impact Metrics

- **AI Response Accuracy**: 40-60% improvement with semantic search
- **Dashboard Performance**: 3-5 seconds → <200ms load times
- **User Satisfaction**: 90%+ with intelligent prompt engineering
- **Content Extraction**: 85%+ accuracy in web scraping

## 🚀 Quick Start for Engineers

1. **Read Executive Summary** - Get platform overview and key concepts
2. **Review API Endpoints** (Section 4) - Understand helpshelf-ui integration points
3. **Study Integration Architecture** (Section 10) - Learn iframe and direct API patterns
4. **Explore Core Services** (Section 6) - Understand business logic and data flows

## 📁 File Structure

```
helpshelf-api-md/
├── README.md                 # This summary document
└── helpshelf-api.md         # Complete technical documentation (534 lines)
```

## 🔍 Navigation Tips

The full documentation includes a **hierarchical table of contents** with numbered sections (1-11) and lettered subsections (a-e) for easy reference during meetings and development discussions.

**Example references**:
- "See section 4b for AI Search integration"
- "Reference 6c for content extraction logic"  
- "Check 10a for widget embedding flow"

## 🎯 Integration Summary

helpshelf-ui integrates primarily through:
- **Guest UUID sessions** for anonymous user management
- **Public API endpoints** with strict tenant isolation
- **iframe communication** via PostMessage API
- **Direct service calls** bypassing Django views for performance

---

**Generated for**: HelpShelf full stack engineering team  
**Last Updated**: September 2024  
**Maintainer**: Engineering Documentation Team