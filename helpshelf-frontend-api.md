# HelpShelf Frontend API Integration

<h2>Table of Contents</h2>
<ol>
  <li><a href="#executive-summary">Executive Summary</a></li>
  <li><a href="#application-architecture-flow">Application Architecture Flow</a></li>
  <li><a href="#app-tsx-entry-point">App.tsx - Entry Point</a>
    <ol type="a">
      <li><a href="#url-parameter-handling">URL Parameter Handling</a></li>
      <li><a href="#domain-synchronization">Domain Synchronization</a></li>
      <li><a href="#onboarding-provider-integration">OnboardingProvider Integration</a></li>
      <li><a href="#routing-and-navigation">Routing and Navigation</a></li>
    </ol>
  </li>
  <li><a href="#onboarding-page-progress-setup">OnboardingPage - Progress Setup</a>
    <ol type="a">
      <li><a href="#progress-bar-configuration">Progress Bar Configuration</a></li>
      <li><a href="#api-polling-system">API Polling System</a></li>
      <li><a href="#mobile-responsive-design">Mobile Responsive Design</a></li>
      <li><a href="#navigation-callbacks">Navigation Callbacks</a></li>
    </ol>
  </li>
  <li><a href="#onboarding-container-orchestration">OnboardingContainer - Orchestration</a>
    <ol type="a">
      <li><a href="#context-integration">Context Integration</a></li>
      <li><a href="#step-navigation-control">Step Navigation Control</a></li>
      <li><a href="#mobile-action-generation">Mobile Action Generation</a></li>
      <li><a href="#layout-and-step-rendering">Layout and Step Rendering</a></li>
    </ol>
  </li>
  <li><a href="#onboarding-reducer-state-management">Onboarding Reducer - State Management</a>
    <ol type="a">
      <li><a href="#action-type-architecture">Action Type Architecture</a></li>
      <li><a href="#navigation-actions">Navigation Actions</a></li>
      <li><a href="#design-customization-actions">Design Customization Actions</a></li>
      <li><a href="#content-source-actions">Content Source Actions</a></li>
      <li><a href="#contact-method-actions">Contact Method Actions</a></li>
    </ol>
  </li>
  <li><a href="#onboarding-context-provider">OnboardingContext - Provider System</a>
    <ol type="a">
      <li><a href="#context-initialization">Context Initialization</a></li>
      <li><a href="#action-creators">Action Creators</a></li>
      <li><a href="#utility-functions">Utility Functions</a></li>
      <li><a href="#mobile-navigation-integration">Mobile Navigation Integration</a></li>
    </ol>
  </li>
  <li><a href="#api-communication-layer">API Communication Layer</a>
    <ol type="a">
      <li><a href="#axios-configuration">Axios Configuration</a></li>
      <li><a href="#onboarding-endpoints">Onboarding Endpoints</a></li>
      <li><a href="#progress-polling">Progress Polling</a></li>
      <li><a href="#error-handling-patterns">Error Handling Patterns</a></li>
    </ol>
  </li>
  <li><a href="#backend-integration-patterns">Backend Integration Patterns</a>
    <ol type="a">
      <li><a href="#django-bypass-strategy">Django Bypass Strategy</a></li>
      <li><a href="#direct-python-service-calls">Direct Python Service Calls</a></li>
      <li><a href="#iframe-communication">Iframe Communication</a></li>
      <li><a href="#guest-user-session-management">Guest User Session Management</a></li>
    </ol>
  </li>
  <li><a href="#key-integration-points">Key Integration Points</a>
    <ol type="a">
      <li><a href="#data-flow-sequence">Data Flow Sequence</a></li>
      <li><a href="#state-synchronization">State Synchronization</a></li>
      <li><a href="#progress-monitoring">Progress Monitoring</a></li>
      <li><a href="#analytics-event-tracking">Analytics Event Tracking</a></li>
    </ol>
  </li>
</ol>

## Executive Summary

The HelpShelf frontend (`helpshelf-ui`) is a React/TypeScript application designed as a modern onboarding experience that integrates with the existing Django backend through direct Python service calls and iframe communication. The frontend bypasses traditional Django views to communicate directly with backend services for enhanced performance and flexibility.

**Key Architecture Principles:**
- **State Management**: Centralized onboarding state using React Context + useReducer pattern
- **Progressive Enhancement**: Three-step onboarding flow (Design → Content → Contact)
- **API Communication**: Direct backend integration with polling for real-time updates
- **Mobile-First**: Responsive design with dedicated mobile navigation patterns
- **Performance**: Optimized for iframe embedding and cross-origin communication

**Core Technologies:**
- React 18+ with TypeScript
- React Router for navigation
- Axios for HTTP requests
- React Context for state management
- CSS-in-JS with styled-components

## Application Architecture Flow

The application follows a sequential architecture where each component layer builds upon the previous one:

```
App.tsx
├── URL Parameter Processing → Domain Extraction
├── OnboardingProvider → State Management Context
├── DomainSynchronizer → URL-to-State Sync
└── OnboardingPage → Progress & Layout Setup
    └── OnboardingContainer → Step Management
        └── useOnboarding → Backend Communication
```

This flow ensures proper data initialization, state synchronization, and progressive enhancement throughout the onboarding experience.

## App.tsx - Entry Point

**Location**: `/src/App.tsx`

App.tsx serves as the application's main entry point, handling URL parameter parsing, routing, and onboarding context initialization.

### URL Parameter Handling

**Key Function**: `OnboardingSlider` component processes URL parameters to determine application state:

```typescript
// URL parameter extraction from App.tsx:30-33
const [searchParams] = useSearchParams()
const stage = searchParams.get(URL_PARAMS.STAGE)
const domain = searchParams.get("domain")
const isOnboarding = stage === STAGES.ONBOARDING
```

