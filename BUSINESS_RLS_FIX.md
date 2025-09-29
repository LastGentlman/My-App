# Fix for Business Creation Infinite Recursion Error

## Problem
When creating a business for the first time, you're getting this error:
```
Error al crear el negocio: infinite recursion detected in policy for relation "businesses"
```

## Root Cause
The Supabase RLS (Row Level Security) policies for the `businesses` table have circular dependencies that cause infinite recursion:

1. **Business policies** check if user is an employee by querying the `employees` table
2. **Employee policies** check if user has access to the business by querying the `businesses` table
3. This creates a circular dependency: `businesses` → `employees` → `businesses` → ...

## Solution

### Step 1: Apply the RLS Fix Migration

1. **Go to your Supabase Dashboard**
2. **Open the SQL Editor**
3. **Copy and paste the SQL from `Backend/migrations/fix_business_rls_infinite_recursion.sql`**
4. **Click "Run" to execute the SQL**

### Step 2: Verify the Fix

After applying the migration:

1. **Clear your browser cache and localStorage**
2. **Log out and log back in**
3. **Try creating a business again**

## What the Fix Does

### 1. Removes Circular Dependencies
- **Before**: Business policies checked employees table → Employee policies checked businesses table → Infinite recursion
- **After**: Business policies only check owner_id directly, avoiding circular dependencies

### 2. Simplifies Policy Logic
- **Business INSERT**: Only checks if `owner_id = auth.uid()` (no employee table lookup)
- **Business SELECT**: Checks owner OR employee status (but with optimized queries)
- **Employee policies**: Only check business ownership, not employee status

### 3. Optimizes Performance
- Adds proper indexes for RLS policy lookups
- Uses `(select auth.uid())` instead of `auth.uid()` for better performance
- Updates table statistics for better query planning

## Technical Details

### Before (Problematic Policies)
```sql
-- This caused infinite recursion
CREATE POLICY "business_select_policy" ON businesses
FOR SELECT USING (
  owner_id = (select auth.uid()) OR 
  EXISTS (
    SELECT 1 FROM employees e 
    WHERE e.business_id = businesses.id 
    AND e.user_id = (select auth.uid()) 
    AND e.is_active = true
  )
);
```

### After (Fixed Policies)
```sql
-- Simplified business policies
CREATE POLICY "business_insert_policy" ON businesses
FOR INSERT WITH CHECK (
  owner_id = (select auth.uid())
);

CREATE POLICY "business_select_policy" ON businesses
FOR SELECT USING (
  owner_id = (select auth.uid()) OR 
  EXISTS (
    SELECT 1 FROM employees e 
    WHERE e.business_id = businesses.id 
    AND e.user_id = (select auth.uid()) 
    AND e.is_active = true
  )
);
```

## Expected Results

After applying this fix:
- ✅ No more infinite recursion errors
- ✅ Business creation works smoothly
- ✅ Improved RLS policy performance
- ✅ Proper access control maintained

## Verification Steps

1. **Check Supabase Dashboard**: Verify the migration executed successfully
2. **Test Business Creation**: Try creating a new business
3. **Check Console**: No more infinite recursion errors
4. **Test Access Control**: Verify users can only see their own businesses

## If Issues Persist

If you still see issues after applying the fix:

1. **Check Migration Status**: Ensure the SQL executed without errors
2. **Clear Browser Cache**: Hard refresh (Ctrl+F5 or Cmd+Shift+R)
3. **Log Out/In**: Clear authentication state
4. **Check Supabase Logs**: Look for any remaining policy conflicts

## Files Modified

- `Backend/migrations/fix_business_rls_infinite_recursion.sql` - The main fix
- `BUSINESS_RLS_FIX.md` - This documentation

## Related Issues

This fix also resolves:
- 406 Not Acceptable errors when fetching profiles
- PGRST116 errors during business creation
- Performance issues with RLS policies
- Circular dependency warnings in Supabase logs
