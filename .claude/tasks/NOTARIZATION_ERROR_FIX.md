# Notarization Error Fix Plan

## Problem Analysis

### Root Cause
The 403 error "Invalid or inaccessible developer team ID for the provided Apple ID" occurs during macOS app notarization. Based on analysis of the GitHub Actions workflow and Apple Developer requirements, the most likely causes are:

1. **Apple ID Permission Issue**: The Apple ID lacks Admin or Account Holder permissions for the specified Team ID
2. **App-Specific Password Issue**: The password is invalid, expired, or not properly configured
3. **Team ID Mismatch**: The Team ID doesn't correspond to the Apple ID's accessible teams
4. **Apple Developer Account Status**: The account may have expired or been suspended

### Current Configuration Analysis
- ✅ Team ID format validation is implemented (10-character alphanumeric)
- ✅ All required secrets are being passed to electron-builder
- ✅ Code signing certificates are properly configured
- ✅ Apple ID validation and format checking implemented
- ✅ Streamlined single notarization method (Apple ID)

### Error Details
- **HTTP 403**: "Invalid or inaccessible developer team ID"
- **Cause**: electron-builder attempts notarization with potentially incorrect/invalid Apple Team ID
- **Location**: The error occurs during the macOS build process in GitHub Actions

## Investigation Results

### Current Configuration Status
✅ **Certificate signing**: Working (fixed in previous session)  
❌ **Notarization**: Conflicting configuration  
✅ **GitHub secrets**: Properly configured in workflow  
✅ **Workflow structure**: Correct environment variable passing  

### Key Findings
1. **package.json** explicitly disables notarization: `"notarize": false`
2. **release.yml** passes Apple credentials that trigger automatic notarization
3. **electron-builder** prioritizes environment variables over config file settings
4. **Team ID validation** fails during the notarization attempt

## Solution Options

### Option 1: Enable Proper Notarization (Recommended)
**Best for**: Production releases that need to be distributed outside the App Store

**Steps**:
1. Enable notarization in package.json
2. Verify Apple Team ID format and validity
3. Ensure all Apple credentials are correct
4. Test the notarization process

**Benefits**: 
- Proper macOS security compliance
- No Gatekeeper warnings for users
- Professional app distribution

### Option 2: Disable Notarization Completely
**Best for**: Development builds or internal distribution

**Steps**:
1. Remove Apple credentials from GitHub Actions workflow
2. Keep `"notarize": false` in package.json
3. Add environment variable to explicitly disable notarization

**Benefits**:
- Faster build times
- No Apple Developer account requirements
- Simpler configuration

## Recommended Implementation (Option 1)

### Step 1: Update package.json Configuration
```json
{
  "build": {
    "mac": {
      "notarize": {
        "teamId": "${APPLE_TEAM_ID}"
      }
    }
  }
}
```

### Step 2: Verify Apple Team ID Format
Apple Team ID should be:
- **Format**: 10-character alphanumeric string (e.g., "ABCD123456")
- **Location**: Apple Developer Account → Membership → Team ID
- **Not**: Bundle ID, App ID, or Certificate ID

### Step 3: Update GitHub Secrets Validation
Add validation step in workflow to ensure Team ID format is correct.

### Step 4: Test Configuration
Run a test build to verify notarization works properly.

## Alternative Implementation (Option 2)

### Step 1: Remove Apple Credentials from Workflow
```yaml
# Remove these environment variables:
# APPLE_ID: ${{ secrets.APPLE_ID }}
# APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
# APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
```

### Step 2: Add Explicit Disable Flag
```yaml
env:
  CSC_IDENTITY_AUTO_DISCOVERY: false
```

## Implementation Status

### ✅ Completed
- Root cause analysis of 403 notarization error
- Comprehensive plan document creation
- Enhanced GitHub Actions validation for Apple credentials
- Workflow improvements with better error handling
- Updated electron-builder configuration for Apple ID notarization
- Removed API key complexity for simplified setup

### 🔄 In Progress
- Testing and validation

### ⏳ Pending
- Final testing and deployment