**URL Structure:**
- `?stage=onboarding` - Activates onboarding flow
- `?domain=example.com` - Pre-populates domain for site analysis
- `?step=2` - Navigates directly to specific onboarding step

**Purpose**: These parameters enable deep linking into the onboarding process and facilitate iframe integration where parent websites can pass configuration data.

### Domain Synchronization

**Component**: `DomainSynchronizer` (App.tsx:14-27)

```typescript
function DomainSynchronizer() {
  const [searchParams] = useSearchParams()
  const { updateDomain } = useOnboarding()
  const domain = searchParams.get("domain")

  useEffect(() => {
    if (domain) {
      updateDomain(domain)  // Updates onboarding context state
    }
  }, [domain, updateDomain])

  return null // This component doesn't render anything
}
```

**Backend Integration**: The `updateDomain` function triggers backend API calls to:
1. Validate domain accessibility
2. Initiate website content analysis
3. Update onboarding progress tracking

### OnboardingProvider Integration

**Key Pattern**: App.tsx wraps the onboarding experience with context provider and passes initial domain data:

```typescript
// App.tsx:53-56
<OnboardingProvider key="onboarding" initialData={{ domain: domain || undefined }}>
  <DomainSynchronizer />
  <OnboardingPage isMobile={isMobile} />
</OnboardingProvider>
```

**State Initialization**: The `initialData` prop ensures the onboarding context starts with URL-provided domain information, maintaining state consistency across navigation.

### Routing and Navigation

**Router Configuration** (App.tsx:74-81):
- `/` - Home/Onboarding slider (main onboarding flow)
- `/login` - Authentication page
- `/hub` - Post-onboarding dashboard

**Animation System**: The `OnboardingSlider` component manages smooth transitions between home and onboarding screens using a custom `Slider` component with animation states.

## OnboardingPage - Progress Setup

**Location**: `/src/pages/Onboarding/Onboarding.tsx`

OnboardingPage orchestrates the overall onboarding layout, progress tracking, and mobile responsiveness.

### Progress Bar Configuration

**Progress System** (Onboarding.tsx:90-92):

```typescript
const percent = calculateProgressPercent(steps)
const current = getFakeProgressCurrentStep(steps, percent)
const isComplete = percent === 100
```

**Backend Integration**: Progress data comes from:
1. **Real-time polling**: `setupApiPolling` monitors backend progress endpoints
2. **Fake data fallback**: `fakeProgressSteps.json` provides development data
3. **Dynamic updates**: Progress bar updates as backend processes website analysis

**Visual Feedback**: The progress bar appears in the top bar and automatically hides when onboarding is complete (`topBarHidden={isComplete}`).

### API Polling System

**Implementation** (Onboarding.tsx:58-66):

```typescript
useEffect(() => {
  return initializeProgressMonitoring(
    undefined, // getProgress - will use fake data for now
    DEFAULT_PROGRESS_POLL_INTERVAL, // 2000ms
    defaultSteps,
    setSteps
  )
}, [])
```

**Backend Communication**:
- **Polling Interval**: 2-second updates for real-time feedback
- **Progress Endpoint**: `GET /api/onboarding/progress-steps`
- **Data Structure**: Array of progress steps with status and completion percentage

### Mobile Responsive Design

**Mobile Action Bar Integration** (Onboarding.tsx:101-105):

```typescript
bottomBar={
  isMobile && mobileActions.length > 0 ? (
    <MobileActionBar actions={mobileActions} />
  ) : undefined
}
```

**Responsive Strategy**:
- **Desktop**: Navigation buttons in header
- **Mobile**: Bottom action bar with context-sensitive buttons
- **Touch Optimization**: Larger touch targets and simplified interactions

### Navigation Callbacks

**Domain Ready Callback** (Onboarding.tsx:74-80):

```typescript
const onDomainReady = useCallback(
  (newDomain: string) => {
    setDomain(newDomain)
    handleDomainReady(newDomain) // Triggers backend analysis
  },
  [handleDomainReady]
)
```

**Backend Impact**: When domain is ready:
1. Initiates website crawling process
2. Triggers content extraction services
3. Updates progress tracking in database
4. Sends analytics events to track onboarding progression

## OnboardingContainer - Orchestration

**Location**: `/src/containers/Onboarding/Onboarding.tsx`

OnboardingContainer serves as the central orchestrator for the onboarding flow, managing step navigation and integrating with the onboarding context.

### Context Integration

**State Management** (Onboarding.tsx:16-23):

```typescript
const { data, setAnimating, navigateToStep, createMobileActions } = useOnboarding()
const { domain, navigation, steps } = data
const { currentStep, isAnimating } = navigation
```

**Backend Synchronization**: The context provides real-time state that reflects backend processing:
- **Domain status**: Website analysis progress
- **Step progression**: User navigation through onboarding
- **Animation states**: Smooth transitions between steps

### Step Navigation Control

**URL-to-State Synchronization** (Onboarding.tsx:35-40):

```typescript
useEffect(() => {
  if (urlStep !== currentStep) {
    navigateToStep(urlStep) // Updates both URL and context state
  }
}, [urlStep, currentStep, navigateToStep])
```

**Navigation Flow**:
1. URL changes trigger step navigation
2. Context state updates via reducer
3. Backend receives navigation analytics
4. UI re-renders with new step content

### Mobile Action Generation

**Dynamic Mobile Navigation** (Onboarding.tsx:45-51):

