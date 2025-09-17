# HelpShelf Onboarding UI – Getting Started

## Project Overview

The goal of this UI integration project is to launch a new React-based onboarding UI (`helpshelf-ui`) that:

- **Bypasses the Django layer entirely** for improved performance and flexibility
- **Connects directly to backend services** via existing API endpoints (with minor tweaks as needed)
- **Deploys as either**:
  - Full-screen takeover using an iframe within the existing application
  - New browser tab that opens independently from the main platform
- **Provides a modern, mobile-responsive experience** that addresses all current limitations

### Backend Implementation for React Onboarding

The integration follows this step-by-step approach:

1. **Integration Overview** - Architecture flow and key integration points
2. **Backend Requirements Analysis** - Frontend state and API communication patterns
3. **Database Schema Changes** - New models for onboarding configuration
4. **API Endpoint Implementation** - RESTful endpoints for the 3-step process
5. **Domain Analysis Integration** - Website crawling and progress tracking
6. **Real-time Progress System** - WebSocket/polling for live updates
7. **Guest User Enhancement** - UUID-based session management
8. **Analytics Integration** - Event tracking for onboarding metrics
9. **Security Considerations** - CORS, authentication, and rate limiting
10. **Testing Strategy** - Unit, integration, and end-to-end testing
11. **Deployment Plan** - Phased rollout and monitoring approach

---

## Welcome to HelpShelf

Welcome aboard 👋 This document will guide you through setting up both the **React/TypeScript UI** and the **Python/Django backend**, and it will also explain what your first assignment will involve. By the end, you should have a working understanding of how the frontend and backend communicate, and how the new onboarding experience fits into the overall architecture.

---

## 1. Getting the UI Running

The frontend is a standard Create React App project written in TypeScript. It is designed with a clear separation of concerns: all business logic lives in a **Context Provider and Reducer** pattern, while components themselves are kept presentational and only receive props. This ensures the UI remains predictable and easy to maintain.

The project also has a dedicated **UI API layer** built on top of Redux/state management. This layer serves as the single integration point for all data coming into the UI. By funneling everything through this API layer, we can quickly diagnose issues when the data structures returned from the backend don't match what the frontend expects. This design decision reduces debugging time and enforces consistency across the application.

To get the UI running locally, you will need to install dependencies and start the development server. Once running, the app should be accessible in your browser. Since the setup is straightforward, this should not take long to get working.

---

## 2. Getting the Backend Running

The backend is a Django application that powers all APIs. Unlike the UI, this setup may not be as smooth. The previous engineer noted that getting the backend running locally can be challenging, and they were not able to fully configure it on another machine in a short amount of time. As a result, you should expect some troubleshooting when setting up your environment.

The typical process involves creating a Python environment, installing the required dependencies, configuring the database, and applying migrations. However, because the environment configuration may not be fully standardized, you may encounter issues related to missing dependencies, incorrect package versions, or database setup. When this happens, review the project's `settings.py` file for database configuration and check whether Postgres or SQLite is required. Be prepared to spend some time ironing out these setup details.

The goal is to have the backend running locally so that you can access the APIs while developing the frontend. If any setup details remain unclear, document your process as you go so we can improve onboarding for the next engineer.

---

## 3. Backend APIs for Onboarding

Recently, new API endpoints were developed to support the onboarding experience. These endpoints are not yet merged into the `main` branch and currently live on a side branch. Your role will be to review this branch, evaluate the changes that were made, and integrate the new logic into `main`.

When you look at the branch, pay special attention to how serializers, views, and endpoints have been structured. Once integrated, these endpoints should provide the necessary backend functionality for the onboarding experience. It will also be your responsibility to confirm that the API contracts are consistent with what the frontend expects. If there are mismatches, they should be resolved either by adjusting the backend responses or updating the frontend API layer to handle the new structure.

---

## 4. First Assignment – Connecting Onboarding Fields

Your first assignment will focus on connecting two fields in the new onboarding flow to the backend. This task is designed to give you practical, hands-on exposure to both the frontend architecture and the backend APIs.

To complete this assignment, start by reviewing the context and reducer logic that manages onboarding data in the UI. Then determine whether the new endpoints from the side branch should be used, or if it makes more sense to extend the existing endpoints. In either case, all data connections should continue to flow through the UI API layer, ensuring consistency with the existing architecture.

Once you have determined the correct approach, implement the connection end-to-end so that data entered into the UI fields is correctly sent to the backend and persisted. By the end of this assignment, you should have a complete understanding of how the UI and backend communicate during the onboarding process.

---

## Development Philosophy

Our development philosophy emphasizes consistency, maintainability, and clarity:

- **Keep business logic in the context/reducer.** Components should remain as simple as possible and focus only on rendering.
- **Route all data through the UI API layer.** This ensures a single source of truth for data handling and makes debugging easier when discrepancies occur.
- **Integrate incrementally.** Always test and validate an API endpoint in isolation before wiring it into the UI. This prevents hard-to-trace bugs and ensures stable progress.

---

## Integration Guide Reference

For a comprehensive technical deep-dive into the backend implementation requirements, refer to the [Integration Guide](./integration-guide.md). This document provides detailed specifications for:

- Complete database schema changes and model definitions
- Full API endpoint specifications with request/response formats
- Domain analysis and website crawling integration details
- Real-time progress tracking implementation patterns
- Security configurations including CORS and authentication setup
- Comprehensive testing strategies and deployment considerations

The integration guide serves as your technical reference for understanding the complete backend architecture needed to support the new onboarding UI. While this getting-started guide focuses on practical setup and initial tasks, the integration guide contains the detailed specifications you'll need when implementing more complex features.

---

Your first deliverable will be to connect a few of the onboarding fields to the backend, using either the newly developed endpoints (once integrated into `main`) or the existing endpoints if they are sufficient. Completing this task will give you a strong understanding of the project's architecture and prepare you for more advanced feature development.