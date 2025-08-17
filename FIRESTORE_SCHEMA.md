# Firestore Schema Documentation

## Overview
This document outlines the Firestore collections and security rules for the PlatinumData application, covering data management for leads, RFPs, enrichment logs, and compliance events.

## Collections Structure

### 1. Leads Collection (`/leads/{leadId}`)

**Purpose**: Store prospective client information and lead tracking data

**Document Structure**:
```javascript
{
  id: string,                    // Auto-generated document ID
  companyName: string,           // Company name
  contactName: string,           // Primary contact name
  email: string,                 // Contact email
  phone: string,                 // Contact phone number
  industry: string,              // Industry classification
  leadSource: string,            // Source of lead (web, referral, cold outreach, etc.)
  status: string,                // current, qualified, converted, closed
  priority: string,              // high, medium, low
  estimatedValue: number,        // Potential contract value
  notes: string,                 // Additional notes
  assignedTo: string,            // User ID of assigned sales rep
  createdAt: timestamp,          // Creation timestamp
  updatedAt: timestamp,          // Last update timestamp
  tags: array<string>,           // Categorization tags
  customFields: map              // Flexible additional data
}
```

**Subcollections**:
- `/leads/{leadId}/interactions` - Track all interactions with the lead
- `/leads/{leadId}/documents` - Related documents and files

### 2. RFPs Collection (`/rfps/{rfpId}`)

**Purpose**: Manage Request for Proposal documents and responses

**Document Structure**:
```javascript
{
  id: string,                    // Auto-generated document ID
  title: string,                 // RFP title
  clientId: string,              // Reference to client/lead
  description: string,           // RFP description
  requirements: array<object>,   // List of requirements with details
  deadline: timestamp,           // Submission deadline
  budget: object: {              // Budget information
    min: number,
    max: number,
    currency: string
  },
  status: string,                // draft, submitted, under_review, awarded, rejected
  submissionDate: timestamp,     // When response was submitted
  attachments: array<object>,    // File references and metadata
  teamMembers: array<string>,    // User IDs of team members
  priority: string,              // high, medium, low
  winProbability: number,        // Estimated win percentage (0-100)
  createdAt: timestamp,
  updatedAt: timestamp,
  createdBy: string              // User ID who created the RFP
}
```

**Subcollections**:
- `/rfps/{rfpId}/responses` - Track response versions and drafts
- `/rfps/{rfpId}/communications` - Client communications related to RFP

### 3. Enrichment Logs Collection (`/enrichmentLogs/{logId}`)

**Purpose**: Track data enrichment activities and results

**Document Structure**:
```javascript
{
  id: string,                    // Auto-generated document ID
  targetType: string,            // lead, company, contact
  targetId: string,              // ID of the enriched entity
  enrichmentType: string,        // email_finder, company_info, social_media, etc.
  dataSource: string,            // Source of enrichment (API name, service)
  status: string,                // pending, completed, failed, partial
  inputData: map,                // Original data used for enrichment
  outputData: map,               // Enriched data results
  confidence: number,            // Confidence score (0-100)
  cost: number,                  // Cost of enrichment operation
  processingTime: number,        // Time taken in milliseconds
  errorDetails: string,          // Error message if failed
  apiUsage: object: {            // API usage tracking
    provider: string,
    requestsUsed: number,
    remainingQuota: number
  },
  triggeredBy: string,           // User ID or system process
  createdAt: timestamp,
  completedAt: timestamp
}
```

**Indexes Needed**:
- targetId + createdAt
- status + createdAt
- dataSource + createdAt

### 4. Compliance Events Collection (`/complianceEvents/{eventId}`)

**Purpose**: Log compliance-related activities and audit trail

**Document Structure**:
```javascript
{
  id: string,                    // Auto-generated document ID
  eventType: string,             // data_access, data_export, data_deletion, consent_update
  entityType: string,            // lead, rfp, enrichment_log, user
  entityId: string,              // ID of affected entity
  userId: string,                // User who triggered the event
  userEmail: string,             // User email for audit purposes
  action: string,                // create, read, update, delete, export
  dataTypes: array<string>,      // Types of data involved (pii, contact_info, etc.)
  legalBasis: string,            // GDPR legal basis (consent, contract, etc.)
  consentStatus: string,         // granted, withdrawn, pending
  ipAddress: string,             // IP address of the user
  userAgent: string,             // Browser user agent
  geolocation: object: {         // Geographic location
    country: string,
    region: string
  },
  dataSubject: object: {         // Information about data subject
    email: string,
    name: string,
    id: string
  },
  retentionPeriod: number,       // Data retention period in days
  metadata: map,                 // Additional context data
  createdAt: timestamp,
  expiresAt: timestamp           // For automatic cleanup
}
```

**Indexes Needed**:
- userId + createdAt
- entityId + eventType + createdAt
- dataSubject.email + createdAt
- expiresAt (for TTL)

## Security Rules

### General Principles
1. Authentication required for all operations
2. Role-based access control
3. Data ownership and team-based permissions
4. Audit logging for sensitive operations
5. Rate limiting for API operations