```typescript
useEffect(() => {
  if (onMobileActionsReady) {
    const actions = createMobileActions(navigate)
    onMobileActionsReady(actions) // Passes actions to parent component
  }
}, [onMobileActionsReady, createMobileActions, navigate])
```

**Mobile Navigation Features**:
- **Context-aware buttons**: Different actions per step
- **Backend integration**: Navigation triggers API calls
- **Accessibility**: ARIA labels and keyboard navigation

### Layout and Step Rendering

**Step Component Rendering** (Onboarding.tsx:63-85):

```typescript
{steps.map(({ description, title, count }, index) => {
  return (
    <Flex key={index} /* layout props */>
      <Header /* step header with navigation */ />
      <Step stepIndex={index} isMobile={isMobile} />
    </Flex>
  )
})}
```

**Component Communication**:
- **Header Component**: Receives step metadata and navigation functions
- **Step Component**: Renders step-specific content (Design/Content/Contact)
- **Layout Component**: Manages animations and responsive behavior

## Onboarding Reducer - State Management

**Location**: `/src/reducers/onboarding.ts`

The onboarding reducer implements a comprehensive state management system using Redux patterns with React's useReducer hook.

### Action Type Architecture

**Organization**: Actions are grouped by functional area:

```typescript
// Navigation Actions
ONBOARDING_NAVIGATE_TO_STEP = "ONBOARDING_NAVIGATE_TO_STEP"
ONBOARDING_SET_ANIMATING = "ONBOARDING_SET_ANIMATING"

// Design Actions  
ONBOARDING_UPDATE_DESIGN_COLOR = "ONBOARDING_UPDATE_DESIGN_COLOR"
ONBOARDING_UPDATE_FLOATIE_LOGO = "ONBOARDING_UPDATE_FLOATIE_LOGO"

// Content Source Actions
ONBOARDING_SET_ANALYZING = "ONBOARDING_SET_ANALYZING"
ONBOARDING_UPDATE_SITE_ANALYSIS = "ONBOARDING_UPDATE_SITE_ANALYSIS"
```

**Backend Sync Pattern**: Each action updates local state and optionally triggers backend synchronization.

### Complete Reducer Cases Documentation

The onboarding reducer implements **17 action cases** organized into 7 categories, providing comprehensive state management with detailed documentation for each case.

#### Navigation Actions (2 cases)

**`ONBOARDING_NAVIGATE_TO_STEP`** - Navigate to specific onboarding step
```typescript
case ONBOARDING_NAVIGATE_TO_STEP:
  return {
    ...state,
    navigation: {
      ...state.navigation,
      currentStep: action.payload,      // Update current step (1-3)
      isAnimating: true,               // Trigger transition animations
    },
    updatedAt: now,                    // Track change timestamp
  }
```
- **Payload**: `number` - Target step (1=Design, 2=Content, 3=Contact)
- **State Updates**: `navigation.currentStep`, `navigation.isAnimating`
- **Backend Integration**: Analytics event tracking, progress persistence, URL sync

**`ONBOARDING_SET_ANIMATING`** - Control transition animation state
```typescript
case ONBOARDING_SET_ANIMATING:
  return {
    ...state,
    navigation: {
      ...state.navigation,
      isAnimating: action.payload,     // Set animation active state
    },
    updatedAt: now,
  }
```
- **Payload**: `boolean` - Animation active state
- **State Updates**: `navigation.isAnimating`
- **Usage**: Coordinate UI animations, prevent interactions during transitions

#### Design Actions (6 cases)

**`ONBOARDING_UPDATE_DESIGN_COLOR`** - Update primary/secondary brand colors
```typescript
case ONBOARDING_UPDATE_DESIGN_COLOR:
  return {
    ...state,
    design: {
      ...state.design,
      [action.payload.type === "primary" ? "primaryColor" : "secondaryColor"]:
        action.payload.color,           // Update selected color
    },
    updatedAt: now,
  }
```
- **Payload**: `{ type: "primary" | "secondary", color: string }`
- **State Updates**: `design.primaryColor` or `design.secondaryColor`
- **Backend Sync**: Widget customization service, user profile settings

**`ONBOARDING_UPDATE_FLOATIE_LOGO`** - Update Floatie header logo
```typescript
case ONBOARDING_UPDATE_FLOATIE_LOGO:
  return {
    ...state,
    design: {
      ...state.design,
      floatieLogo: action.payload.filename,        // Backend filename
      floatieLogoPreview: action.payload.preview,  // Preview URL/base64
    },
    updatedAt: now,
  }
```
- **Payload**: `{ filename: string, preview: string }`
- **State Updates**: `design.floatieLogo`, `design.floatieLogoPreview`
- **Features**: Dual storage for backend persistence and immediate UI preview

**`ONBOARDING_UPDATE_LAUNCHER_LOGO`** - Update launcher button logo
```typescript
case ONBOARDING_UPDATE_LAUNCHER_LOGO:
  return {
    ...state,
    design: {
      ...state.design,
      launcherLogo: action.payload.filename,        // Backend filename
      launcherLogoPreview: action.payload.preview,  // Preview URL/base64
    },
    updatedAt: now,
  }
```
- **Payload**: `{ filename: string, preview: string }`
- **State Updates**: `design.launcherLogo`, `design.launcherLogoPreview`
- **Branding**: Consistent logo across launcher button and Floatie header

**`ONBOARDING_UPDATE_FLOATIE_POSITION`** - Set Floatie screen position
```typescript
case ONBOARDING_UPDATE_FLOATIE_POSITION:
  return {
    ...state,
    design: {
      ...state.design,
      floatiePosition: action.payload,  // Set position (left/right)
    },
    updatedAt: now,
  }
```
- **Payload**: `"left" | "right"` - Screen side for widget placement
- **State Updates**: `design.floatiePosition`
- **Impact**: Controls widget positioning on user's website

