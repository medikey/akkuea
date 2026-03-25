# Notification Service Implementation

## Overview

The Notification Service is a comprehensive system for managing user notifications across the AkkuEA DeFi RWA platform. It handles notifications for KYC verifications, loan repayments, risk alerts, and portfolio updates with multi-channel delivery support (in-app, email, SMS).

## Architecture

### Core Components

1. **Database Schema** (`src/db/schema/notifications.ts`)
   - Notifications table with rich metadata support
   - Event types, delivery channels, and status tracking
   - Retry logic with configurable max retries

2. **Repository Layer** (`src/repositories/NotificationRepository.ts`)
   - Database operations for notifications
   - Query methods for user, event type, and delivery status filtering
   - Delivery state management and cleanup operations

3. **Service Layer** (`src/services/NotificationService.ts`)
   - High-level notification creation and management
   - Delivery state transitions (pending → sent → delivered/failed)
   - Retry scheduling with exponential backoff
   - Domain-specific notification helpers

4. **Controller Layer** (`src/controllers/NotificationController.ts`)
   - REST API endpoints for user notifications
   - Authentication and authorization checks
   - Pagination and filtering support

5. **Routes** (`src/routes/notifications.ts`)
   - REST endpoint definitions
   - Request validation and error handling

6. **Tests** (`src/__tests__/NotificationService.test.ts`)
   - Comprehensive unit tests for the service layer
   - Mock repository tests

## API Endpoints

### User Notifications

```
GET /notifications
  - Get paginated notifications for authenticated user
  - Query params: limit (1-100, default 20), offset (default 0)
  - Response: { data: Notification[], pagination: { limit, offset } }

GET /notifications/unread-count
  - Get count of unread notifications
  - Response: { unreadCount: number }

GET /notifications/:id
  - Get a specific notification by ID
  - Response: Notification object

PATCH /notifications/:id/read
  - Mark notification as read
  - Response: Updated Notification object

POST /notifications/read-multiple
  - Mark multiple notifications as read
  - Body: { notificationIds: string[] }
  - Response: { data: Notification[], message: string }

POST /notifications/read-all
  - Mark all notifications as read for the user
  - Response: { message: string, count: number }

DELETE /notifications/:id
  - Delete a notification
  - Response: { message: string }
```

## Notification Event Types

### Verification Events
- `VERIFICATION_APPROVED` - User's KYC verification approved
- `VERIFICATION_REJECTED` - User's KYC verification rejected

### Financial Events
- `VALUATION_UPDATED` - Property valuation has been updated
- `REPAYMENT_REMINDER` - Upcoming loan repayment
- `REPAYMENT_OVERDUE` - Loan payment is overdue
- `REPAYMENT_PROCESSED` - Loan repayment successfully processed

### Risk Events
- `RISK_WARNING` - Loan health factor warning
- `LIQUIDATION_RISK` - Loan at critical liquidation risk
- `PORTFOLIO_UPDATE` - Portfolio value changes

### System Events
- `SYSTEM_ALERT` - General system alerts
- `INVESTMENT_OPPORTUNITY` - New investment opportunities

## Delivery Channels

- `IN_APP` - In-application notifications
- `EMAIL` - Email delivery
- `SMS` - SMS delivery

## Delivery Status Flow

```
PENDING → SENT → DELIVERED
       ↘ FAILED (with retry scheduling)
              ↘ BOUNCED (if max retries exceeded)
```

## Integration Points

### KYC Service Integration
When KYC verification is completed:
```typescript
// In KYCController.verifyDocument()
if (anyRejected) {
  await notificationService.notifyVerificationRejected(userId, reason);
} else if (allApproved) {
  await notificationService.notifyVerificationApproved(userId);
}
```

### Lending Service Integration
When loan repayment is processed:
```typescript
// In LendingController.repay()
await notificationService.notifyRepaymentProcessed(userId, loanId, amount);
```