### Leads Collection Security Rules
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Leads collection rules
    match /leads/{leadId} {
      // Allow read if user is authenticated and part of the organization
      allow read: if request.auth != null && 
                     (request.auth.token.orgId == resource.data.orgId ||
                      request.auth.uid in resource.data.teamMembers);
      
      // Allow create if user is authenticated and sets correct orgId
      allow create: if request.auth != null &&
                       request.auth.token.orgId == request.resource.data.orgId &&
                       request.resource.data.createdBy == request.auth.uid;
      
      // Allow update if user owns the lead or is assigned to it
      allow update: if request.auth != null &&
                       (request.auth.uid == resource.data.createdBy ||
                        request.auth.uid == resource.data.assignedTo ||
                        hasRole('admin'));
      
      // Only admins can delete leads
      allow delete: if request.auth != null && hasRole('admin');
      
      // Subcollection rules
      match /interactions/{interactionId} {
        allow read, write: if request.auth != null &&
                             (request.auth.uid in get(/databases/$(database)/documents/leads/$(leadId)).data.teamMembers ||
                              hasRole('admin'));
      }
    }
  }
}
```

### RFPs Collection Security Rules
```javascript
match /rfps/{rfpId} {
  // Read access for team members and admins
  allow read: if request.auth != null &&
                 (request.auth.uid in resource.data.teamMembers ||
                  request.auth.token.orgId == resource.data.orgId ||
                  hasRole('admin'));
  
  // Create access for authenticated users
  allow create: if request.auth != null &&
                   request.resource.data.createdBy == request.auth.uid &&
                   request.auth.token.orgId == request.resource.data.orgId;
  
  // Update access for team members and creator
  allow update: if request.auth != null &&
                   (request.auth.uid in resource.data.teamMembers ||
                    request.auth.uid == resource.data.createdBy ||
                    hasRole('admin'));
  
  // Delete restricted to admins and creator
  allow delete: if request.auth != null &&
                   (request.auth.uid == resource.data.createdBy ||
                    hasRole('admin'));
}
```

### Enrichment Logs Security Rules
```javascript
match /enrichmentLogs/{logId} {
  // Read access based on organization and role
  allow read: if request.auth != null &&
                 (request.auth.token.orgId == resource.data.orgId ||
                  hasRole('admin') ||
                  hasRole('data_analyst'));
  
  // Create access for system processes and authorized users
  allow create: if request.auth != null &&
                   (hasRole('admin') ||
                    hasRole('data_analyst') ||
                    request.auth.uid == request.resource.data.triggeredBy);
  
  // Update restricted to system and admins
  allow update: if request.auth != null &&
                   (hasRole('admin') ||
                    request.auth.uid == 'system');
  
  // No deletion allowed except for admins with retention policy
  allow delete: if request.auth != null &&
                   hasRole('admin') &&
                   request.time > resource.data.createdAt + duration.value(365, 'd');
}
```

### Compliance Events Security Rules
```javascript
match /complianceEvents/{eventId} {
  // Read access for compliance officers and admins only
  allow read: if request.auth != null &&
                 (hasRole('compliance_officer') ||
                  hasRole('admin') ||
                  request.auth.uid == resource.data.userId);
  
  // Create access for system and authorized processes
  allow create: if request.auth != null &&
                   request.resource.data.userId == request.auth.uid &&
                   request.resource.data.userEmail == request.auth.token.email;
  
  // No updates allowed to maintain audit integrity
  allow update: if false;
  
  // Automatic deletion based on retention policy
  allow delete: if request.auth != null &&
                   hasRole('admin') &&
                   request.time > resource.data.expiresAt;
}
```

### Utility Functions
```javascript
// Helper function to check user roles
function hasRole(role) {
  return request.auth != null &&
         request.auth.token.roles != null &&
         role in request.auth.token.roles;
}

// Helper function to check organization membership
function sameOrganization() {
  return request.auth.token.orgId == resource.data.orgId;
}

// Helper function to validate required fields
function hasRequiredFields(fields) {
  return fields.all(field, field in request.resource.data);
}
```

## Data Retention and Cleanup

### Automatic Cleanup Rules
1. **Enrichment Logs**: Retain for 2 years, then archive
2. **Compliance Events**: Retain based on `expiresAt` field
3. **Leads**: Mark as inactive after 1 year of no activity
4. **RFPs**: Archive completed RFPs after 3 years

### Backup Strategy
1. Daily automated backups
2. Cross-region replication for disaster recovery
3. Point-in-time recovery capability
4. Monthly backup verification

## Monitoring and Alerting

### Key Metrics to Monitor
1. Read/write operations per collection
2. Security rule violations
3. Failed authentication attempts
4. Data export activities
5. Unusual access patterns

### Alerts Configuration
1. High volume of failed security rule evaluations
2. Unusual data access patterns
3. Failed enrichment operations
4. Compliance event anomalies
5. Storage quota approaching limits

## Migration Considerations

### Initial Data Migration
1. Validate existing data structure
2. Transform data to match new schema
3. Preserve audit trail during migration
4. Implement rollback strategy

### Schema Evolution
1. Backward compatibility maintenance
2. Gradual migration approach
3. Version-aware client applications
4. Data validation during transition

This schema provides a robust foundation for managing leads, RFPs, data enrichment, and compliance in a scalable and secure manner.
