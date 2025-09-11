# HelpShelf Platform Documentation

Complete technical documentation for HelpShelf's full-stack platform, covering both Django backend services and React/TypeScript frontend architecture for engineering team integration.

## 🏢 Business Context

[HelpShelf.com](https://helpshelf.com) is a multi-tenant SaaS platform that provides AI-powered customer support solutions. The HelpShelf widget, which a customer installs on their website as a JavaScript snippet, connects to API sources and indexes the content for AI-based search. The platform currently features an existing onboarding process integrated within the Django application. However, this legacy onboarding experience has several limitations:

- **Not mobile-friendly**: The current flow doesn't adapt well to mobile devices
- **Known bugs**: Various UX issues and technical debt have accumulated
- **Design opportunities**: The interface needs modernization to match current standards

## 🎯 Onboarding Integration Project

The goal of this integration project is to launch a new React-based onboarding UI (`helpshelf-ui`) that:

- **Bypasses the Django layer entirely** for improved performance and flexibility
- **Connects directly to backend services** via existing API endpoints (with minor tweaks as needed)
- **Deploys as either**:
  - Full-screen takeover using an iframe within the existing application
  - New browser tab that opens independently from the main platform
- **Provides a modern, mobile-responsive experience** that addresses all current limitations

## 📋 Overview

This repository contains comprehensive documentation for HelpShelf's multi-tenant SaaS platform, including backend APIs, frontend architecture, and integration patterns between the Django backend and React/TypeScript frontend (`helpshelf-ui`).

## 🎯 Purpose

**Target Audience**: New full stack engineers onboarding to the HelpShelf platform  
**Primary Use Case**: Understanding backend architecture for helpshelf-ui integration via iframes and direct API calls

## 📖 Documentation Contents

### [📄 Backend API Documentation](./helpshelf-backend-api.md)

**Django/Python Backend Analysis** - Comprehensive backend system documentation:

1. **Executive Summary** - Platform overview and key integration points
2. **Database Schema & Data Models** - PostgreSQL models and relationships
3. **System Architecture** - Multi-tenant foundation, technology stack, data flows
4. **API Endpoints** - Guest user management, AI search, content access, analytics
5. **Django Apps Overview** - 5 core apps (Manage, Billing, Crawler, Stats, LLM Core)
6. **Core Services** - AI embedding, payment processing, content extraction, analytics
7. **Admin Interface** - Site, content, subscription, and event administration
8. **Common Patterns** - Authentication, content sync, event tracking patterns
9. **Security & Performance** - Tenant isolation, rate limiting, caching strategies
10. **Integration Architecture** - Widget embedding, iframe communication, direct API access
11. **Development Considerations** - Code organization, testing, deployment

### [📄 Frontend API Documentation](./helpshelf-frontend-api.md)

**React/TypeScript Frontend Architecture** - Complete frontend integration guide:

1. **Executive Summary** - React/TS architecture and backend integration approach
2. **Application Architecture Flow** - Sequential component communication patterns
3. **App.tsx - Entry Point** - URL parameters, routing, and context initialization
4. **OnboardingPage - Progress Setup** - Progress bar, polling, and mobile responsive design
5. **OnboardingContainer - Orchestration** - Context integration and step navigation
6. **Onboarding Reducer - State Management** - 17 action types and state transitions
7. **OnboardingContext - Provider System** - React Context with backend integration
8. **API Communication Layer** - Axios configuration, endpoints, and polling patterns
9. **Backend Integration Patterns** - Django bypass strategy and direct service calls
10. **Key Integration Points** - Data flow sequences and real-time synchronization

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

### Backend
- **Framework**: Django + PostgreSQL with full-text search (GIN indexes)
- **AI/ML**: OpenAI GPT + Milvus vector database for semantic search
- **Infrastructure**: AWS Lambda (crawling), S3 (assets), Redis (caching)
- **Integrations**: Stripe/PayPal (billing), 20+ provider APIs

### Frontend
- **Framework**: React 18+ with TypeScript
- **State Management**: React Context + useReducer pattern
- **Routing**: React Router with URL parameter handling
- **HTTP Client**: Axios with real-time polling
- **Styling**: CSS-in-JS with styled-components
- **Build**: Create React App with Craco configuration

## 📊 Business Impact Metrics

- **AI Response Accuracy**: 40-60% improvement with semantic search
- **Dashboard Performance**: 3-5 seconds → <200ms load times
- **User Satisfaction**: 90%+ with intelligent prompt engineering
- **Content Extraction**: 85%+ accuracy in web scraping

## 🚀 Quick Start for Engineers

### Backend Integration
1. **Read Backend Executive Summary** - Get Django/PostgreSQL platform overview
2. **Review Database Schema** (Backend Section 2) - Understand data models and relationships
3. **Study API Endpoints** (Backend Section 4) - Learn guest user and AI search integration
4. **Explore Core Services** (Backend Section 6) - Understand business logic and data flows

### Frontend Integration  
1. **Read Frontend Executive Summary** - Get React/TypeScript architecture overview
2. **Study App.tsx Flow** (Frontend Section 3) - Understand URL parameters and routing
3. **Review State Management** (Frontend Sections 5-6) - Learn reducer and context patterns
4. **Explore API Communication** (Frontend Section 7) - Understand backend integration patterns

## 📁 File Structure

```
helpshelf-docs/
├── README.md                        # This comprehensive overview document
├── helpshelf-backend-api.md         # Django backend documentation (800+ lines)
└── helpshelf-frontend-api.md        # React frontend documentation (780+ lines)
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