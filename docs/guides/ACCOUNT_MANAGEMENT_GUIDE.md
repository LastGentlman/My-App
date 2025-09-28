# Account Management Guide

This comprehensive guide covers all account management features including deletion, recovery, and user lifecycle management.

## 👤 Account Management Overview

The application provides comprehensive account management features including secure account deletion, recovery flows, and user lifecycle management.

## 📋 Account Management Features

### 1. Account Deletion
- **Grace Period Deletion** - 7-day grace period before permanent deletion
- **Immediate Deletion** - For urgent account removal
- **Data Cleanup** - Complete removal of user data
- **Audit Trail** - Full logging of deletion process

### 2. Account Recovery
- **Password Reset** - Secure password recovery
- **Account Reactivation** - Restore deleted accounts
- **Data Recovery** - Recover user data when possible
- **Recovery Verification** - Secure recovery process

### 3. User Lifecycle
- **Account Creation** - User registration process
- **Account Verification** - Email/phone verification
- **Account Updates** - Profile and preference updates
- **Account Suspension** - Temporary account restrictions

## 🗑️ Account Deletion System

### Grace Period Deletion

#### Deletion Request Process
```typescript
// Request Account Deletion
export const requestAccountDeletion = async (
  userId: string,
  reason?: string
) => {
  const deletionLog = {
    user_id: userId,
    user_email: user.email,
    deletion_reason: reason,
    grace_period_end: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days
  };
  
  const { data, error } = await supabase
    .from('account_deletion_logs')
    .insert(deletionLog);
    
  if (error) throw error;
  return data;
};
```

#### Deletion Trigger Handler
```sql
-- Account Deletion Trigger Function
CREATE OR REPLACE FUNCTION handle_account_deletion_request()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  -- Mark user as deleted in auth.users metadata
  UPDATE auth.users
  SET
    raw_user_meta_data = COALESCE(raw_user_meta_data, '{}'::jsonb) ||
                         jsonb_build_object(
                           'account_deleted', true,
                           'deleted_at', now()::text,
                           'deletion_log_id', NEW.id::text
                         ),
    user_metadata = COALESCE(user_metadata, '{}'::jsonb) ||
                    jsonb_build_object(
                      'account_deleted', true,
                      'deleted_at', now()::text,
                      'deletion_log_id', NEW.id::text
                    ),
    updated_at = now()
  WHERE id = NEW.user_id;
  
  -- Invalidate all active sessions
  DELETE FROM auth.sessions WHERE user_id = NEW.user_id;
  
  -- Log audit trail
  INSERT INTO audit_logs (action, user_id, details, created_at)
  VALUES (
    'ACCOUNT_DELETION_INITIATED',
    NEW.user_id,
    jsonb_build_object(
      'deletion_log_id', NEW.id,
      'reason', NEW.deletion_reason,
      'grace_period_end', NEW.grace_period_end,
      'user_email', NEW.user_email
    ),
    now()
  );
  
  RETURN NEW;
END;
$$;
```

### Immediate Account Deletion

#### Immediate Deletion Function
```sql
-- Immediate Account Deletion Function
CREATE OR REPLACE FUNCTION delete_user_completely(user_id_to_delete UUID)
RETURNS BOOLEAN
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
BEGIN
  -- Log the deletion request
  INSERT INTO audit_logs (action, user_id, details, created_at)
  VALUES (
    'IMMEDIATE_ACCOUNT_DELETION',
    user_id_to_delete,
    jsonb_build_object(
      'deletion_type', 'immediate',
      'requested_at', NOW()
    ),
    now()
  );
  
  -- Delete user data from all tables (in dependency order)
  DELETE FROM user_preferences WHERE user_id = user_id_to_delete;
  DELETE FROM business_invitation_codes WHERE invited_by = user_id_to_delete;
  DELETE FROM businesses WHERE owner_id = user_id_to_delete;
  DELETE FROM profiles WHERE id = user_id_to_delete;
  DELETE FROM account_recovery_requests WHERE user_id = user_id_to_delete;
  
  -- Delete from auth.users (this will cascade to sessions, etc.)
  DELETE FROM auth.users WHERE id = user_id_to_delete;
  
  -- Log successful deletion
  INSERT INTO audit_logs (action, user_id, details, created_at)
  VALUES (
    'IMMEDIATE_ACCOUNT_DELETION_COMPLETED',
    user_id_to_delete,
    jsonb_build_object(
      'deletion_type', 'immediate',
      'completed_at', NOW()
    ),
    now()
  );
  
  RAISE NOTICE 'Immediate account deletion completed for user: %', user_id_to_delete;
  RETURN TRUE;
END;
$$;
```

### Process Account Deletion After Grace Period

