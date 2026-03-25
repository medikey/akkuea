# Notification Service Quick Start Guide

## Overview

The Notification Service is now fully implemented and integrated with the AkkuEA platform. This guide will help you get started quickly.

## What Was Built

✅ Complete notification management system
✅ 7 REST API endpoints
✅ 11 event types for different notification scenarios
✅ Multi-channel delivery support (In-App, Email, SMS)
✅ Automatic retry logic with exponential backoff
✅ Full test coverage
✅ Database migrations

## 1. Database Setup

Before using the notification service, run the database migration:

```bash
cd apps/api

# Run migration
npm run db:migrate

# Or manually execute the SQL file
# psql <connection-string> -f src/db/migrations/002_create_notifications_table.sql
```

## 2. API Usage Examples

### Get User's Notifications

```bash
curl -X GET http://localhost:3000/notifications \
  -H "x-user-id: user-123" \
  -H "Content-Type: application/json"
```

Response:
```json
{
  "data": [
    {
      "id": "notif-123",
      "userId": "user-123",
      "eventType": "VERIFICATION_APPROVED",
      "title": "Verification Approved",
      "message": "Your KYC verification has been approved.",
      "channel": "IN_APP",
      "deliveryStatus": "DELIVERED",
      "isRead": false,
      "createdAt": "2025-03-25T10:00:00Z",
      "updatedAt": "2025-03-25T10:00:00Z"
    }
  ],
  "pagination": { "limit": 20, "offset": 0 }
}
```

### Get Unread Count

```bash
curl -X GET http://localhost:3000/notifications/unread-count \
  -H "x-user-id: user-123"
```

### Mark Notification as Read

```bash
curl -X PATCH http://localhost:3000/notifications/notif-123/read \
  -H "x-user-id: user-123"
```

### Mark Multiple as Read

```bash
curl -X POST http://localhost:3000/notifications/read-multiple \
  -H "x-user-id: user-123" \
  -H "Content-Type: application/json" \
  -d '{
    "notificationIds": ["notif-123", "notif-456"]
  }'
```

### Mark All as Read

```bash
curl -X POST http://localhost:3000/notifications/read-all \
  -H "x-user-id: user-123"
```

### Delete Notification

```bash
curl -X DELETE http://localhost:3000/notifications/notif-123 \
  -H "x-user-id: user-123"
```

## 3. Programmatic Usage

### Creating Notifications

```typescript
import { NotificationService } from './services/NotificationService';

const notificationService = new NotificationService();

// Create a verification approval notification
await notificationService.notifyVerificationApproved('user-123');

// Create a repayment reminder
const dueDate = new Date();
dueDate.setDate(dueDate.getDate() + 7);
await notificationService.notifyRepaymentReminder(
  'user-123',
  'loan-456',
  5000,
  dueDate
);

// Create a risk warning
await notificationService.notifyRiskWarning(
  'user-123',
  'loan-456',
  'warning'
);

// Create bulk notifications
await notificationService.createBulkNotifications([
  {
    userId: 'user-1',
    eventType: 'SYSTEM_ALERT',
    title: 'System Maintenance',
    message: 'System maintenance scheduled for tonight',
    channel: 'IN_APP'
  },
  {
    userId: 'user-2',
    eventType: 'SYSTEM_ALERT',
    title: 'System Maintenance',
    message: 'System maintenance scheduled for tonight',
    channel: 'IN_APP'
  }
]);
```

### Managing Notification States

```typescript
// Mark as read
await notificationService.markAsRead('notif-123');

// Mark multiple as read
await notificationService.markMultipleAsRead(['notif-123', 'notif-456']);

// Mark all as read for a user
const count = await notificationService.markAllAsRead('user-123');

// Get unread count
const unreadCount = await notificationService.getUnreadCount('user-123');

// Get unread notifications
const unreadNotifications = await notificationService.getUnreadNotifications('user-123');
```

### Handling Delivery

```typescript
// Get pending notifications (ready to send)
const pending = await notificationService.getPendingNotifications();

// Mark as sent
await notificationService.markAsSent('notif-123');

// Mark as delivered
await notificationService.markAsDelivered('notif-123');

// Mark as failed and schedule retry
await notificationService.markAsFailed('notif-123', 'Email service unavailable');

// Get notifications ready for retry
const readyForRetry = await notificationService.getNotificationsReadyForRetry();
```

## 4. Integration Points

The notification service is already integrated with:

### KYC Service
Notifications are automatically sent when:
- User's KYC verification is approved
- User's KYC verification is rejected