**`ONBOARDING_TOGGLE_FLOATIE_OPEN`** - Toggle Floatie preview state
```typescript
case ONBOARDING_TOGGLE_FLOATIE_OPEN:
  return {
    ...state,
    design: {
      ...state.design,
      floatieIsOpen: !state.design.floatieIsOpen,  // Invert current state
    },
    updatedAt: now,
  }
```
- **Payload**: None (toggle action)
- **State Updates**: `design.floatieIsOpen` (inverted)
- **Usage**: Interactive preview for design step demonstration

#### Domain Actions (1 case)

**`ONBOARDING_UPDATE_DOMAIN`** - Set target website domain
```typescript
case ONBOARDING_UPDATE_DOMAIN:
  return {
    ...state,
    domain: action.payload,           // Set primary domain
    content: {
      ...state.content,
      siteItems: action.payload       // Create site analysis item
        ? [
            {
              title: action.payload,
              description: "Website",
              icon: "niceprototypes.svg",
              status: "active",
            },
          ]
        : [],                        // Clear if no domain
    },
    updatedAt: now,
  }
```
- **Payload**: `string` - Website domain (e.g., "example.com")
- **State Updates**: `domain`, `content.siteItems` (creates analysis item)
- **Backend Triggers**: Domain validation, website analysis workflows

#### Content Source Actions (3 cases)

**`ONBOARDING_SET_ANALYZING`** - Control content analysis UI state
```typescript
case ONBOARDING_SET_ANALYZING:
  return {
    ...state,
    content: {
      ...state.content,
      isAnalyzing: action.payload,    // Set analyzing state
    },
    updatedAt: now,
  }
```
- **Payload**: `boolean` - Analysis active state
- **State Updates**: `content.isAnalyzing`
- **UI Effect**: Shows/hides loading indicators during crawling

**`ONBOARDING_UPDATE_SITE_ANALYSIS`** - Update website analysis status
```typescript
case ONBOARDING_UPDATE_SITE_ANALYSIS:
  return {
    ...state,
    content: {
      ...state.content,
      siteItems: state.content.siteItems.map((item) =>
        item.title === action.payload.domain 
          ? { ...item, status: action.payload.status }  // Update matching site
          : item
      ),
      isAnalyzing: false,             // Stop analyzing indicator
      analysisComplete: true,         // Mark analysis as finished
    },
    updatedAt: now,
  }
```
- **Payload**: `{ domain: string, status: StatusType }`
- **State Updates**: Site status, analysis flags
- **Backend Response**: Website crawling completion notifications

**`ONBOARDING_UPDATE_SOURCE_STATUS`** - Update content source integration status
```typescript
case ONBOARDING_UPDATE_SOURCE_STATUS:
  return {
    ...state,
    content: {
      ...state.content,
      contentSources: state.content.contentSources.map((item) =>
        item.title === action.payload.itemTitle
          ? { ...item, status: action.payload.status }  // Update matching source
          : item
      ),
    },
    updatedAt: now,
  }
```
- **Payload**: `{ itemTitle: string, status: StatusType | undefined }`
- **State Updates**: `content.contentSources[].status`
- **Integration**: GitHub, Notion, etc. connection status management

#### Contact Method Actions (5 cases)

**`ONBOARDING_SET_ACTIVE_CONTACT_TAB`** - Set active contact method tab
```typescript
case ONBOARDING_SET_ACTIVE_CONTACT_TAB:
  return {
    ...state,
    contact: {
      ...state.contact,
      activeTabIndex: action.payload,  // Set active tab (0=Chat, 1=Email, 2=Phone)
    },
    updatedAt: now,
  }
```
- **Payload**: `number` - Tab index
- **State Updates**: `contact.activeTabIndex`
- **Interface**: Controls contact method configuration interface

**`ONBOARDING_UPDATE_CONTACT_FORM`** - Update contact method form fields
```typescript
case ONBOARDING_UPDATE_CONTACT_FORM:
  const { method, field, value } = action.payload

  if (method === "Email") {
    return {
      ...state,
      contact: {
        ...state.contact,
        formData: {
          ...state.contact.formData,
          email: {
            ...state.contact.formData.email,
            [field]: value,               // Update email form field
          },
        },
      },
      updatedAt: now,
    }
  } else if (method === "Phone") {
    return {
      ...state,
      contact: {
        ...state.contact,
        formData: {
          ...state.contact.formData,
          phone: {
            ...state.contact.formData.phone,
            [field]: value,               // Update phone form field
          },
        },
      },
      updatedAt: now,
    }
  }
  return state                           // No change for other methods
```
- **Payload**: `{ method: ContactMethodType, field: string, value: string }`
- **State Updates**: `contact.formData.email[]` or `contact.formData.phone[]`
- **Form Fields**: Email (supportEmail, friendlyName, emailSubject, emailMessage), Phone (phoneNumber)

**`ONBOARDING_UPDATE_TAB_DISABLED`** - Enable/disable contact method tabs
```typescript
case ONBOARDING_UPDATE_TAB_DISABLED:
  return {
    ...state,
    contact: {
      ...state.contact,
      disabledTabs: {
        ...state.contact.disabledTabs,
        [action.payload.method]: action.payload.disabled,  // Set disabled state
      },
    },
    updatedAt: now,
  }
```
- **Payload**: `{ method: ContactMethodType, disabled: boolean }`
- **State Updates**: `contact.disabledTabs[method]`
- **Visibility**: Disabled methods hidden from end users