#### Grace Period Processing
```sql
-- Process Account Deletion After Grace Period
CREATE OR REPLACE FUNCTION process_account_deletion(deletion_log_id UUID)
RETURNS BOOLEAN
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  deletion_record RECORD;
  user_id_to_delete UUID;
BEGIN
  -- Get the deletion record
  SELECT * INTO deletion_record 
  FROM account_deletion_logs 
  WHERE id = deletion_log_id AND processed_at IS NULL;
  
  IF NOT FOUND THEN
    RAISE NOTICE 'Deletion record not found or already processed: %', deletion_log_id;
    RETURN FALSE;
  END IF;
  
  user_id_to_delete := deletion_record.user_id;
  
  -- Log the start of deletion process
  INSERT INTO audit_logs (action, user_id, details, created_at)
  VALUES (
    'ACCOUNT_DELETION_PROCESSING',
    user_id_to_delete,
    jsonb_build_object(
      'deletion_log_id', deletion_log_id,
      'grace_period_end', deletion_record.grace_period_end
    ),
    now()
  );
  
  -- Delete user data from all tables (in dependency order)
  DELETE FROM user_preferences WHERE user_id = user_id_to_delete;
  DELETE FROM business_invitation_codes WHERE invited_by = user_id_to_delete;
  DELETE FROM businesses WHERE owner_id = user_id_to_delete;
  DELETE FROM profiles WHERE id = user_id_to_delete;
  DELETE FROM account_recovery_requests WHERE user_id = user_id_to_delete;
  
  -- Delete from auth.users (this will cascade to sessions, etc.)
  DELETE FROM auth.users WHERE id = user_id_to_delete;
  
  -- Mark deletion as processed
  UPDATE account_deletion_logs 
  SET processed_at = NOW() 
  WHERE id = deletion_log_id;
  
  -- Log successful deletion
  INSERT INTO audit_logs (action, user_id, details, created_at)
  VALUES (
    'ACCOUNT_DELETION_COMPLETED',
    user_id_to_delete,
    jsonb_build_object(
      'deletion_log_id', deletion_log_id,
      'processed_at', NOW()
    ),
    now()
  );
  
  RAISE NOTICE 'Account deletion completed for user: %', user_id_to_delete;
  RETURN TRUE;
END;
$$;
```

## 🔄 Account Recovery System

### Account Recovery Request
```typescript
// Account Recovery Request
export const requestAccountRecovery = async (email: string) => {
  // Check if account exists
  const { data: profile } = await supabase
    .from('profiles')
    .select('*')
    .eq('email', email)
    .single();
    
  if (!profile) {
    throw new Error('Account not found');
  }
  
  // Create recovery request
  const { data, error } = await supabase
    .from('account_recovery_requests')
    .insert({
      user_id: profile.id,
      email: email,
      recovery_token: generateRecoveryToken(),
      expires_at: new Date(Date.now() + 24 * 60 * 60 * 1000) // 24 hours
    });
    
  if (error) throw error;
  
  // Send recovery email
  await sendRecoveryEmail(email, data.recovery_token);
  return data;
};
```

### Account Recovery Verification
```typescript
// Verify Account Recovery
export const verifyAccountRecovery = async (token: string) => {
  const { data, error } = await supabase
    .from('account_recovery_requests')
    .select('*')
    .eq('recovery_token', token)
    .eq('used', false)
    .gt('expires_at', new Date().toISOString())
    .single();
    
  if (error || !data) {
    throw new Error('Invalid or expired recovery token');
  }
  
  return data;
};
```

### Account Reactivation
```typescript
// Reactivate Account
export const reactivateAccount = async (token: string, newPassword: string) => {
  const recoveryData = await verifyAccountRecovery(token);
  
  // Update user password
  const { error: passwordError } = await supabase.auth.updateUser({
    password: newPassword
  });
  
  if (passwordError) throw passwordError;
  
  // Mark recovery request as used
  await supabase
    .from('account_recovery_requests')
    .update({ used: true, used_at: new Date().toISOString() })
    .eq('id', recoveryData.id);
    
  return { success: true };
};
```

## 🔐 Account Security Features

### Account Lockout
```typescript
// Account Lockout Implementation
export const handleFailedLogin = async (email: string) => {
  const attempts = await getFailedAttempts(email);
  
  if (attempts >= 5) {
    // Lock account for 15 minutes
    await lockAccount(email, 15 * 60 * 1000);
    throw new Error('Account temporarily locked due to too many failed attempts');
  }
  
  await incrementFailedAttempts(email);
};
```

### Account Suspension
```typescript
// Suspend Account
export const suspendAccount = async (userId: string, reason: string) => {
  const { error } = await supabase
    .from('profiles')
    .update({
      suspended: true,
      suspension_reason: reason,
      suspended_at: new Date().toISOString()
    })
    .eq('id', userId);
    
  if (error) throw error;
  
  // Log suspension
  await supabase
    .from('audit_logs')
    .insert({
      action: 'ACCOUNT_SUSPENDED',
      user_id: userId,
      details: { reason, suspended_at: new Date().toISOString() }
    });
};
```

### Account Unsuspension
```typescript
// Unsuspend Account
export const unsuspendAccount = async (userId: string) => {
  const { error } = await supabase
    .from('profiles')
    .update({
      suspended: false,
      suspension_reason: null,
      suspended_at: null,
      unsuspended_at: new Date().toISOString()
    })
    .eq('id', userId);
    
  if (error) throw error;
  
  // Log unsuspension
  await supabase
    .from('audit_logs')
    .insert({
      action: 'ACCOUNT_UNSUSPENDED',
      user_id: userId,
      details: { unsuspended_at: new Date().toISOString() }
    });
};
```