### Risk Monitoring Integration
When position health changes:
```typescript
// In RiskMonitoringController.assessAllPositions()
if (health.riskLevel === 'critical') {
  await notificationService.notifyLiquidationRisk(userId, positionId);
} else if (health.riskLevel === 'warning') {
  await notificationService.notifyRiskWarning(userId, positionId, 'warning');
}
```

## Usage Examples

### Creating a Notification
```typescript
const notificationService = new NotificationService();

// Create a single notification
await notificationService.createNotification({
  userId: 'user-123',
  eventType: 'SYSTEM_ALERT',
  title: 'Alert Title',
  message: 'Alert message content',
  channel: 'IN_APP',
  metadata: { customKey: 'customValue' }
});

// Create bulk notifications
await notificationService.createBulkNotifications([
  { userId: 'user-1', eventType: 'SYSTEM_ALERT', ... },
  { userId: 'user-2', eventType: 'SYSTEM_ALERT', ... }
]);
```

### Managing Notification State
```typescript
// Mark as read
await notificationService.markAsRead(notificationId);

// Mark multiple as read
await notificationService.markMultipleAsRead([id1, id2, id3]);

// Mark all as read for a user
await notificationService.markAllAsRead(userId);

// Get unread count
const count = await notificationService.getUnreadCount(userId);
```

### Handling Delivery
```typescript
// Get pending notifications
const pending = await notificationService.getPendingNotifications();

// Mark as sent
await notificationService.markAsSent(notificationId);

// Mark as delivered
await notificationService.markAsDelivered(notificationId);

// Mark as failed and schedule retry
await notificationService.markAsFailed(notificationId, 'Delivery failed');

// Get notifications ready for retry
const readyForRetry = await notificationService.getNotificationsReadyForRetry();
```

## Database Migration

Run the migration to create the notifications table:
```bash
npm run db:migrate
# or manually execute: apps/api/src/db/migrations/002_create_notifications_table.sql
```

## Testing

Run notification service tests:
```bash
npm test -- NotificationService
```

## Configuration

The NotificationService accepts optional configuration:

```typescript
const config: NotificationDeliveryConfig = {
  maxRetries: 3,          // Maximum retry attempts
  retryDelayMs: 60000,    // Initial retry delay in milliseconds
};

const service = new NotificationService(repository, config);
```

Retry scheduling uses exponential backoff:
- Retry 1: 1 minute delay
- Retry 2: 2 minutes delay
- Retry 3: 3 minutes delay

## Security Considerations

1. **Authentication**: All endpoints require `x-user-id` header
2. **Authorization**: Users can only access their own notifications
3. **Input Validation**: All inputs validated with Zod schemas
4. **Rate Limiting**: Suitable for production deployment with rate limiting middleware
5. **Sensitive Data**: Avoid storing sensitive information in notification metadata

## Performance Optimization

1. **Indexes**: Comprehensive indexes on frequently queried columns
2. **Pagination**: Default limit of 20, maximum of 100 notifications per request
3. **Cleanup**: Old notifications can be cleaned up with `cleanupOldNotifications(daysToKeep)`
4. **Batch Operations**: Support for bulk notification creation and marking as read

## Future Enhancements

1. **Delivery Adapters**: Pluggable delivery implementations for email/SMS providers
2. **Notification Preferences**: User preferences for notification types and channels
3. **Notification Templates**: Customizable message templates
4. **Real-time Updates**: WebSocket support for real-time notification delivery
5. **Analytics**: Track notification delivery success rates and user engagement
6. **Scheduled Notifications**: Support for scheduled/delayed notification delivery

## Troubleshooting

### Notifications not appearing
1. Check that notifications are created with correct userId
2. Verify user authentication in API requests
3. Check notification delivery status in database

### Delivery failures
1. Review `failure_reason` field in notification record
2. Check `retry_count` and `nextRetryAt` for retry scheduling
3. Verify email/SMS provider configuration if using external channels

### Performance issues
1. Review database indexes are being used efficiently
2. Consider archiving/deleting old notifications
3. Check pagination parameters in API requests