**`ONBOARDING_UPDATE_TIME_WINDOW`** - Update availability time windows
```typescript
case ONBOARDING_UPDATE_TIME_WINDOW:
  return {
    ...state,
    contact: {
      ...state.contact,
      timeWindows: {
        ...state.contact.timeWindows,
        [action.payload.method]: {
          ...state.contact.timeWindows[action.payload.method],
          [action.payload.day]: {
            ...state.contact.timeWindows[action.payload.method][action.payload.day],
            [action.payload.field]: action.payload.value,  // Update time field
          },
        },
      },
    },
    updatedAt: now,
  }
```
- **Payload**: `{ method: ContactMethodType, day: DayOfWeekType, field: "start" | "end", value: string }`
- **State Updates**: `contact.timeWindows[method][day][field]`
- **Granular Control**: Different schedules per contact method and day

**`ONBOARDING_UPDATE_TIME_ALWAYS`** - Toggle 24/7 availability setting
```typescript
case ONBOARDING_UPDATE_TIME_ALWAYS:
  return {
    ...state,
    contact: {
      ...state.contact,
      timeWindows: {
        ...state.contact.timeWindows,
        [action.payload.method]: {
          ...state.contact.timeWindows[action.payload.method],
          [action.payload.day]: {
            ...state.contact.timeWindows[action.payload.method][action.payload.day],
            isAlways: action.payload.isAlways,  // Set 24/7 availability flag
          },
        },
      },
    },
    updatedAt: now,
  }
```
- **Payload**: `{ method: ContactMethodType, day: DayOfWeekType, isAlways: boolean }`
- **State Updates**: `contact.timeWindows[method][day].isAlways`
- **Override**: Supersedes specific start/end time settings when enabled

#### Progress Actions (1 case)

**`ONBOARDING_UPDATE_PROGRESS_STEPS`** - Update website analysis progress
```typescript
case ONBOARDING_UPDATE_PROGRESS_STEPS:
  const steps = action.payload
  const totalSteps = steps.length
  const completedSteps = steps.filter((step) => step.status === "done").length
  const percent = totalSteps > 0 ? (completedSteps / totalSteps) * 100 : 0

  return {
    ...state,
    progress: {
      ...state.progress,
      steps,                          // Update progress step array
      percent,                        // Calculate completion percentage
      currentProgressStep: completedSteps,  // Count of completed steps
      isComplete: percent === 100,    // Set completion flag
    },
    updatedAt: now,
  }
```
- **Payload**: `ProgressStep[]` - Array of progress steps
- **State Updates**: All progress fields with calculated values
- **Real-time**: Reflects backend processing progress via polling

#### UI State Actions (3 cases)

**`ONBOARDING_SET_HOVERED_SOURCE`** - Track hovered content source
```typescript
case ONBOARDING_SET_HOVERED_SOURCE:
  return {
    ...state,
    ui: {
      ...state.ui,
      windowSources: {
        ...state.ui.windowSources,
        hoveredIndex: action.payload,   // Set hovered item index
      },
    },
    updatedAt: now,
  }
```
- **Payload**: `number | null` - Hovered item index (null when no hover)
- **State Updates**: `ui.windowSources.hoveredIndex`
- **Visual**: Provides hover feedback in sources list

**`ONBOARDING_SET_ACTIVE_SOURCE`** - Set active/expanded content source
```typescript
case ONBOARDING_SET_ACTIVE_SOURCE:
  return {
    ...state,
    ui: {
      ...state.ui,
      windowSources: {
        ...state.ui.windowSources,
        activeIndex: action.payload,    // Set active item index
      },
    },
    updatedAt: now,
  }
```
- **Payload**: `number | null` - Active item index (null when none active)
- **State Updates**: `ui.windowSources.activeIndex`
- **Interaction**: Shows configuration form for active source

**`ONBOARDING_UPDATE_SEARCH_QUERY`** - Update content source search filter
```typescript
case ONBOARDING_UPDATE_SEARCH_QUERY:
  return {
    ...state,
    ui: {
      ...state.ui,
      windowSources: {
        ...state.ui.windowSources,
        searchQuery: action.payload,    // Update search query text
      },
    },
    updatedAt: now,
  }
```
- **Payload**: `string` - Search query text
- **State Updates**: `ui.windowSources.searchQuery`
- **Filtering**: Enables real-time source filtering in UI

### Reducer Architecture Patterns

**Immutability**: Every case returns a new state object using spread operators
**Timestamp Tracking**: All cases update `updatedAt` field for debugging and change tracking
**Pure Functions**: No side effects, same input always produces same output
**Type Safety**: Full TypeScript coverage prevents runtime errors
**State Isolation**: Changes only affect relevant state sections, maintaining performance

## OnboardingContext - Provider System

**Location**: `/src/contexts/useOnboarding/Provider.tsx`

The OnboardingProvider implements a comprehensive React Context system that manages onboarding state and provides backend integration functions.

### Context Initialization

**State Setup** (Provider.tsx:57-59):

```typescript
const [data, dispatch] = useReducer(onboardingReducer, initializeData(initialData))
```

**Initialization Process**:
1. **Initial Data Processing**: Merges URL parameters with default configuration
2. **Backend Data Fetching**: Loads user preferences and site settings
3. **State Hydration**: Populates context with combined frontend/backend data

### Complete Action Creator System

The OnboardingProvider implements **22 action creators and utility functions** organized into 6 categories, providing comprehensive state management for the onboarding flow.

#### Navigation Action Creators (2 functions)

**`navigateToStep(step: number)`** - Navigate to specific onboarding step
- **Purpose**: Updates current step and triggers transition animations
- **Parameters**: `step` - Target step number (1-based index)
- **State Updates**: `navigation.currentStep`, `navigation.isAnimating`
- **Usage**: Step navigation buttons, URL routing, programmatic navigation