## 📊 Account Analytics

### Account Statistics
```typescript
// Get Account Statistics
export const getAccountStats = async (userId: string) => {
  const { data: profile } = await supabase
    .from('profiles')
    .select('*')
    .eq('id', userId)
    .single();
    
  const { data: businesses } = await supabase
    .from('businesses')
    .select('id')
    .eq('owner_id', userId);
    
  const { data: auditLogs } = await supabase
    .from('audit_logs')
    .select('action, created_at')
    .eq('user_id', userId)
    .order('created_at', { ascending: false })
    .limit(10);
    
  return {
    profile,
    businessCount: businesses?.length || 0,
    recentActivity: auditLogs || []
  };
};
```

### Account Activity Log
```typescript
// Get Account Activity
export const getAccountActivity = async (userId: string, limit = 50) => {
  const { data, error } = await supabase
    .from('audit_logs')
    .select('*')
    .eq('user_id', userId)
    .order('created_at', { ascending: false })
    .limit(limit);
    
  if (error) throw error;
  return data;
};
```

## 🔧 Account Management Configuration

### Environment Variables
```bash
# Account Management Configuration
ACCOUNT_DELETION_GRACE_PERIOD=604800  # 7 days in seconds
ACCOUNT_RECOVERY_TOKEN_EXPIRY=86400   # 24 hours in seconds
ACCOUNT_LOCKOUT_ATTEMPTS=5
ACCOUNT_LOCKOUT_DURATION=900          # 15 minutes in seconds
```

### Database Configuration
```sql
-- Account Deletion Logs Table
CREATE TABLE account_deletion_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    user_email TEXT NOT NULL,
    deletion_reason TEXT,
    grace_period_end TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT (NOW() + INTERVAL '7 days'),
    processed_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Account Recovery Requests Table
CREATE TABLE account_recovery_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    email TEXT NOT NULL,
    recovery_token TEXT NOT NULL UNIQUE,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    used BOOLEAN DEFAULT FALSE,
    used_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

## 🧪 Account Management Testing

### Test Account Deletion
```typescript
// Test Account Deletion
describe('Account Deletion', () => {
  test('should create deletion request', async () => {
    const { data, error } = await requestAccountDeletion(
      'user-id',
      'User requested deletion'
    );
    
    expect(error).toBeNull();
    expect(data).toBeDefined();
  });
  
  test('should process deletion after grace period', async () => {
    const result = await processAccountDeletion('deletion-log-id');
    expect(result).toBe(true);
  });
});
```

### Test Account Recovery
```typescript
// Test Account Recovery
describe('Account Recovery', () => {
  test('should create recovery request', async () => {
    const { data, error } = await requestAccountRecovery('user@example.com');
    
    expect(error).toBeNull();
    expect(data).toBeDefined();
  });
  
  test('should verify recovery token', async () => {
    const data = await verifyAccountRecovery('valid-token');
    expect(data).toBeDefined();
  });
});
```

## 📚 Best Practices

### Account Deletion Best Practices
1. **Always use grace period** for user-initiated deletions
2. **Log all deletion activities** for audit purposes
3. **Verify user identity** before processing deletions
4. **Clean up all user data** including related records
5. **Notify users** of deletion completion

### Account Recovery Best Practices
1. **Use secure tokens** for recovery links
2. **Set reasonable expiry times** for recovery tokens
3. **Verify user identity** before account reactivation
4. **Log recovery activities** for security monitoring
5. **Provide clear instructions** to users

### Security Best Practices
1. **Implement account lockout** after failed attempts
2. **Use secure session management** for account operations
3. **Audit all account changes** for security monitoring
4. **Implement proper authorization** for account management
5. **Use HTTPS** for all account-related communications

## 🆘 Troubleshooting

### Common Issues
1. **Deletion not processing** - Check grace period and triggers
2. **Recovery tokens not working** - Verify token expiry and usage
3. **Account lockout issues** - Check lockout configuration
4. **Data not being deleted** - Verify cascade relationships
5. **Audit logs missing** - Check trigger functions

### Debug Account Management
```typescript
// Debug Account Management
export const debugAccountManagement = async (userId: string) => {
  const deletionLogs = await supabase
    .from('account_deletion_logs')
    .select('*')
    .eq('user_id', userId);
    
  const recoveryRequests = await supabase
    .from('account_recovery_requests')
    .select('*')
    .eq('user_id', userId);
    
  const auditLogs = await supabase
    .from('audit_logs')
    .select('*')
    .eq('user_id', userId)
    .order('created_at', { ascending: false })
    .limit(10);
    
  return {
    deletionLogs: deletionLogs.data,
    recoveryRequests: recoveryRequests.data,
    auditLogs: auditLogs.data
  };
};
```

---

**Last Updated**: [Current Date]  
**Version**: 1.0  
**Maintained By**: Account Management Team
