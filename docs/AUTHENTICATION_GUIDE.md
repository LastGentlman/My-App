# Authentication Guide

This comprehensive guide covers all authentication implementations, OAuth integrations, and user management features.

## 🔐 Authentication Overview

The application supports multiple authentication methods with secure session management and comprehensive user account features.

## 📋 Authentication Features

### 1. Authentication Methods
- **Email/Password Authentication**
- **OAuth Integration** (Google, GitHub, etc.)
- **Phone Number Authentication**
- **Magic Link Authentication**
- **Offline Authentication Support**

### 2. User Management
- **User Registration & Login**
- **Email Verification**
- **Password Reset & Recovery**
- **Account Deletion**
- **Profile Management**

### 3. Session Management
- **Secure Session Tokens**
- **Session Timeout**
- **Concurrent Session Control**
- **Session Invalidation**

## 🚀 OAuth Implementation

### Supported OAuth Providers
- **Google OAuth 2.0**
- **GitHub OAuth**
- **Custom OAuth Providers**

### OAuth Configuration

#### Google OAuth Setup
```typescript
// OAuth Configuration
const oauthConfig = {
  google: {
    clientId: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    redirectUri: process.env.GOOGLE_REDIRECT_URI,
    scope: ['openid', 'profile', 'email']
  }
};
```

#### OAuth Flow Implementation
```typescript
// OAuth Login Handler
export const handleOAuthLogin = async (provider: string) => {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: provider as any,
    options: {
      redirectTo: `${window.location.origin}/auth/callback`
    }
  });
  
  if (error) throw error;
  return data;
};
```

### OAuth Callback Handling
```typescript
// OAuth Callback Handler
export const handleOAuthCallback = async () => {
  const { data, error } = await supabase.auth.getSession();
  
  if (error) {
    throw new Error(`OAuth callback error: ${error.message}`);
  }
  
  if (data.session) {
    // Handle successful authentication
    await updateUserProfile(data.session.user);
    return data.session;
  }
  
  throw new Error('No session found');
};
```

### OAuth Avatar Handling
```typescript
// Handle OAuth Avatar Updates
export const handleOAuthAvatar = async (user: User) => {
  if (user.user_metadata?.avatar_url) {
    const avatarUrl = user.user_metadata.avatar_url;
    
    // Update user profile with avatar
    await supabase
      .from('profiles')
      .update({ avatar_url: avatarUrl })
      .eq('id', user.id);
      
    return avatarUrl;
  }
};
```

## 🔑 Password Management

### Password Requirements
- **Minimum Length**: 8 characters
- **Complexity**: Mixed case, numbers, symbols
- **History**: Cannot reuse last 5 passwords
- **Strength Meter**: Real-time validation

### Password Strength Implementation
```typescript
// Password Strength Validation
export const validatePasswordStrength = (password: string) => {
  const requirements = {
    length: password.length >= 8,
    lowercase: /[a-z]/.test(password),
    uppercase: /[A-Z]/.test(password),
    numbers: /\d/.test(password),
    symbols: /[!@#$%^&*(),.?":{}|<>]/.test(password)
  };
  
  const score = Object.values(requirements).filter(Boolean).length;
  return { requirements, score, isValid: score >= 4 };
};
```

### Password Change Implementation
```typescript
// Change Password Handler
export const changePassword = async (
  currentPassword: string,
  newPassword: string
) => {
  // Verify current password
  const { error: verifyError } = await supabase.auth.signInWithPassword({
    email: user.email,
    password: currentPassword
  });
  
  if (verifyError) throw new Error('Current password is incorrect');
  
  // Update password
  const { error: updateError } = await supabase.auth.updateUser({
    password: newPassword
  });
  
  if (updateError) throw updateError;
};
```

## 📧 Email Authentication

### Email Verification
```typescript
// Send Email Verification
export const sendEmailVerification = async (email: string) => {
  const { error } = await supabase.auth.resend({
    type: 'signup',
    email: email
  });
  
  if (error) throw error;
};
```

### Email Confirmation Handling
```typescript
// Handle Email Confirmation
export const handleEmailConfirmation = async (token: string) => {
  const { data, error } = await supabase.auth.verifyOtp({
    token_hash: token,
    type: 'email'
  });
  
  if (error) throw error;
  return data;
};
```

### Magic Link Authentication
```typescript
// Send Magic Link
export const sendMagicLink = async (email: string) => {
  const { error } = await supabase.auth.signInWithOtp({
    email: email,
    options: {
      emailRedirectTo: `${window.location.origin}/auth/callback`
    }
  });
  
  if (error) throw error;
};
```

## 📱 Phone Authentication

### Phone Number Validation
```typescript
// Phone Number Validation
export const validatePhoneNumber = (phone: string) => {
  const phoneRegex = /^\+?[1-9]\d{1,14}$/;
  return phoneRegex.test(phone);
};
```

### Phone OTP Implementation
```typescript
// Send Phone OTP
export const sendPhoneOTP = async (phone: string) => {
  const { error } = await supabase.auth.signInWithOtp({
    phone: phone
  });
  
  if (error) throw error;
};

// Verify Phone OTP
export const verifyPhoneOTP = async (phone: string, token: string) => {
  const { data, error } = await supabase.auth.verifyOtp({
    phone: phone,
    token: token,
    type: 'sms'
  });
  
  if (error) throw error;
  return data;
};
```

## 🔄 Session Management