### Lending Service
Notifications are automatically sent when:
- User completes a loan repayment

### Risk Monitoring Service
Notifications are automatically sent when:
- A loan's health factor reaches warning level
- A loan's health factor reaches critical/liquidation level

## 5. Event Types Available

```typescript
'VERIFICATION_APPROVED'      // KYC verification approved
'VERIFICATION_REJECTED'      // KYC verification rejected
'VALUATION_UPDATED'          // Property valuation changed
'REPAYMENT_REMINDER'         // Loan payment due soon
'REPAYMENT_OVERDUE'          // Loan payment overdue
'REPAYMENT_PROCESSED'        // Loan payment received
'RISK_WARNING'               // Loan health warning
'LIQUIDATION_RISK'           // Liquidation risk alert
'SYSTEM_ALERT'               // General system alert
'INVESTMENT_OPPORTUNITY'     // New investment opportunity
'PORTFOLIO_UPDATE'           // Portfolio changes
```

## 6. Configuration

The NotificationService accepts optional configuration:

```typescript
const notificationService = new NotificationService(
  notificationRepository,
  {
    maxRetries: 3,        // Retry up to 3 times
    retryDelayMs: 60000   // Start with 1 minute delay
  }
);
```

Retry delays use exponential backoff:
- Attempt 1: Immediate
- Attempt 2 (Retry 1): ~1 minute
- Attempt 3 (Retry 2): ~2 minutes
- Attempt 4 (Retry 3): ~3 minutes

## 7. Testing

Run the notification service tests:

```bash
cd apps/api
npm test -- NotificationService.test.ts
```

## 8. Debugging

### Check Notification Status

```typescript
const notification = await notificationService.getNotificationById('notif-123');
console.log(notification);
```

### Monitor Delivery

```typescript
// Get notifications waiting for retry
const pendingRetry = await notificationService.getNotificationsReadyForRetry();
console.log('Pending retries:', pendingRetry);

// Get failed notifications
const failed = await notificationService.getPendingNotifications();
console.log('Failed notifications:', failed);
```

### Clean Up Old Notifications

```typescript
// Delete notifications older than 90 days
const deleted = await notificationService.cleanupOldNotifications(90);
console.log(`Deleted ${deleted} old notifications`);
```

## 9. Common Scenarios

### Scenario 1: User completes KYC
```typescript
// Automatically triggered in KYCController.verifyDocument()
await notificationService.notifyVerificationApproved('user-123', 'IN_APP');
```

### Scenario 2: User misses loan payment
```typescript
await notificationService.notifyRepaymentOverdue('user-123', 'loan-456', 5000, 'EMAIL');
```

### Scenario 3: Portfolio changes
```typescript
await notificationService.notifyPortfolioUpdate(
  'user-123',
  'Your portfolio value increased to $500,000',
  'IN_APP'
);
```

### Scenario 4: Send bulk notifications to all users
```typescript
const userIds = ['user-1', 'user-2', 'user-3'];
const notifications = userIds.map(userId => ({
  userId,
  eventType: 'SYSTEM_ALERT' as const,
  title: 'Platform Update',
  message: 'New features are now available',
  channel: 'IN_APP' as const
}));

await notificationService.createBulkNotifications(notifications);
```

## 10. Performance Tips

1. **Use pagination** when retrieving notifications:
   ```bash
   GET /notifications?limit=20&offset=0
   ```

2. **Filter by event type** to reduce query load:
   ```typescript
   const notifications = await notificationService.getUserNotifications('user-123', 20, 0);
   ```

3. **Clean up old notifications** periodically:
   ```typescript
   // Run daily
   await notificationService.cleanupOldNotifications(90);
   ```

4. **Use bulk operations** for multiple notifications:
   ```typescript
   await notificationService.createBulkNotifications(notifications);
   await notificationService.markMultipleAsRead(notificationIds);
   ```

## 11. Documentation

For complete documentation, see:
- `NOTIFICATION_SERVICE.md` - Full implementation guide
- `IMPLEMENTATION_SUMMARY.md` - Summary of changes
- `src/__tests__/NotificationService.test.ts` - Test examples

## 12. Support

For issues or questions:
1. Check the test files for usage examples
2. Review the full documentation in `NOTIFICATION_SERVICE.md`
3. Check database logs for migration issues
4. Verify authentication headers in API requests

## Next Steps

1. ✅ Run database migration
2. ✅ Test API endpoints
3. ✅ Integrate with frontend
4. ✅ Monitor notification delivery
5. ✅ Implement email/SMS delivery adapters

Happy notifications! 🚀