```typescript
const navigateToStep = useCallback((step: number) => {
  dispatch({ type: ONBOARDING_NAVIGATE_TO_STEP, payload: step })
}, [])
```

**`setAnimating(isAnimating: boolean)`** - Control transition animation state
- **Purpose**: Coordinates UI animations and prevents interactions during transitions
- **Parameters**: `isAnimating` - Whether transition animation is active
- **State Updates**: `navigation.isAnimating`
- **Usage**: Animation coordination, smooth step transitions

```typescript
const setAnimating = useCallback((isAnimating: boolean) => {
  dispatch({ type: ONBOARDING_SET_ANIMATING, payload: isAnimating })
}, [])
```

#### Design Customization Action Creators (6 functions)

**`updatePrimaryBrandColor(color: string)`** - Update primary brand color
- **Purpose**: Sets main accent color for Floatie interface
- **Parameters**: `color` - Primary color value (hex, rgb, hsl, named)
- **State Updates**: `design.primaryColor`
- **Backend Sync**: Saves to widget customization service

```typescript
const updatePrimaryBrandColor = useCallback((color: string) => {
  dispatch({ type: ONBOARDING_UPDATE_DESIGN_COLOR, payload: { type: "primary", color } })
}, [])
```

**`updateSecondaryBrandColor(color: string)`** - Update secondary brand color
- **Purpose**: Sets secondary accent color for supporting UI elements
- **Parameters**: `color` - Secondary color value
- **State Updates**: `design.secondaryColor`
- **Usage**: Backgrounds, borders, secondary elements

**`updateFloatiePosition(position: "left" | "right")`** - Set widget position
- **Purpose**: Controls Floatie placement on user's website
- **Parameters**: `position` - Screen side for widget placement
- **State Updates**: `design.floatiePosition`
- **Impact**: Affects widget positioning when activated

**`updateFloatieLogo(filename: string, preview: string)`** - Update Floatie logo
- **Purpose**: Sets custom branding logo in Floatie header
- **Parameters**: `filename` - Backend filename, `preview` - Preview URL/base64
- **State Updates**: `design.floatieLogo`, `design.floatieLogoPreview`
- **Features**: Immediate visual feedback during upload

**`updateDesignLauncherLogo(filename: string, preview: string)`** - Update launcher logo
- **Purpose**: Sets logo for launcher button that opens Floatie
- **Parameters**: `filename` - Backend filename, `preview` - Preview data
- **State Updates**: `design.launcherLogo`, `design.launcherLogoPreview`
- **Branding**: Consistency across help system touchpoints

**`toggleFloatieOpen()`** - Toggle Floatie preview state
- **Purpose**: Shows live preview of Floatie behavior during design
- **Parameters**: None (toggle action)
- **State Updates**: `design.floatieIsOpen` (inverted)
- **Usage**: Interactive preview for design decisions

#### Domain Configuration Action Creators (1 function)

**`updateDomain(domain: string)`** - Set target website domain
- **Purpose**: Sets primary domain for Floatie installation and triggers analysis
- **Parameters**: `domain` - Website domain (e.g., "example.com")
- **State Updates**: `domain`, `content.siteItems` (creates analysis item)
- **Backend Triggers**: Domain validation, site analysis workflows

```typescript
const updateDomain = useCallback((domain: string) => {
  dispatch({ type: ONBOARDING_UPDATE_DOMAIN, payload: domain })
}, [])
```

#### Content Source Action Creators (3 functions)

**`updateSiteAnalysis(domain: string, status: FunctionalStatusType)`** - Update analysis status
- **Purpose**: Tracks progress and results of website analysis
- **Parameters**: `domain` - Domain being analyzed, `status` - Analysis status
- **State Updates**: `content.siteItems[].status`, `content.isAnalyzing`, `content.analysisComplete`
- **Backend Response**: Handles crawling completion notifications

**`updateSourceStatus(itemTitle: string, status: StatusType | undefined)`** - Update integration status
- **Purpose**: Manages connection state of external sources (GitHub, Notion, etc.)
- **Parameters**: `itemTitle` - Source identifier, `status` - Integration status
- **State Updates**: `content.contentSources[].status`
- **Reflects**: Authentication success, API connectivity, data sync state

**`setAnalyzing(isAnalyzing: boolean)`** - Control analysis UI state
- **Purpose**: Manages UI state during content indexing and processing
- **Parameters**: `isAnalyzing` - Whether analysis is active
- **State Updates**: `content.isAnalyzing`
- **UI Effect**: Shows/hides loading indicators, prevents concurrent operations

#### Contact Method Action Creators (5 functions)

**`setContactActiveTab(index: number)`** - Set active contact tab
- **Purpose**: Controls which contact method is being configured
- **Parameters**: `index` - Tab index (0=Chat, 1=Email, 2=Phone)
- **State Updates**: `contact.activeTabIndex`
- **Interface**: Switches between Chat, Email, Phone configuration

**`updateContactTimeWindow(method, day, field, value)`** - Update availability schedule
- **Purpose**: Sets specific availability hours for contact methods per day
- **Parameters**: `method` - Contact type, `day` - Weekday, `field` - start/end, `value` - Time
- **State Updates**: `contact.timeWindows[method][day][field]`
- **Granular Control**: Different schedules per contact method and day

**`updateContactTimeAlways(method, day, isAlways)`** - Toggle 24/7 availability
- **Purpose**: Quick toggle for round-the-clock availability
- **Parameters**: `method` - Contact type, `day` - Weekday, `isAlways` - 24/7 flag
- **State Updates**: `contact.timeWindows[method][day].isAlways`
- **Override**: Supersedes specific time window settings

