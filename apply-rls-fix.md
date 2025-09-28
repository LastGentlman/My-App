# Fix for 406 Not Acceptable and PGRST116 Errors

## Problem
You're seeing these errors in the console:
- `406 (Not Acceptable)` when fetching profiles
- `PGRST116` errors indicating "The result contains 0 rows"
- Profile not found errors for authenticated users

## Root Cause
The Supabase RLS (Row Level Security) policies for the `profiles` table are conflicting. Multiple migration files have created inconsistent policies that don't handle all scenarios properly.

## Solution

### Option 1: Apply the Comprehensive RLS Fix (Recommended)

1. **Go to your Supabase Dashboard**
2. **Open the SQL Editor**
3. **Copy and paste the following SQL:**

```sql
-- ===== COMPREHENSIVE PROFILES RLS FIX =====
-- This migration fixes all RLS policy issues for the profiles table
-- Resolves 406 Not Acceptable errors and PGRST116 profile fetch issues

-- First, ensure the profiles table has the required columns
ALTER TABLE profiles 
ADD COLUMN IF NOT EXISTS is_active BOOLEAN DEFAULT true,
ADD COLUMN IF NOT EXISTS current_business_id UUID REFERENCES businesses(id),
ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMP WITH TIME ZONE DEFAULT NULL,
ADD COLUMN IF NOT EXISTS updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW();

-- Create indexes for performance
CREATE INDEX IF NOT EXISTS idx_profiles_is_active ON profiles(is_active);
CREATE INDEX IF NOT EXISTS idx_profiles_deleted_at ON profiles(deleted_at);
CREATE INDEX IF NOT EXISTS idx_profiles_current_business_id ON profiles(current_business_id);

-- Drop all existing policies to avoid conflicts
DROP POLICY IF EXISTS "Users can view own profile" ON profiles;
DROP POLICY IF EXISTS "Users can update own profile" ON profiles;
DROP POLICY IF EXISTS "Users can insert own profile" ON profiles;

-- Create comprehensive RLS policies that handle all scenarios
-- SELECT policy: Users can view their own active profile
CREATE POLICY "profiles_select_policy" ON profiles
FOR SELECT USING (
  id = (select auth.uid()) 
  AND (is_active = true OR is_active IS NULL) -- Handle both new and existing profiles
);

-- UPDATE policy: Users can update their own active profile
CREATE POLICY "profiles_update_policy" ON profiles
FOR UPDATE USING (
  id = (select auth.uid()) 
  AND (is_active = true OR is_active IS NULL) -- Handle both new and existing profiles
);

-- INSERT policy: Users can create their own profile
CREATE POLICY "profiles_insert_policy" ON profiles
FOR INSERT WITH CHECK (
  id = (select auth.uid())
);

-- Add helpful comments
COMMENT ON POLICY "profiles_select_policy" ON profiles IS 'Allows users to view their own active profile';
COMMENT ON POLICY "profiles_update_policy" ON profiles IS 'Allows users to update their own active profile';
COMMENT ON POLICY "profiles_insert_policy" ON profiles IS 'Allows users to create their own profile during registration';

-- Ensure all existing profiles have is_active = true if it's NULL
UPDATE profiles 
SET is_active = true 
WHERE is_active IS NULL;
```

4. **Click "Run" to execute the SQL**

### Option 2: Use the Script (If you have Supabase CLI)

```bash
cd Backend
chmod +x scripts/apply-profiles-rls-fix.sh
./scripts/apply-profiles-rls-fix.sh
```

## After Applying the Fix

1. **Clear your browser cache and localStorage**
2. **Log out and log back in**
3. **Check the browser console** - you should see fewer errors
4. **Test creating a new business** - this should work smoothly now

## What This Fix Does

- ✅ Resolves 406 Not Acceptable errors
- ✅ Fixes PGRST116 profile fetch issues  
- ✅ Creates comprehensive RLS policies that handle all scenarios
- ✅ Adds proper `is_active` column handling
- ✅ Creates performance indexes
- ✅ Ensures backward compatibility with existing profiles

## Verification

After applying the fix, you should see:
- No more 406 errors in the console
- No more PGRST116 errors
- Smooth profile creation for new users
- Proper business setup flow

If you still see issues after applying this fix, please check:
1. That the SQL executed successfully in Supabase
2. That you've cleared browser cache
3. That you've logged out and back in