### Changes Made

1. **Updated package.json** (Line 61-63):
   ```json
   "notarize": {
     "teamId": "${APPLE_TEAM_ID}"
   }
   ```
   - Changed from `"notarize": false` to proper notarization configuration
   - Uses environment variable for Team ID (passed from GitHub secrets)

2. **Enhanced GitHub Actions Validation** (.github/workflows/release.yml):
   ```yaml
   # Validate Apple Team ID format
   if [ -n "${{ secrets.APPLE_TEAM_ID }}" ]; then
     TEAM_ID="${{ secrets.APPLE_TEAM_ID }}"
     if [[ ${#TEAM_ID} -eq 10 && "$TEAM_ID" =~ ^[A-Z0-9]+$ ]]; then
       echo "✅ Apple Team ID format is valid: $TEAM_ID"
     else
       echo "❌ Apple Team ID format is invalid: $TEAM_ID"
       echo "Expected: 10-character alphanumeric string (e.g., ABCD123456)"
       exit 1
     fi
   fi
   ```
   - Added Team ID format validation (10-character alphanumeric)
   - Provides clear error messages and guidance
   - Fails fast if Team ID format is incorrect

## Next Steps for User

1. **Verify Team ID in GitHub Secrets**:
   - Go to GitHub repository → Settings → Secrets and variables → Actions
   - Check that `APPLE_TEAM_ID` is exactly 10 characters, alphanumeric only
   - Find correct Team ID at: https://developer.apple.com/account/ → Membership → Team ID

2. **Test the Fix**:
   - Push a new tag or trigger the release workflow
   - The validation step will now check Team ID format before attempting notarization
   - If Team ID is invalid, the build will fail early with clear instructions

3. **Monitor Build Results**:
   - Check GitHub Actions logs for validation messages
   - Successful notarization should complete without HTTP 403 errors

## Required GitHub Secrets

### Apple ID Authentication
- `CSC_LINK`: Base64-encoded .p12 certificate
- `CSC_KEY_PASSWORD`: Password for the .p12 certificate
- `APPLE_ID`: Apple ID email (must have Admin/Account Holder role)
- `APPLE_APP_SPECIFIC_PASSWORD`: App-specific password
- `APPLE_TEAM_ID`: 10-character Team ID

### Apple ID Setup Requirements

1. **Apple ID Permissions**:
   - Your Apple ID must have **Admin** or **Account Holder** role
   - Developer role is insufficient for notarization
   - Verify at [Apple Developer Portal](https://developer.apple.com/account/)

2. **App-Specific Password**:
   - Generate at [Apple ID Account](https://appleid.apple.com/)
   - Go to Sign-In and Security → App-Specific Passwords
   - Create new password for "Xcode" or "Development Tools"

3. **Team ID Verification**:
   - Must be exactly 10 characters (alphanumeric)
   - Found at Developer Portal → Membership → Team ID
   - Example format: `ABCD123456`

## Team ID Verification Guide

### How to Find Your Apple Team ID
1. Log in to [Apple Developer Portal](https://developer.apple.com/account/)
2. Go to **Membership** section
3. Look for **Team ID** (10-character string)
4. **Example format**: `ABCD123456`

### Common Team ID Mistakes
- Using Bundle ID instead of Team ID
- Using Certificate Common Name
- Including extra characters or spaces
- Using App Store Connect Team ID (different from Developer Team ID)

## Risk Assessment

### Option 1 Risks
- Requires valid Apple Developer account
- More complex configuration
- Potential for Team ID validation issues

### Option 2 Risks
- Users will see "unidentified developer" warnings
- May require users to bypass Gatekeeper
- Not suitable for wide distribution

## Success Criteria

✅ **Build completes** without notarization errors  
✅ **macOS app runs** without security warnings (Option 1) or with expected warnings (Option 2)  
✅ **Configuration is consistent** between package.json and workflow  
✅ **Documentation updated** with correct process  

---

**Recommendation**: Implement Option 1 (proper notarization) for production releases, as it provides the best user experience and follows Apple's security guidelines.