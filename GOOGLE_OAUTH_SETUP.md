# Google Calendar OAuth Setup Guide

This guide will walk you through setting up Google OAuth to enable Google Calendar integration in the LAMC website admin portal.

## Overview

The application has **built-in Google Calendar OAuth support**. When configured, it allows:
- **Automatic calendar event creation** when appointments are confirmed
- **Google Calendar sync** for inspection bookings
- **Calendar invite emails** sent to customers
- **Real-time availability checking** from your Google Calendar

## Prerequisites

- A Google account (Gmail or Google Workspace)
- Access to [Google Cloud Console](https://console.cloud.google.com/)
- Your application's domain URL (for production) or localhost URL (for development)

## Step-by-Step Setup

### Step 1: Create a Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Click the project dropdown at the top
3. Click **"New Project"**
4. Enter a project name (e.g., "LAMC Website Calendar")
5. Click **"Create"**
6. Wait for the project to be created, then select it

### Step 2: Enable Google Calendar API

1. In the Google Cloud Console, go to **"APIs & Services"** > **"Library"**
2. Search for **"Google Calendar API"**
3. Click on **"Google Calendar API"**
4. Click **"Enable"**
5. Wait for the API to be enabled

### Step 3: Create OAuth 2.0 Credentials

1. Go to **"APIs & Services"** > **"Credentials"**
2. Click **"+ CREATE CREDENTIALS"** at the top
3. Select **"OAuth client ID"**
4. If prompted, configure the OAuth consent screen first:
   - Choose **"External"** (unless you have Google Workspace)
   - Fill in the required fields:
     - **App name**: "LAMC Painting Admin Portal"
     - **User support email**: Your email (e.g., luis@lamcpainting.com)
     - **Developer contact information**: Your email
   - Click **"Save and Continue"**
   - On **Scopes** page, click **"Save and Continue"**
   - On **Test users** page, add your admin email addresses:
     - brian@lamcpainting.com
     - isiah@lamcpainting.com
     - luis@lamcpainting.com
     - brittany@lamcpainting.com
   - Click **"Save and Continue"**
   - Review and click **"Back to Dashboard"**

5. Now create the OAuth client:
   - **Application type**: Select **"Web application"**
   - **Name**: "LAMC Website Admin Portal"
   - **Authorized JavaScript origins**:
     ```
     http://localhost:3000
     https://your-domain.com
     ```
     (Replace `your-domain.com` with your actual production domain)
   - **Authorized redirect URIs**:
     ```
     http://localhost:3000/api/auth/callback/google
     https://your-domain.com/api/auth/callback/google
     ```
     (Replace `your-domain.com` with your actual production domain)
   - Click **"Create"**

6. **Important**: Copy the credentials:
   - **Client ID**: Copy this value (starts with something like `123456789-abc...`)
   - **Client secret**: Click "Show" and copy this value
   - Keep these secure - you'll need them in the next step

### Step 4: Add Credentials to Environment Variables

#### For Local Development

1. Open your `.env.local` file in the project root
2. Add the following lines:
   ```bash
   GOOGLE_CLIENT_ID="your-client-id-here"
   GOOGLE_CLIENT_SECRET="your-client-secret-here"
   ```
3. Replace the values with your actual Client ID and Client Secret
4. Save the file
5. Restart your development server (`npm run dev`)

#### For Production (Vercel)

1. Go to your [Vercel Dashboard](https://vercel.com/dashboard)
2. Select your project
3. Go to **Settings** > **Environment Variables**
4. Add two new variables:
   - **Name**: `GOOGLE_CLIENT_ID`
     **Value**: Your Client ID
   - **Name**: `GOOGLE_CLIENT_SECRET`
     **Value**: Your Client Secret
5. Make sure to select the **Production** environment (and Preview/Development if needed)
6. Click **"Save"**
7. **Redeploy** your application for changes to take effect

### Step 5: Test the Integration

1. **Restart your application** (if running locally)
2. Log into the admin portal at `/admin/login`
3. You should see a **"Sign in with Google"** button (if credentials are configured)
4. When you click it, you'll be redirected to Google to authorize the application
5. Grant the requested permissions (Calendar access)
6. After authorization, you'll be redirected back to the admin portal

### Step 6: Verify Calendar Integration

1. Go to any ticket/inspection in the admin portal
2. When you confirm an appointment, check if:
   - The calendar sync status shows **"Successfully synced to Google Calendar"** (green)
   - A calendar event appears in your Google Calendar
   - The customer receives a calendar invite email

## What the Integration Does

Once configured, the Google Calendar integration will:

1. **Create Calendar Events**: When you confirm an inspection appointment, it automatically creates an event on your Google Calendar
2. **Send Invites**: Customers receive calendar invites via email (both Google Calendar invites and .ics file attachments)
3. **Check Availability**: The system can check your calendar for blocked times
4. **Store Event IDs**: Calendar event IDs are stored in the database for future reference

## Troubleshooting

### "Google Calendar OAuth not configured" Warning

**Cause**: The `GOOGLE_CLIENT_ID` and/or `GOOGLE_CLIENT_SECRET` environment variables are not set.

**Solution**:
1. Verify the variables are in your `.env.local` file (local) or Vercel environment variables (production)
2. Restart your development server or redeploy (production)
3. Check that there are no typos in the variable names

### "Invalid credentials" Error

**Cause**: The Client ID or Client Secret is incorrect.

**Solution**:
1. Double-check that you copied the credentials correctly
2. Make sure there are no extra spaces or quotes in the environment variables
3. Regenerate the credentials in Google Cloud Console if needed

### "Redirect URI mismatch" Error

**Cause**: The redirect URI in your OAuth client doesn't match your application URL.

**Solution**:
1. Go to Google Cloud Console > APIs & Services > Credentials
2. Click on your OAuth 2.0 Client ID
3. Add the correct redirect URI:
   - For local: `http://localhost:3000/api/auth/callback/google`
   - For production: `https://your-domain.com/api/auth/callback/google`
4. Save the changes

### Calendar Events Not Creating

**Cause**: The user hasn't authorized calendar access, or the access token has expired.

**Solution**:
1. Make sure you signed in with Google OAuth (not just credentials)
2. Check that you granted calendar permissions during OAuth
3. Sign out and sign back in with Google to refresh the token
4. Check the browser console and server logs for error messages

### "Access denied" or "Insufficient permissions"

**Cause**: The OAuth consent screen is in testing mode and the user isn't added as a test user.

**Solution**:
1. Go to Google Cloud Console > APIs & Services > OAuth consent screen
2. Add the user's email to the "Test users" list
3. Or publish the app (if ready for production)

## Security Notes

- **Never commit** your `.env.local` file to version control
- Keep your Client Secret secure - treat it like a password
- Regularly rotate credentials if compromised
- Use different credentials for development and production environments

## Next Steps

After setting up Google OAuth:

1. **Test calendar event creation** by confirming an inspection appointment
2. **Verify email invites** are being sent to customers
3. **Check your Google Calendar** to see events appearing automatically
4. **Monitor the calendar sync status** in the admin portal ticket details

## Additional Resources

- [Google Calendar API Documentation](https://developers.google.com/calendar/api)
- [NextAuth.js Google Provider Documentation](https://next-auth.js.org/providers/google)
- [OAuth 2.0 Setup Guide](https://developers.google.com/identity/protocols/oauth2)

---

**Need Help?** Check the application logs or contact your development team.