### Session Configuration
```typescript
// Session Configuration
const sessionConfig = {
  timeout: 3600, // 1 hour
  refreshThreshold: 300, // 5 minutes before expiry
  maxConcurrentSessions: 3
};
```

### Session Monitoring
```typescript
// Session Monitoring
export const monitorSession = () => {
  supabase.auth.onAuthStateChange((event, session) => {
    if (event === 'SIGNED_OUT' || event === 'TOKEN_REFRESHED') {
      // Handle session changes
      handleSessionChange(event, session);
    }
  });
};
```

### Session Invalidation
```typescript
// Invalidate All Sessions
export const invalidateAllSessions = async () => {
  const { error } = await supabase.auth.signOut({ scope: 'global' });
  if (error) throw error;
};
```

## 👤 User Profile Management

### Profile Creation
```typescript
// Create User Profile
export const createUserProfile = async (user: User) => {
  const profile = {
    id: user.id,
    email: user.email,
    full_name: user.user_metadata?.full_name || '',
    avatar_url: user.user_metadata?.avatar_url || null,
    created_at: new Date().toISOString()
  };
  
  const { error } = await supabase
    .from('profiles')
    .insert(profile);
    
  if (error) throw error;
  return profile;
};
```

### Profile Updates
```typescript
// Update User Profile
export const updateUserProfile = async (
  userId: string,
  updates: Partial<Profile>
) => {
  const { data, error } = await supabase
    .from('profiles')
    .update(updates)
    .eq('id', userId)
    .select()
    .single();
    
  if (error) throw error;
  return data;
};
```

## 🔒 Account Security

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

### Two-Factor Authentication
```typescript
// 2FA Setup
export const setup2FA = async () => {
  const { data, error } = await supabase.auth.mfa.enroll({
    factorType: 'totp'
  });
  
  if (error) throw error;
  return data;
};
```

## 🗑️ Account Deletion

### Account Deletion Request
```typescript
// Request Account Deletion
export const requestAccountDeletion = async (reason?: string) => {
  const { data, error } = await supabase
    .from('account_deletion_logs')
    .insert({
      user_id: user.id,
      user_email: user.email,
      deletion_reason: reason,
      grace_period_end: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days
    });
    
  if (error) throw error;
  return data;
};
```

### Immediate Account Deletion
```typescript
// Immediate Account Deletion
export const deleteAccountImmediately = async (userId: string) => {
  const { error } = await supabase.rpc('delete_user_completely', {
    user_id_to_delete: userId
  });
  
  if (error) throw error;
};
```

## 🔄 Account Recovery

### Password Reset
```typescript
// Request Password Reset
export const requestPasswordReset = async (email: string) => {
  const { error } = await supabase.auth.resetPasswordForEmail(email, {
    redirectTo: `${window.location.origin}/auth/reset-password`
  });
  
  if (error) throw error;
};
```

### Account Recovery Flow
```typescript
// Account Recovery Implementation
export const recoverAccount = async (email: string) => {
  // Check if account exists
  const { data: profile } = await supabase
    .from('profiles')
    .select('*')
    .eq('email', email)
    .single();
    
  if (!profile) {
    throw new Error('Account not found');
  }
  
  // Send recovery email
  await sendRecoveryEmail(email);
};
```

## 🔧 Authentication Configuration

### Environment Variables
```bash
# Authentication Configuration
SUPABASE_URL=your-supabase-url
SUPABASE_ANON_KEY=your-supabase-anon-key
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
```

### Supabase Configuration
```typescript
// Supabase Client Configuration
const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  {
    auth: {
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: true
    }
  }
);
```

## 🧪 Authentication Testing

### Test Authentication Flows
```typescript
// Test Authentication
describe('Authentication', () => {
  test('should login with email and password', async () => {
    const { data, error } = await supabase.auth.signInWithPassword({
      email: 'test@example.com',
      password: 'password123'
    });
    
    expect(error).toBeNull();
    expect(data.session).toBeDefined();
  });
  
  test('should handle OAuth login', async () => {
    const { data, error } = await supabase.auth.signInWithOAuth({
      provider: 'google'
    });
    
    expect(error).toBeNull();
    expect(data.url).toBeDefined();
  });
});
```

## 📚 Best Practices

### Security Best Practices
1. **Always validate user input**
2. **Use HTTPS for all authentication**
3. **Implement proper session management**
4. **Use secure password requirements**
5. **Enable account lockout protection**

### Development Best Practices
1. **Use environment variables for secrets**
2. **Implement proper error handling**
3. **Log authentication events**
4. **Test all authentication flows**
5. **Keep authentication libraries updated**

### User Experience Best Practices
1. **Provide clear error messages**
2. **Implement loading states**
3. **Use progressive enhancement**
4. **Support offline authentication**
5. **Provide account recovery options**

## 🆘 Troubleshooting

### Common Issues
1. **OAuth callback errors** - Check redirect URIs
2. **Session timeout** - Implement session refresh
3. **Password validation** - Check requirements
4. **Email delivery** - Verify SMTP configuration
5. **Phone verification** - Check SMS provider setup

### Debug Authentication
```typescript
// Debug Authentication State
export const debugAuth = () => {
  supabase.auth.onAuthStateChange((event, session) => {
    console.log('Auth Event:', event);
    console.log('Session:', session);
  });
};
```

---

**Last Updated**: [Current Date]  
**Version**: 1.0  
**Maintained By**: Authentication Team