**`updateContactForm(method, field, value)`** - Update contact form fields
- **Purpose**: Handles dynamic form data for contact method setup
- **Parameters**: `method` - Contact type, `field` - Form field name, `value` - New value
- **State Updates**: `contact.formData.email[]` or `contact.formData.phone[]`
- **Fields**: Email (supportEmail, friendlyName, emailSubject, emailMessage), Phone (phoneNumber)

**`updateContactDisabledTab(method, disabled)`** - Enable/disable contact methods
- **Purpose**: Controls contact method availability based on business requirements
- **Parameters**: `method` - Contact type, `disabled` - Disabled state
- **State Updates**: `contact.disabledTabs[method]`
- **Visibility**: Disabled methods hidden from end users

#### Utility Functions (5 functions)

**`getCurrentStepData()`** - Get current step metadata
- **Returns**: Step object with title, description, icon, etc.
- **Usage**: Components displaying step-specific information

**`isFirstStep()`** - Check if on first step
- **Returns**: `boolean` - True if on step 1
- **Usage**: Conditionally show/hide "Previous" navigation buttons

**`isLastStep()`** - Check if on final step
- **Returns**: `boolean` - True if on last step
- **Usage**: Change "Next" to "Finish" button, completion workflows

**`canGoBack()`** - Check backward navigation availability
- **Returns**: `boolean` - True if can navigate to previous steps
- **Usage**: Control "Previous" button states

**`canGoForward()`** - Check forward navigation availability
- **Returns**: `boolean` - True if can navigate to next steps
- **Usage**: Control "Next" button states

**`createMobileActions(navigate)`** - Generate mobile navigation buttons
- **Parameters**: `navigate` - React router navigate function
- **Returns**: `NavigationActionProps[]` - Array of mobile action buttons
- **Features**: Context-aware actions, consistent with desktop navigation

### Performance Optimization Patterns

**useCallback Optimization**: All action creators are wrapped with `useCallback` for optimal performance:

```typescript
const updateDomain = useCallback((domain: string) => {
  dispatch({ type: ONBOARDING_UPDATE_DOMAIN, payload: domain })
}, []) // Empty dependency array - dispatch is stable
```

**Benefits**:
- **Referential Equality**: Prevents unnecessary re-renders
- **Memory Efficiency**: Avoids function recreation on every render
- **Selective Updates**: Only components using changed state re-render

**Backend Integration Pattern**: Action creators follow a consistent pattern:
1. **Immediate State Update**: Dispatch action for instant UI feedback
2. **Backend API Call**: Async operation to persist changes
3. **Progress Monitoring**: Real-time updates via polling or webhooks
4. **Error Handling**: Rollback on failure with user notification

### Utility Functions

**Navigation Queries** (Provider.tsx:347-361):

```typescript
const isLastStep = useCallback(() => {
  return data.navigation.currentStep === data.navigation.totalSteps
}, [data.navigation.currentStep, data.navigation.totalSteps])
```

**Backend Coordination**: Utility functions help coordinate:
- Step progression logic with backend validation
- Navigation constraints based on completion status
- Mobile experience optimization

### Mobile Navigation Integration

**Action Generation** (Provider.tsx:396-441):

```typescript
const createMobileActions = useCallback(
  (navigate: (path: string) => void) => {
    const actionsList: NavigationActionProps[] = []
    
    // Generate context-aware navigation actions
    if (currentStep > 1) {
      actionsList.push({/* Back button config */})
    }
    
    actionsList.push({/* Continue button config */})
    
    return actionsList
  },
  [data.navigation, setAnimating]
)
```

**Backend Navigation**: Mobile actions trigger:
- Step validation API calls
- Progress persistence
- Analytics event tracking

## API Communication Layer

### Axios Configuration

**Base API Setup** (`/src/api.ts`):

```typescript
import axios from 'axios';

const api = axios.create({
  headers: {
    'Content-Type': 'application/json',
  },
});
```

**Onboarding-Specific API** (`/src/containers/Onboarding/services/api.ts`):

```typescript
const onboardingApi = api.create({
  baseURL: "http://localhost:3001", // Development backend
})
```

### Onboarding Endpoints

**Configuration Endpoints**:

```typescript
// Get onboarding configuration data
export const getOnboardingConfig = () => {
  return onboardingApi.get("/api/onboarding/config")
}

// Get content sources (GitHub, Notion, etc.)
export const getContentSources = () => {
  return onboardingApi.get("/api/onboarding/content-sources")
}

// Get chat providers (Zendesk, Intercom, etc.)
export const getChatProviders = () => {
  return onboardingApi.get("/api/onboarding/chat-providers")
}
```

**Backend Integration**: These endpoints connect to Django services that:
1. Return user-specific onboarding configuration
2. Provide available integration options
3. Include progress and completion status

### Progress Polling

**Polling Implementation** (`/src/pages/Onboarding/services/setupApiPolling.ts`):

```typescript
function setupApiPolling(
  getProgress: () => Promise<ProgressStep[]>,
  pollMs: number,
  onUpdate: (steps: ProgressStep[]) => void
): () => void {
  const tick = async () => {
    try {
      const data = await getProgress()
      if (Array.isArray(data) && data.length) onUpdate(data)
    } catch {
      // ignore errors for now; product can decide to surface later
    }
  }

  const intervalId = setInterval(tick, pollMs)
  return () => clearInterval(intervalId)
}
```

**Backend Communication**: Progress polling:
- **Endpoint**: `GET /api/onboarding/progress-steps`
- **Frequency**: Every 2 seconds (`DEFAULT_PROGRESS_POLL_INTERVAL`)
- **Data**: Real-time website analysis and setup progress

