# HelpShelf Onboarding – Integration Guide

## Table of Contents

1. [Integration Overview](#integration-overview)
2. [Backend Requirements Analysis](#backend-requirements-analysis)
3. [Database Schema Changes](#database-schema-changes)
4. [API Endpoint Implementation](#api-endpoint-implementation)
5. [Domain Analysis Integration](#domain-analysis-integration)
6. [Real-time Progress System](#real-time-progress-system)
7. [Guest User Enhancement](#guest-user-enhancement)
8. [Analytics Integration](#analytics-integration)
9. [Security Considerations](#security-considerations)
10. [Testing Strategy](#testing-strategy)
11. [Deployment Plan](#deployment-plan)

## Integration Overview

The new React/TypeScript onboarding experience (`helpshelf-ui`) requires minimal backend modifications to support a streamlined 3-step onboarding process (Design → Content → Contact) while maintaining compatibility with the existing Django architecture.

### Architecture Flow
```
Existing Django UI → Domain Entry → helpshelf-ui Onboarding → Backend API Integration
```

**Key Integration Points:**
- **Domain Handoff**: Django captures domain, passes to React onboarding
- **Progress Tracking**: Real-time website analysis monitoring
- **Configuration Storage**: Design, content, and contact preferences
- **Guest User Management**: UUID-based session handling
- **Analytics Events**: Onboarding completion tracking

## Backend Requirements Analysis

Based on the frontend API documentation, the backend needs to support the following functionality:

### Frontend State Requirements
1. **Navigation State**: Step progression tracking
2. **Design Configuration**: Brand colors, logos, positioning
3. **Content Analysis**: Website crawling and source integration
4. **Contact Setup**: Availability schedules and provider configuration
5. **Progress Monitoring**: Real-time analysis status updates

### API Communication Patterns
- **Configuration Endpoints**: GET/POST for onboarding data
- **Progress Polling**: Real-time status updates (2-second intervals)
- **File Upload**: Logo management for branding
- **Analytics Events**: Step completion tracking

## Database Schema Changes

### 1. Onboarding Configuration Model (NEW)

**Action**: **ADD** new model to store onboarding state

```python
# helpshelf/apps/manage/models.py (or new onboarding app)

class OnboardingConfiguration(models.Model):
    """
    Stores onboarding configuration for sites during setup process
    """
    # Site relationship
    site = models.OneToOneField(
        Site, 
        on_delete=models.CASCADE,
        related_name='onboarding_config'
    )
    
    # Design configuration
    primary_color = models.CharField(max_length=7, default="#FB8537")  # Hex color
    secondary_color = models.CharField(max_length=7, default="#4884FA")
    floatie_position = models.CharField(
        max_length=10, 
        choices=[('left', 'Left'), ('right', 'Right')],
        default='right'
    )
    floatie_logo = models.FileField(upload_to='onboarding/logos/', null=True, blank=True)
    launcher_logo = models.FileField(upload_to='onboarding/launchers/', null=True, blank=True)
    
    # Progress tracking
    onboarding_step = models.IntegerField(default=1)  # 1=Design, 2=Content, 3=Contact
    is_completed = models.BooleanField(default=False)
    
    # Contact configuration
    contact_methods = models.JSONField(default=dict)  # Store timeWindows, formData
    disabled_contact_tabs = models.JSONField(default=dict)
    
    # Timestamps
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        db_table = 'onboarding_configuration'
```

**Benefits**:
- Isolated onboarding data doesn't pollute existing Site model
- JSON fields provide flexibility for complex contact configurations
- One-to-one relationship maintains data integrity

**Risks**:
- New table adds complexity
- JSON fields require careful validation
- Migration required for existing sites

### 2. Website Analysis Progress Model (NEW)

**Action**: **ADD** new model for real-time progress tracking

```python
class WebsiteAnalysisProgress(models.Model):
    """
    Tracks real-time progress of website analysis during onboarding
    """
    site = models.OneToOneField(
        Site,
        on_delete=models.CASCADE,
        related_name='analysis_progress'
    )
    
    # Progress tracking
    steps = models.JSONField(default=list)  # Array of ProgressStep objects
    current_step = models.IntegerField(default=0)
    total_steps = models.IntegerField(default=0)
    completion_percentage = models.FloatField(default=0.0)
    
    # Analysis status
    is_analyzing = models.BooleanField(default=False)
    is_complete = models.BooleanField(default=False)
    
    # Error handling
    error_message = models.TextField(null=True, blank=True)
    retry_count = models.IntegerField(default=0)
    
    # Timestamps
    started_at = models.DateTimeField(null=True, blank=True)
    completed_at = models.DateTimeField(null=True, blank=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        db_table = 'website_analysis_progress'
```

**Benefits**:
- Real-time progress tracking for frontend polling
- Error handling and retry logic support
- Clean separation from existing crawling logic

**Risks**:
- Additional database queries for polling
- JSON field complexity for step management

### 3. Guest User Session Enhancement (MODIFY)

**Action**: **MODIFY** existing GuestUserProfile to support onboarding

```python
# helpshelf/apps/profile/guest_user_models.py

class GuestUserProfile(models.Model):
    # Existing fields...
    
    # ADD: Onboarding tracking
    onboarding_domain = models.CharField(max_length=255, null=True, blank=True)
    onboarding_step = models.IntegerField(default=0)  # 0=not started, 1-3=steps
    onboarding_completed_at = models.DateTimeField(null=True, blank=True)
    
    # ADD: Configuration preferences (before site creation)
    temp_configuration = models.JSONField(default=dict, blank=True)
```

**Benefits**:
- Tracks onboarding progress for anonymous users
- Stores temporary configuration before site creation
- Maintains existing guest user functionality

**Risks**:
- Migration required for existing guest users
- JSON field adds complexity

## API Endpoint Implementation

### 1. Onboarding Configuration Endpoints (NEW)

**Action**: **ADD** new API endpoints for onboarding management

```python
# helpshelf/apps/api/onboarding_views.py (NEW FILE)

from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from django.shortcuts import get_object_or_404
from ..manage.models import Site, OnboardingConfiguration
from ..profile.guest_user_models import GuestUserProfile

@api_view(['GET', 'POST'])
def onboarding_config(request, user_uuid=None):
    """
    GET: Retrieve current onboarding configuration
    POST: Update onboarding configuration
    """
    # Get guest user or create new session
    guest_user = get_object_or_404(GuestUserProfile, uuid=user_uuid) if user_uuid else None
    
    if request.method == 'GET':
        if guest_user and guest_user.onboarding_domain:
            try:
                site = Site.objects.get(domain=guest_user.onboarding_domain)
                config = getattr(site, 'onboarding_config', None)
                return Response({
                    'domain': guest_user.onboarding_domain,
                    'currentStep': guest_user.onboarding_step,
                    'configuration': serialize_onboarding_config(config) if config else {},
                    'isCompleted': bool(guest_user.onboarding_completed_at)
                })
            except Site.DoesNotExist:
                pass
        
        # Return default configuration
        return Response({
            'domain': '',
            'currentStep': 1,
            'configuration': get_default_onboarding_config(),
            'isCompleted': False
        })
    
    elif request.method == 'POST':
        data = request.data
        
        # Update guest user tracking
        if guest_user:
            if 'domain' in data:
                guest_user.onboarding_domain = data['domain']
            if 'currentStep' in data:
                guest_user.onboarding_step = data['currentStep']
            if 'configuration' in data:
                guest_user.temp_configuration.update(data['configuration'])
            guest_user.save()
        
        # Update site configuration if site exists
        if data.get('domain'):
            site, created = Site.objects.get_or_create(domain=data['domain'])
            config, created = OnboardingConfiguration.objects.get_or_create(site=site)
            
            # Update configuration fields
            update_onboarding_configuration(config, data.get('configuration', {}))
            config.onboarding_step = data.get('currentStep', config.onboarding_step)
            config.save()
        
        return Response({'status': 'updated'}, status=status.HTTP_200_OK)

def serialize_onboarding_config(config):
    """Serialize OnboardingConfiguration to frontend format"""
    return {
        'design': {
            'primaryColor': config.primary_color,
            'secondaryColor': config.secondary_color,
            'floatiePosition': config.floatie_position,
            'floatieLogo': config.floatie_logo.url if config.floatie_logo else '',
            'launcherLogo': config.launcher_logo.url if config.launcher_logo else '',
        },
        'contact': config.contact_methods,
        'disabledTabs': config.disabled_contact_tabs,
    }

def update_onboarding_configuration(config, data):
    """Update OnboardingConfiguration from frontend data"""
    design = data.get('design', {})
    if 'primaryColor' in design:
        config.primary_color = design['primaryColor']
    if 'secondaryColor' in design:
        config.secondary_color = design['secondaryColor']
    if 'floatiePosition' in design:
        config.floatie_position = design['floatiePosition']
    
    if 'contact' in data:
        config.contact_methods.update(data['contact'])
    if 'disabledTabs' in data:
        config.disabled_contact_tabs.update(data['disabledTabs'])
```

**Benefits**:
- Clean separation between guest users and site configuration
- Progressive enhancement from guest to authenticated user
- Flexible JSON storage for complex frontend state

**Risks**:
- Additional API surface area to maintain
- Complex state synchronization between guest and site data

### 2. Progress Polling Endpoint (NEW)

**Action**: **ADD** real-time progress tracking endpoint

```python
@api_view(['GET'])
def progress_steps(request, user_uuid=None):
    """
    Real-time progress tracking for website analysis
    Polled every 2 seconds by frontend
    """
    guest_user = get_object_or_404(GuestUserProfile, uuid=user_uuid) if user_uuid else None
    
    if not guest_user or not guest_user.onboarding_domain:
        return Response({
            'steps': [],
            'percent': 0,
            'isComplete': False,
            'isAnalyzing': False
        })
    
    try:
        site = Site.objects.get(domain=guest_user.onboarding_domain)
        progress = getattr(site, 'analysis_progress', None)
        
        if not progress:
            # Initialize progress tracking
            progress = WebsiteAnalysisProgress.objects.create(
                site=site,
                steps=get_default_progress_steps(),
                total_steps=4
            )
            
            # Trigger website analysis
            trigger_website_analysis.delay(site.id)
        
        return Response({
            'steps': progress.steps,
            'percent': progress.completion_percentage,
            'currentStep': progress.current_step,
            'isComplete': progress.is_complete,
            'isAnalyzing': progress.is_analyzing,
            'error': progress.error_message
        })
        
    except Site.DoesNotExist:
        return Response({
            'steps': [],
            'percent': 0,
            'isComplete': False,
            'isAnalyzing': False,
            'error': 'Site not found'
        })

def get_default_progress_steps():
    """Default progress steps for website analysis"""
    return [
        {"name": "Domain Validation", "status": "pending", "progress": 0},
        {"name": "Content Crawling", "status": "pending", "progress": 0},
        {"name": "AI Processing", "status": "pending", "progress": 0},
        {"name": "Integration Setup", "status": "pending", "progress": 0}
    ]
```

**Benefits**:
- Real-time feedback improves user experience
- Lazy initialization of progress tracking
- Error handling and fallback support

**Risks**:
- High polling frequency increases server load
- Complex state management for concurrent users

### 3. File Upload Endpoints (NEW)

**Action**: **ADD** logo upload handling

```python
@api_view(['POST'])
def upload_logo(request, logo_type, user_uuid=None):
    """
    Handle logo uploads for floatie and launcher
    logo_type: 'floatie' or 'launcher'
    """
    if logo_type not in ['floatie', 'launcher']:
        return Response({'error': 'Invalid logo type'}, status=status.HTTP_400_BAD_REQUEST)
    
    if 'file' not in request.FILES:
        return Response({'error': 'No file provided'}, status=status.HTTP_400_BAD_REQUEST)
    
    file = request.FILES['file']
    
    # Validate file type and size
    if not validate_logo_file(file):
        return Response({'error': 'Invalid file format'}, status=status.HTTP_400_BAD_REQUEST)
    
    # Save file and return URLs
    filename = save_logo_file(file, logo_type, user_uuid)
    preview_url = generate_preview_url(filename)
    
    return Response({
        'filename': filename,
        'preview': preview_url,
        'status': 'uploaded'
    })

def validate_logo_file(file):
    """Validate uploaded logo file"""
    allowed_types = ['image/png', 'image/jpeg', 'image/svg+xml']
    max_size = 5 * 1024 * 1024  # 5MB
    
    return (
        file.content_type in allowed_types and
        file.size <= max_size
    )
```

**Benefits**:
- Immediate file upload feedback
- File validation and security checks
- Preview URL generation for instant UI updates

**Risks**:
- File storage management complexity
- Security validation for uploaded files

## Domain Analysis Integration

### 1. Website Analysis Trigger (MODIFY)

**Action**: **MODIFY** existing crawling system to support onboarding

```python
# helpshelf/apps/crawler/tasks.py

from celery import shared_task
from ..manage.models import Site, WebsiteAnalysisProgress

@shared_task
def trigger_website_analysis(site_id):
    """
    Enhanced website analysis for onboarding process
    Updates progress in real-time for frontend polling
    """
    try:
        site = Site.objects.get(id=site_id)
        progress = site.analysis_progress
        
        # Step 1: Domain Validation
        progress.is_analyzing = True
        progress.started_at = timezone.now()
        update_progress_step(progress, 0, "in_progress", 25)
        
        domain_valid = validate_domain_access(site.domain)
        if not domain_valid:
            update_progress_step(progress, 0, "failed", 100)
            progress.error_message = "Domain not accessible"
            progress.is_analyzing = False
            progress.save()
            return
        
        update_progress_step(progress, 0, "completed", 100)
        
        # Step 2: Content Crawling
        update_progress_step(progress, 1, "in_progress", 0)
        crawl_results = enhanced_crawl_website(site)
        update_progress_step(progress, 1, "completed", 100)
        
        # Step 3: AI Processing
        update_progress_step(progress, 2, "in_progress", 0)
        ai_results = process_with_ai(crawl_results)
        update_progress_step(progress, 2, "completed", 100)
        
        # Step 4: Integration Setup
        update_progress_step(progress, 3, "in_progress", 0)
        setup_integrations(site)
        update_progress_step(progress, 3, "completed", 100)
        
        # Mark as complete
        progress.is_complete = True
        progress.is_analyzing = False
        progress.completed_at = timezone.now()
        progress.completion_percentage = 100.0
        progress.save()
        
        # Trigger onboarding completion event
        track_onboarding_event(site, 'analysis_completed')
        
    except Exception as e:
        handle_analysis_error(site_id, str(e))

def update_progress_step(progress, step_index, status, step_progress):
    """Update specific progress step and recalculate overall progress"""
    if step_index < len(progress.steps):
        progress.steps[step_index]['status'] = status
        progress.steps[step_index]['progress'] = step_progress
        
        # Recalculate overall progress
        total_progress = sum(step.get('progress', 0) for step in progress.steps)
        progress.completion_percentage = total_progress / len(progress.steps)
        progress.current_step = step_index + 1
        progress.save()

def enhanced_crawl_website(site):
    """Enhanced crawling with onboarding-specific features"""
    # Integrate with existing crawling logic
    from .services.crawler import crawl_site
    
    # Add onboarding-specific crawling features
    results = crawl_site(site.domain)
    
    # Extract onboarding-relevant information
    results['onboarding_metadata'] = extract_onboarding_metadata(results)
    
    return results

def extract_onboarding_metadata(crawl_results):
    """Extract metadata useful for onboarding configuration"""
    return {
        'suggested_colors': extract_brand_colors(crawl_results),
        'existing_support_channels': detect_support_channels(crawl_results),
        'content_categories': categorize_content(crawl_results)
    }
```

**Benefits**:
- Real-time progress updates improve user experience
- Enhanced crawling provides onboarding-specific insights
- Integration with existing crawling infrastructure

**Risks**:
- Increased complexity in crawling process
- Potential performance impact from real-time updates
- Error handling complexity

## Real-time Progress System

### 1. Progress Update Service (NEW)

**Action**: **ADD** service for managing progress updates

```python
# helpshelf/apps/api/services/progress_service.py (NEW FILE)

class OnboardingProgressService:
    """Service for managing onboarding progress updates"""
    
    @staticmethod
    def initialize_progress(site):
        """Initialize progress tracking for a site"""
        progress, created = WebsiteAnalysisProgress.objects.get_or_create(
            site=site,
            defaults={
                'steps': get_default_progress_steps(),
                'total_steps': 4,
                'is_analyzing': False
            }
        )
        return progress
    
    @staticmethod
    def start_analysis(site):
        """Start website analysis process"""
        progress = OnboardingProgressService.initialize_progress(site)
        
        if not progress.is_analyzing:
            progress.is_analyzing = True
            progress.started_at = timezone.now()
            progress.save()
            
            # Trigger async analysis
            trigger_website_analysis.delay(site.id)
        
        return progress
    
    @staticmethod
    def get_progress_for_polling(site):
        """Get current progress for frontend polling"""
        progress = getattr(site, 'analysis_progress', None)
        
        if not progress:
            progress = OnboardingProgressService.initialize_progress(site)
        
        return {
            'steps': progress.steps,
            'percent': progress.completion_percentage,
            'currentStep': progress.current_step,
            'isComplete': progress.is_complete,
            'isAnalyzing': progress.is_analyzing,
            'error': progress.error_message,
            'estimatedTimeRemaining': calculate_eta(progress)
        }
    
    @staticmethod
    def handle_analysis_completion(site):
        """Handle completion of website analysis"""
        progress = site.analysis_progress
        
        # Update onboarding configuration
        config = getattr(site, 'onboarding_config', None)
        if config:
            config.is_completed = True
            config.save()
        
        # Track analytics event
        track_onboarding_event(site, 'website_analysis_completed')
        
        return progress

def calculate_eta(progress):
    """Calculate estimated time remaining for analysis"""
    if not progress.started_at or progress.is_complete:
        return 0
    
    elapsed = (timezone.now() - progress.started_at).total_seconds()
    if progress.completion_percentage > 0:
        total_estimated = elapsed / (progress.completion_percentage / 100)
        remaining = total_estimated - elapsed
        return max(0, int(remaining))
    
    return 120  # Default 2 minutes
```

**Benefits**:
- Centralized progress management
- ETA calculation improves user experience
- Clean separation of concerns

**Risks**:
- Additional service layer complexity
- ETA calculation accuracy depends on consistent analysis timing

## Guest User Enhancement

### 1. Session Management Enhancement (MODIFY)

**Action**: **MODIFY** existing guest user session handling

```python
# helpshelf/apps/profile/services/guest_user_service.py

class GuestUserService:
    """Enhanced guest user service for onboarding"""
    
    @staticmethod
    def create_or_update_onboarding_session(uuid, domain=None, step=None, config=None):
        """Create or update guest user onboarding session"""
        guest_user, created = GuestUserProfile.objects.get_or_create(
            uuid=uuid,
            defaults={'created_at': timezone.now()}
        )
        
        # Update onboarding tracking
        if domain:
            guest_user.onboarding_domain = domain
        if step is not None:
            guest_user.onboarding_step = step
        if config:
            guest_user.temp_configuration.update(config)
        
        guest_user.last_activity = timezone.now()
        guest_user.save()
        
        return guest_user
    
    @staticmethod
    def complete_onboarding(uuid):
        """Mark onboarding as completed for guest user"""
        try:
            guest_user = GuestUserProfile.objects.get(uuid=uuid)
            guest_user.onboarding_completed_at = timezone.now()
            guest_user.save()
            
            # Create or update site with final configuration
            if guest_user.onboarding_domain:
                site = Site.objects.get_or_create(domain=guest_user.onboarding_domain)[0]
                OnboardingService.apply_guest_configuration(site, guest_user)
            
            return guest_user
        except GuestUserProfile.DoesNotExist:
            return None
    
    @staticmethod
    def cleanup_expired_sessions():
        """Clean up expired guest user sessions"""
        cutoff = timezone.now() - timedelta(days=7)
        
        expired_sessions = GuestUserProfile.objects.filter(
            last_activity__lt=cutoff,
            onboarding_completed_at__isnull=True
        )
        
        count = expired_sessions.count()
        expired_sessions.delete()
        
        return count

class OnboardingService:
    """Service for managing onboarding completion"""
    
    @staticmethod
    def apply_guest_configuration(site, guest_user):
        """Apply guest user configuration to site"""
        config, created = OnboardingConfiguration.objects.get_or_create(site=site)
        
        temp_config = guest_user.temp_configuration
        
        # Apply design configuration
        design = temp_config.get('design', {})
        if 'primaryColor' in design:
            config.primary_color = design['primaryColor']
        if 'secondaryColor' in design:
            config.secondary_color = design['secondaryColor']
        if 'floatiePosition' in design:
            config.floatie_position = design['floatiePosition']
        
        # Apply contact configuration
        if 'contact' in temp_config:
            config.contact_methods.update(temp_config['contact'])
        
        config.onboarding_step = guest_user.onboarding_step
        config.is_completed = True
        config.save()
        
        return config
```

**Benefits**:
- Seamless transition from guest to authenticated user
- Configuration preservation across sessions
- Automatic cleanup of expired sessions

**Risks**:
- Data migration complexity
- Session management overhead

## Analytics Integration

### 1. Onboarding Event Tracking (NEW)

**Action**: **ADD** analytics tracking for onboarding events

```python
# helpshelf/apps/stats/onboarding_events.py (NEW FILE)

from .models import Event
from .services.event_service import EventService

class OnboardingEventTracker:
    """Track onboarding-specific analytics events"""
    
    @staticmethod
    def track_step_navigation(guest_uuid, from_step, to_step, site_domain=None):
        """Track navigation between onboarding steps"""
        EventService.create_event(
            event_type='onboarding_step_navigation',
            metadata={
                'guest_uuid': guest_uuid,
                'from_step': from_step,
                'to_step': to_step,
                'domain': site_domain,
                'timestamp': timezone.now().isoformat()
            }
        )
    
    @staticmethod
    def track_design_customization(guest_uuid, design_changes, site_domain=None):
        """Track design customization events"""
        EventService.create_event(
            event_type='onboarding_design_update',
            metadata={
                'guest_uuid': guest_uuid,
                'changes': design_changes,
                'domain': site_domain,
                'timestamp': timezone.now().isoformat()
            }
        )
    
    @staticmethod
    def track_onboarding_completion(guest_uuid, site_domain, completion_time):
        """Track successful onboarding completion"""
        EventService.create_event(
            event_type='onboarding_completed',
            metadata={
                'guest_uuid': guest_uuid,
                'domain': site_domain,
                'completion_time_seconds': completion_time,
                'timestamp': timezone.now().isoformat()
            }
        )
    
    @staticmethod
    def track_onboarding_abandonment(guest_uuid, last_step, site_domain=None):
        """Track onboarding abandonment events"""
        EventService.create_event(
            event_type='onboarding_abandoned',
            metadata={
                'guest_uuid': guest_uuid,
                'last_step': last_step,
                'domain': site_domain,
                'timestamp': timezone.now().isoformat()
            }
        )

# Integration with existing event tracking
def track_onboarding_event(site_or_guest, event_type, **kwargs):
    """Unified onboarding event tracking"""
    if event_type == 'step_completed':
        OnboardingEventTracker.track_step_navigation(
            kwargs.get('guest_uuid'),
            kwargs.get('from_step'),
            kwargs.get('to_step'),
            kwargs.get('domain')
        )
    elif event_type == 'design_updated':
        OnboardingEventTracker.track_design_customization(
            kwargs.get('guest_uuid'),
            kwargs.get('changes'),
            kwargs.get('domain')
        )
    elif event_type == 'onboarding_completed':
        OnboardingEventTracker.track_onboarding_completion(
            kwargs.get('guest_uuid'),
            kwargs.get('domain'),
            kwargs.get('completion_time')
        )
```

**Benefits**:
- Comprehensive onboarding analytics
- Integration with existing event system
- Data-driven onboarding optimization

**Risks**:
- Additional database load from event tracking
- Privacy considerations for guest user tracking

## Security Considerations

### 1. API Security Enhancements (MODIFY)

**Action**: **MODIFY** existing API security for onboarding endpoints

```python
# helpshelf/apps/api/middleware.py

class OnboardingSecurityMiddleware:
    """Security middleware for onboarding endpoints"""
    
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        # Apply security checks for onboarding endpoints
        if request.path.startswith('/api/onboarding/'):
            self.validate_onboarding_request(request)
        
        response = self.get_response(request)
        return response
    
    def validate_onboarding_request(self, request):
        """Validate onboarding API requests"""
        # Rate limiting for onboarding endpoints
        if not self.check_rate_limit(request):
            raise PermissionDenied("Rate limit exceeded")
        
        # Validate guest UUID format
        uuid_param = request.GET.get('user_uuid') or request.POST.get('user_uuid')
        if uuid_param and not self.validate_uuid_format(uuid_param):
            raise ValidationError("Invalid UUID format")
        
        # CORS validation for iframe embedding
        if not self.validate_cors_origin(request):
            raise PermissionDenied("Invalid origin")
    
    def check_rate_limit(self, request):
        """Rate limiting for onboarding endpoints"""
        # Implement rate limiting (5 requests per second per IP)
        cache_key = f"onboarding_rate_limit:{self.get_client_ip(request)}"
        current_requests = cache.get(cache_key, 0)
        
        if current_requests >= 5:
            return False
        
        cache.set(cache_key, current_requests + 1, 1)  # 1 second window
        return True
    
    def validate_cors_origin(self, request):
        """Validate CORS origin for iframe embedding"""
        origin = request.META.get('HTTP_ORIGIN')
        if not origin:
            return True  # Allow requests without origin header
        
        # Check against allowed origins or site domains
        allowed_origins = self.get_allowed_origins()
        return origin in allowed_origins
    
    def get_allowed_origins(self):
        """Get allowed origins for CORS"""
        # Include all site domains plus development origins
        site_domains = Site.objects.values_list('domain', flat=True)
        dev_origins = ['http://localhost:3000', 'http://localhost:3001']
        
        return list(site_domains) + dev_origins
```

**Benefits**:
- Rate limiting prevents abuse
- CORS validation secures iframe embedding
- UUID validation prevents injection attacks

**Risks**:
- Additional middleware complexity
- Potential blocking of legitimate requests

### 2. File Upload Security (NEW)

**Action**: **ADD** secure file upload handling

```python
# helpshelf/apps/api/services/file_security.py (NEW FILE)

import magic
from PIL import Image
from django.core.files.storage import default_storage

class FileSecurityService:
    """Secure file upload handling for onboarding"""
    
    ALLOWED_MIME_TYPES = [
        'image/png',
        'image/jpeg', 
        'image/svg+xml'
    ]
    
    MAX_FILE_SIZE = 5 * 1024 * 1024  # 5MB
    MAX_IMAGE_DIMENSIONS = (2048, 2048)
    
    @classmethod
    def validate_upload(cls, file):
        """Comprehensive file validation"""
        # Size check
        if file.size > cls.MAX_FILE_SIZE:
            raise ValidationError("File too large")
        
        # MIME type check
        file_type = magic.from_buffer(file.read(1024), mime=True)
        file.seek(0)  # Reset file pointer
        
        if file_type not in cls.ALLOWED_MIME_TYPES:
            raise ValidationError("Invalid file type")
        
        # Image dimension check
        if file_type.startswith('image/') and file_type != 'image/svg+xml':
            cls.validate_image_dimensions(file)
        
        return True
    
    @classmethod
    def validate_image_dimensions(cls, file):
        """Validate image dimensions"""
        try:
            with Image.open(file) as img:
                if img.size[0] > cls.MAX_IMAGE_DIMENSIONS[0] or img.size[1] > cls.MAX_IMAGE_DIMENSIONS[1]:
                    raise ValidationError("Image dimensions too large")
        except Exception:
            raise ValidationError("Invalid image file")
        finally:
            file.seek(0)
    
    @classmethod
    def sanitize_filename(cls, filename):
        """Sanitize uploaded filename"""
        import re
        import uuid
        
        # Remove path components
        filename = os.path.basename(filename)
        
        # Remove dangerous characters
        filename = re.sub(r'[^a-zA-Z0-9._-]', '', filename)
        
        # Add UUID prefix to prevent conflicts
        name, ext = os.path.splitext(filename)
        return f"{uuid.uuid4().hex[:8]}_{name}{ext}"
```

**Benefits**:
- Comprehensive file validation
- Protection against malicious uploads
- Safe filename handling

**Risks**:
- Additional processing overhead
- Potential false positives in validation

## Testing Strategy

### 1. API Endpoint Tests (NEW)

**Action**: **ADD** comprehensive test suite

```python
# tests/test_onboarding_api.py (NEW FILE)

from django.test import TestCase, Client
from django.urls import reverse
from rest_framework import status
import json
import uuid

class OnboardingAPITests(TestCase):
    """Test suite for onboarding API endpoints"""
    
    def setUp(self):
        self.client = Client()
        self.guest_uuid = str(uuid.uuid4())
        self.test_domain = "example.com"
    
    def test_onboarding_config_get_default(self):
        """Test GET /api/onboarding/config returns default configuration"""
        response = self.client.get(f'/api/onboarding/config/')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        data = response.json()
        
        self.assertIn('domain', data)
        self.assertIn('currentStep', data)
        self.assertIn('configuration', data)
        self.assertEqual(data['currentStep'], 1)
        self.assertFalse(data['isCompleted'])
    
    def test_onboarding_config_post_update(self):
        """Test POST /api/onboarding/config updates configuration"""
        config_data = {
            'domain': self.test_domain,
            'currentStep': 2,
            'configuration': {
                'design': {
                    'primaryColor': '#FF0000',
                    'secondaryColor': '#00FF00'
                }
            }
        }
        
        response = self.client.post(
            f'/api/onboarding/config/?user_uuid={self.guest_uuid}',
            data=json.dumps(config_data),
            content_type='application/json'
        )
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        
        # Verify guest user was updated
        guest_user = GuestUserProfile.objects.get(uuid=self.guest_uuid)
        self.assertEqual(guest_user.onboarding_domain, self.test_domain)
        self.assertEqual(guest_user.onboarding_step, 2)
    
    def test_progress_steps_initialization(self):
        """Test GET /api/onboarding/progress-steps initializes progress"""
        # Create guest user with domain
        guest_user = GuestUserProfile.objects.create(
            uuid=self.guest_uuid,
            onboarding_domain=self.test_domain
        )
        
        response = self.client.get(f'/api/onboarding/progress-steps/?user_uuid={self.guest_uuid}')
        
        self.assertEqual(response.status_code, status.HTTP_200_OK)
        data = response.json()
        
        self.assertIn('steps', data)
        self.assertIn('percent', data)
        self.assertIn('isComplete', data)
        self.assertEqual(len(data['steps']), 4)
    
    def test_logo_upload_validation(self):
        """Test logo upload validation"""
        # Test with invalid file type
        with open('test_file.txt', 'w') as f:
            f.write('test content')
        
        with open('test_file.txt', 'rb') as f:
            response = self.client.post(
                f'/api/onboarding/upload-logo/floatie/',
                {'file': f}
            )
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
        self.assertIn('error', response.json())

class OnboardingProgressTests(TestCase):
    """Test suite for onboarding progress tracking"""
    
    def test_progress_service_initialization(self):
        """Test progress service initialization"""
        site = Site.objects.create(domain="test.com")
        
        progress = OnboardingProgressService.initialize_progress(site)
        
        self.assertIsNotNone(progress)
        self.assertEqual(len(progress.steps), 4)
        self.assertFalse(progress.is_analyzing)
    
    def test_analysis_trigger(self):
        """Test website analysis trigger"""
        site = Site.objects.create(domain="test.com")
        
        progress = OnboardingProgressService.start_analysis(site)
        
        self.assertTrue(progress.is_analyzing)
        self.assertIsNotNone(progress.started_at)

class OnboardingSecurityTests(TestCase):
    """Test suite for onboarding security features"""
    
    def test_rate_limiting(self):
        """Test rate limiting for onboarding endpoints"""
        # Make 6 rapid requests (exceeds limit of 5)
        for i in range(6):
            response = self.client.get('/api/onboarding/config/')
            if i < 5:
                self.assertEqual(response.status_code, status.HTTP_200_OK)
            else:
                self.assertEqual(response.status_code, status.HTTP_429_TOO_MANY_REQUESTS)
    
    def test_uuid_validation(self):
        """Test UUID format validation"""
        invalid_uuid = "not-a-uuid"
        
        response = self.client.get(f'/api/onboarding/config/?user_uuid={invalid_uuid}')
        
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
```

**Benefits**:
- Comprehensive test coverage
- Automated validation of API behavior
- Security testing for rate limiting and validation

**Risks**:
- Test maintenance overhead
- False positives in security tests

### 2. Integration Tests (NEW)

**Action**: **ADD** end-to-end integration tests

```python
# tests/test_onboarding_integration.py (NEW FILE)

from django.test import TransactionTestCase
from django.test.utils import override_settings
from unittest.mock import patch, Mock
import time

class OnboardingIntegrationTests(TransactionTestCase):
    """End-to-end integration tests for onboarding flow"""
    
    @patch('helpshelf.apps.crawler.tasks.trigger_website_analysis.delay')
    def test_complete_onboarding_flow(self, mock_analysis):
        """Test complete onboarding flow from start to finish"""
        guest_uuid = str(uuid.uuid4())
        test_domain = "integration-test.com"
        
        # Step 1: Initialize onboarding
        response = self.client.get('/api/onboarding/config/')
        self.assertEqual(response.status_code, 200)
        
        # Step 2: Set domain
        config_data = {
            'domain': test_domain,
            'currentStep': 1
        }
        response = self.client.post(
            f'/api/onboarding/config/?user_uuid={guest_uuid}',
            data=json.dumps(config_data),
            content_type='application/json'
        )
        self.assertEqual(response.status_code, 200)
        
        # Step 3: Update design configuration
        design_data = {
            'domain': test_domain,
            'currentStep': 1,
            'configuration': {
                'design': {
                    'primaryColor': '#FF0000',
                    'floatiePosition': 'left'
                }
            }
        }
        response = self.client.post(
            f'/api/onboarding/config/?user_uuid={guest_uuid}',
            data=json.dumps(design_data),
            content_type='application/json'
        )
        self.assertEqual(response.status_code, 200)
        
        # Step 4: Progress to content step
        response = self.client.post(
            f'/api/onboarding/config/?user_uuid={guest_uuid}',
            data=json.dumps({'domain': test_domain, 'currentStep': 2}),
            content_type='application/json'
        )
        self.assertEqual(response.status_code, 200)
        
        # Step 5: Check progress polling
        response = self.client.get(f'/api/onboarding/progress-steps/?user_uuid={guest_uuid}')
        self.assertEqual(response.status_code, 200)
        
        # Verify analysis was triggered
        mock_analysis.assert_called_once()
        
        # Step 6: Complete onboarding
        guest_user = GuestUserService.complete_onboarding(guest_uuid)
        self.assertIsNotNone(guest_user.onboarding_completed_at)
        
        # Verify site configuration was created
        site = Site.objects.get(domain=test_domain)
        config = site.onboarding_config
        self.assertEqual(config.primary_color, '#FF0000')
        self.assertEqual(config.floatie_position, 'left')
```

**Benefits**:
- End-to-end validation of onboarding flow
- Integration testing between components
- Real-world scenario testing

**Risks**:
- Complex test setup and teardown
- Potential test flakiness with async operations

## Deployment Plan

### 1. Database Migration Strategy

**Action**: **ADD** database migrations with rollback plan

```python
# migrations/0001_add_onboarding_models.py

from django.db import migrations, models

class Migration(migrations.Migration):
    
    dependencies = [
        ('manage', '0001_initial'),
        ('profile', '0001_initial'),
    ]
    
    operations = [
        # Add OnboardingConfiguration model
        migrations.CreateModel(
            name='OnboardingConfiguration',
            fields=[
                ('id', models.AutoField(primary_key=True)),
                ('site', models.OneToOneField('manage.Site', on_delete=models.CASCADE)),
                ('primary_color', models.CharField(max_length=7, default='#FB8537')),
                ('secondary_color', models.CharField(max_length=7, default='#4884FA')),
                ('floatie_position', models.CharField(max_length=10, default='right')),
                ('floatie_logo', models.FileField(upload_to='onboarding/logos/', null=True, blank=True)),
                ('launcher_logo', models.FileField(upload_to='onboarding/launchers/', null=True, blank=True)),
                ('onboarding_step', models.IntegerField(default=1)),
                ('is_completed', models.BooleanField(default=False)),
                ('contact_methods', models.JSONField(default=dict)),
                ('disabled_contact_tabs', models.JSONField(default=dict)),
                ('created_at', models.DateTimeField(auto_now_add=True)),
                ('updated_at', models.DateTimeField(auto_now=True)),
            ],
            options={'db_table': 'onboarding_configuration'},
        ),
        
        # Add WebsiteAnalysisProgress model
        migrations.CreateModel(
            name='WebsiteAnalysisProgress',
            fields=[
                ('id', models.AutoField(primary_key=True)),
                ('site', models.OneToOneField('manage.Site', on_delete=models.CASCADE)),
                ('steps', models.JSONField(default=list)),
                ('current_step', models.IntegerField(default=0)),
                ('total_steps', models.IntegerField(default=0)),
                ('completion_percentage', models.FloatField(default=0.0)),
                ('is_analyzing', models.BooleanField(default=False)),
                ('is_complete', models.BooleanField(default=False)),
                ('error_message', models.TextField(null=True, blank=True)),
                ('retry_count', models.IntegerField(default=0)),
                ('started_at', models.DateTimeField(null=True, blank=True)),
                ('completed_at', models.DateTimeField(null=True, blank=True)),
                ('updated_at', models.DateTimeField(auto_now=True)),
            ],
            options={'db_table': 'website_analysis_progress'},
        ),
        
        # Add fields to GuestUserProfile
        migrations.AddField(
            model_name='GuestUserProfile',
            name='onboarding_domain',
            field=models.CharField(max_length=255, null=True, blank=True),
        ),
        migrations.AddField(
            model_name='GuestUserProfile',
            name='onboarding_step',
            field=models.IntegerField(default=0),
        ),
        migrations.AddField(
            model_name='GuestUserProfile',
            name='onboarding_completed_at',
            field=models.DateTimeField(null=True, blank=True),
        ),
        migrations.AddField(
            model_name='GuestUserProfile',
            name='temp_configuration',
            field=models.JSONField(default=dict, blank=True),
        ),
    ]
```

### 2. Feature Flag Implementation

**Action**: **ADD** feature flags for gradual rollout

```python
# helpshelf/apps/core/feature_flags.py (NEW FILE)

from django.conf import settings
from django.core.cache import cache

class FeatureFlags:
    """Feature flag management for onboarding rollout"""
    
    ONBOARDING_V2_ENABLED = 'onboarding_v2_enabled'
    ONBOARDING_PROGRESS_POLLING = 'onboarding_progress_polling'
    ONBOARDING_ANALYTICS = 'onboarding_analytics'
    
    @classmethod
    def is_enabled(cls, flag_name, site=None, user=None):
        """Check if feature flag is enabled"""
        # Check cache first
        cache_key = f"feature_flag:{flag_name}"
        if site:
            cache_key += f":site:{site.id}"
        
        cached_value = cache.get(cache_key)
        if cached_value is not None:
            return cached_value
        
        # Check database or settings
        enabled = cls._check_flag_database(flag_name, site, user)
        
        # Cache result for 5 minutes
        cache.set(cache_key, enabled, 300)
        
        return enabled
    
    @classmethod
    def _check_flag_database(cls, flag_name, site, user):
        """Check feature flag in database"""
        if flag_name == cls.ONBOARDING_V2_ENABLED:
            # Enable for beta sites first
            if site and hasattr(site, 'is_beta') and site.is_beta:
                return True
            
            # Enable based on percentage rollout
            return cls._percentage_rollout(flag_name, 25)  # 25% rollout
        
        return getattr(settings, f'FEATURE_{flag_name.upper()}', False)
    
    @classmethod
    def _percentage_rollout(cls, flag_name, percentage):
        """Implement percentage-based rollout"""
        import hashlib
        
        # Use flag name to generate consistent hash
        hash_input = f"{flag_name}:{settings.SECRET_KEY}"
        hash_value = int(hashlib.md5(hash_input.encode()).hexdigest()[:8], 16)
        
        return (hash_value % 100) < percentage

# Usage in views
def onboarding_config(request, user_uuid=None):
    """Onboarding config endpoint with feature flag"""
    if not FeatureFlags.is_enabled(FeatureFlags.ONBOARDING_V2_ENABLED):
        return Response({'error': 'Feature not available'}, status=status.HTTP_404_NOT_FOUND)
    
    # Continue with onboarding logic...
```

### 3. Monitoring and Alerts

**Action**: **ADD** monitoring for onboarding performance

```python
# helpshelf/apps/core/monitoring.py (NEW FILE)

import logging
from django.core.cache import cache
from django.utils import timezone
from datetime import timedelta

logger = logging.getLogger('onboarding')

class OnboardingMonitoring:
    """Monitoring and alerting for onboarding system"""
    
    @classmethod
    def track_api_performance(cls, endpoint, response_time, status_code):
        """Track API endpoint performance"""
        # Log slow requests
        if response_time > 1.0:  # 1 second threshold
            logger.warning(f"Slow onboarding API request: {endpoint} took {response_time:.2f}s")
        
        # Track error rates
        if status_code >= 400:
            cls._increment_error_counter(endpoint)
        
        # Update performance metrics
        cls._update_performance_metrics(endpoint, response_time)
    
    @classmethod
    def track_onboarding_completion_rate(cls):
        """Monitor onboarding completion rates"""
        now = timezone.now()
        day_ago = now - timedelta(days=1)
        
        # Count started vs completed onboardings
        started_count = GuestUserProfile.objects.filter(
            onboarding_step__gt=0,
            created_at__gte=day_ago
        ).count()
        
        completed_count = GuestUserProfile.objects.filter(
            onboarding_completed_at__gte=day_ago
        ).count()
        
        completion_rate = (completed_count / started_count * 100) if started_count > 0 else 0
        
        # Alert if completion rate drops below threshold
        if completion_rate < 50:  # 50% threshold
            logger.error(f"Low onboarding completion rate: {completion_rate:.1f}%")
            cls._send_alert('onboarding_completion_rate_low', {
                'completion_rate': completion_rate,
                'started_count': started_count,
                'completed_count': completed_count
            })
        
        return completion_rate
    
    @classmethod
    def monitor_analysis_performance(cls):
        """Monitor website analysis performance"""
        # Check for stuck analysis processes
        stuck_threshold = timezone.now() - timedelta(minutes=10)
        
        stuck_analyses = WebsiteAnalysisProgress.objects.filter(
            is_analyzing=True,
            started_at__lt=stuck_threshold
        )
        
        if stuck_analyses.exists():
            logger.error(f"Found {stuck_analyses.count()} stuck website analyses")
            
            for analysis in stuck_analyses:
                # Reset stuck analysis
                analysis.is_analyzing = False
                analysis.error_message = "Analysis timed out and was reset"
                analysis.save()
    
    @classmethod
    def _send_alert(cls, alert_type, data):
        """Send alert notification"""
        # Implement alerting system (email, Slack, etc.)
        logger.critical(f"ALERT: {alert_type} - {data}")
```

### 4. Rollback Plan

**Action**: **DEFINE** rollback procedures

```python
# rollback_procedures.md

## Onboarding Integration Rollback Plan

### 1. Feature Flag Rollback (IMMEDIATE)
- Set `ONBOARDING_V2_ENABLED` to `False` in feature flags
- This immediately disables new onboarding endpoints
- Frontend reverts to existing Django onboarding flow

### 2. Database Rollback (IF NEEDED)
- Run migration rollback: `python manage.py migrate onboarding 0000_initial`
- This removes new tables but preserves existing data
- Guest user fields are rolled back to previous schema

### 3. Code Rollback (IF NEEDED)
- Revert to previous git commit
- Remove new API endpoints
- Disable onboarding middleware

### 4. Data Preservation
- Export onboarding configurations before rollback
- Backup guest user onboarding data
- Maintain audit trail of changes

### 5. Communication Plan
- Notify engineering team of rollback
- Update monitoring dashboards
- Document rollback reasons for future reference
```

**Benefits**:
- Quick rollback capability via feature flags
- Data preservation during rollback
- Clear rollback procedures

**Risks**:
- Potential data loss if not properly planned
- User experience disruption during rollback

## Implementation Timeline

### Phase 1: Foundation (Week 1-2)
1. **Database Schema Changes** - Add new models and migrations
2. **Basic API Endpoints** - Implement core configuration and progress endpoints
3. **Guest User Enhancement** - Extend guest user model for onboarding tracking
4. **Feature Flags** - Implement feature flag system for gradual rollout

### Phase 2: Core Functionality (Week 3-4)
1. **Website Analysis Integration** - Integrate with existing crawling system
2. **File Upload System** - Implement secure logo upload handling
3. **Progress Tracking** - Real-time progress monitoring system
4. **Basic Security** - Rate limiting and validation

### Phase 3: Advanced Features (Week 5-6)
1. **Analytics Integration** - Comprehensive event tracking
2. **Error Handling** - Robust error handling and recovery
3. **Performance Optimization** - Caching and query optimization
4. **Enhanced Security** - CORS validation, file security

### Phase 4: Testing & Deployment (Week 7-8)
1. **Comprehensive Testing** - Unit, integration, and security tests
2. **Monitoring Setup** - Performance monitoring and alerting
3. **Beta Deployment** - Limited rollout with feature flags
4. **Full Deployment** - Complete rollout with monitoring

## Risk Assessment

### High Risk Items
1. **Database Migrations** - Risk of downtime during migration
   - **Mitigation**: Use migration with minimal downtime strategies
   - **Rollback**: Feature flag disable + migration rollback

2. **Performance Impact** - Real-time polling may impact server performance
   - **Mitigation**: Implement caching and rate limiting
   - **Monitoring**: Track API response times and server load

3. **Data Consistency** - Complex state synchronization between guest users and sites
   - **Mitigation**: Comprehensive testing and validation
   - **Rollback**: Data export before changes

### Medium Risk Items
1. **Security Vulnerabilities** - New file upload and API endpoints
   - **Mitigation**: Comprehensive security testing and validation
   - **Monitoring**: Security scanning and penetration testing

2. **Integration Complexity** - Integration with existing crawling system
   - **Mitigation**: Gradual integration with feature flags
   - **Testing**: Comprehensive integration testing

### Low Risk Items
1. **Frontend Integration** - Well-defined API contract with frontend
   - **Mitigation**: Clear API documentation and testing
   - **Benefits**: Clean separation of concerns

2. **Analytics Enhancement** - Extension of existing event system
   - **Mitigation**: Use existing event infrastructure
   - **Benefits**: Improved onboarding insights

## Success Metrics

### Technical Metrics
- **API Response Time**: < 200ms for 95% of requests
- **Analysis Completion Rate**: > 95% success rate
- **Error Rate**: < 1% of API requests
- **Uptime**: > 99.9% availability

### Business Metrics
- **Onboarding Completion Rate**: > 70% completion
- **Time to Complete**: < 10 minutes average
- **User Satisfaction**: > 4.5/5 rating
- **Support Ticket Reduction**: 30% reduction in onboarding-related tickets

### Performance Metrics
- **Database Performance**: No degradation in existing queries
- **Server Load**: < 10% increase in average CPU usage
- **Memory Usage**: < 15% increase in memory consumption
- **Cache Hit Rate**: > 90% for onboarding data