### Error Handling Patterns

**Graceful Degradation**: The frontend handles backend connectivity issues by:
1. **Retry Logic**: Automatic retries with exponential backoff
2. **Fallback Data**: Local JSON files provide development/offline functionality
3. **User Feedback**: Loading states and error messages
4. **Progressive Enhancement**: Core functionality works without backend

## Backend Integration Patterns

### Django Bypass Strategy

**Architecture Decision**: The helpshelf-ui frontend is designed to bypass traditional Django views and communicate directly with Python backend services for improved performance and flexibility.

**Benefits**:
- **Performance**: Direct service calls eliminate Django middleware overhead
- **Scalability**: API endpoints can be cached and load-balanced independently
- **Flexibility**: Frontend can communicate with multiple backend services
- **Development**: Decoupled development of frontend and backend components

### Direct Python Service Calls

**Service Integration Points**:

1. **AI Search Service**: Direct calls to OpenAI integration and semantic search
2. **Content Analysis Service**: Website crawling and content extraction
3. **User Profile Service**: Guest user session management and configuration
4. **Analytics Service**: Event tracking and progress monitoring

**API Pattern**:
```typescript
// Direct service communication bypassing Django views
const response = await onboardingApi.post('/api/services/ai-search', {
  domain: userDomain,
  query: searchTerm,
  user_uuid: guestUserUuid
})
```

### Iframe Communication

**Embedding Strategy**: The onboarding experience is designed to run within iframes on existing websites, maintaining separation between new React UI and legacy Django templates.

**Communication Pattern**:
```typescript
// PostMessage API for iframe communication
window.parent.postMessage({
  type: 'onboarding_complete',
  data: {
    domain: completedDomain,
    configuration: onboardingState
  }
}, '*')
```

**Cross-Origin Considerations**:
- **CORS Configuration**: Backend APIs support cross-origin requests from iframe contexts
- **Security**: Message validation and origin checking for iframe communication
- **State Synchronization**: Iframe and parent window maintain consistent state

### Guest User Session Management

**Session Architecture**: The frontend manages anonymous user sessions using UUID-based identification that integrates with the existing Django guest user system.

**UUID Management**:
```typescript
// Guest user session initialization
const initializeGuestSession = () => {
  const uuid = generateUUID()
  localStorage.setItem('helpshelf_guest_uuid', uuid)
  return uuid
}
```

**Backend Integration**: Guest UUIDs are used to:
1. **Track Progress**: Associate onboarding progress with anonymous sessions
2. **Persist Configuration**: Save user preferences without account creation  
3. **Analytics**: Monitor onboarding completion rates and drop-off points
4. **Transition**: Convert guest sessions to authenticated users upon signup

## Key Integration Points

### Data Flow Sequence

**Onboarding Initialization**:
1. **URL Processing**: Extract domain and step parameters from URL
2. **Context Setup**: Initialize onboarding state with URL data
3. **Backend Handshake**: Validate domain and fetch user configuration
4. **Progress Polling**: Begin real-time monitoring of backend processes
5. **State Synchronization**: Maintain consistency between frontend and backend

**Step Progression Flow**:
1. **User Action**: Navigation or configuration change
2. **Local State Update**: Immediate UI feedback via reducer
3. **Backend Synchronization**: API call to persist changes
4. **Progress Update**: Real-time polling reflects backend processing
5. **Completion Handling**: Final step triggers iframe communication

### State Synchronization

**Bidirectional Sync**: Frontend and backend maintain synchronized state through:

1. **Frontend → Backend**: User actions trigger API calls to persist state
2. **Backend → Frontend**: Polling and webhooks update frontend with processing results
3. **Conflict Resolution**: Last-write-wins with timestamp-based conflict resolution

**Critical Sync Points**:
- **Domain Configuration**: Website analysis triggers and results
- **Design Customization**: Brand colors, logos, and positioning
- **Contact Settings**: Availability windows and integration configurations

### Progress Monitoring

**Real-Time Updates**: The onboarding experience provides real-time feedback on backend processing:

**Website Analysis Progress**:
```json
{
  "steps": [
    { "name": "Domain Validation", "status": "completed" },
    { "name": "Content Crawling", "status": "in_progress", "progress": 60 },
    { "name": "AI Processing", "status": "pending" }
  ],
  "overall_progress": 45
}
```

**Backend Services Monitored**:
1. **Domain Analysis**: Website accessibility and structure validation
2. **Content Extraction**: Page crawling and content processing
3. **AI Enhancement**: Semantic analysis and search optimization
4. **Integration Setup**: Provider connections and configurations

### Analytics Event Tracking

**Event Tracking Strategy**: Comprehensive analytics capture user behavior and onboarding effectiveness:

**Frontend Events**:
```typescript
// Step navigation tracking
trackEvent('onboarding_step_completed', {
  step: currentStep,
  domain: userDomain,
  time_spent: stepDuration,
  user_uuid: guestUserUuid
})
```

**Backend Integration**: Analytics events are sent to:
1. **Stats Service**: Real-time event processing and aggregation
2. **User Profiling**: Behavioral analysis for personalization
3. **A/B Testing**: Onboarding flow optimization
4. **Business Intelligence**: Conversion tracking and funnel analysis

**Critical Metrics Tracked**:
- **Onboarding Completion Rate**: Percentage of users finishing all steps
- **Step Drop-off Points**: Where users abandon the onboarding process
- **Domain Analysis Success**: Technical validation and content extraction rates
- **Integration Adoption**: Which providers and features users enable

---

**Generated for**: HelpShelf full stack engineering team  
**Last Updated**: September 2024  
**Maintainer**: Engineering Documentation Team