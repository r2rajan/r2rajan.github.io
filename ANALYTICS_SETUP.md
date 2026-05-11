# Google Analytics 4 Setup Guide

## What's Been Configured

I've added Google Analytics 4 tracking to your GitHub Pages site:

1. **_config.yml** - Added analytics configuration
2. **_includes/analytics.html** - GA4 tracking code
3. **_includes/head.html** - Custom head include with analytics

## Next Steps: Get Your Google Analytics Tracking ID

### 1. Create Google Analytics Account
1. Go to https://analytics.google.com/
2. Sign in with your Google account
3. Click **"Start measuring"** or **"Admin"** (gear icon)

### 2. Create a Property (GA4)
1. Click **"Create Property"**
2. Enter property details:
   - **Property name**: "Ramesh's Tech Blog" (or your preference)
   - **Reporting time zone**: Your timezone
   - **Currency**: USD (or your preference)
3. Click **"Next"**

### 3. Set Up Data Stream
1. Choose **"Web"** as the platform
2. Enter your website details:
   - **Website URL**: `https://r2rajan.github.io`
   - **Stream name**: "GitHub Pages Blog"
3. Click **"Create stream"**

### 4. Get Your Measurement ID
1. You'll see a **Measurement ID** that looks like: `G-XXXXXXXXXX`
2. **Copy this ID**

### 5. Update Your Site Configuration
1. Open `_config.yml`
2. Find this section:
   ```yaml
   analytics:
     provider: "google-gtag"
     google:
       tracking_id: "G-XXXXXXXXXX"  # Replace with your GA4 Measurement ID
   ```
3. Replace `G-XXXXXXXXXX` with your actual Measurement ID
4. Save the file

### 6. Deploy Your Changes
```bash
git add _config.yml _includes/analytics.html _includes/head.html
git commit -m "Add Google Analytics 4 tracking"
git push origin main
```

Wait 2-5 minutes for GitHub Pages to rebuild your site.

### 7. Test Your Analytics
1. Visit your website: https://r2rajan.github.io
2. In Google Analytics, go to **Reports** → **Realtime**
3. You should see your visit appear within 30 seconds!

## How to Track Top Performing Posts

### After 1-2 weeks of data collection:

1. Go to **Reports** → **Engagement** → **Pages and screens**
2. You'll see a list of all pages with:
   - Views
   - Users
   - Average engagement time
   - Events per session

3. To see your top blog posts specifically:
   - Click **Add filter** (funnel icon)
   - Select "Page path and screen class"
   - Choose "contains" and enter `/20` (to filter blog posts by year)
   - Apply filter

4. **Export Reports**:
   - Click the **Share** icon (top right)
   - Download as PDF, CSV, or Google Sheets

### Custom Reports for Blog Posts

1. Go to **Explore** (left sidebar)
2. Create a **Free form exploration**
3. Add dimensions:
   - Page path
   - Page title
4. Add metrics:
   - Views
   - Users
   - Average engagement time
5. Save and bookmark this report

## Privacy & GDPR Compliance

The current setup:
- ✅ Uses GA4 (more privacy-focused than Universal Analytics)
- ✅ Cookie consent flags set
- ⚠️ IP anonymization is OFF (set to `true` in _config.yml if needed for GDPR)

To enable IP anonymization:
```yaml
analytics:
  provider: "google-gtag"
  google:
    tracking_id: "G-XXXXXXXXXX"
    anonymize_ip: true  # Change to true
```

## Troubleshooting

### Analytics not showing data?
1. Check your Measurement ID is correct in `_config.yml`
2. Verify site has been rebuilt (check GitHub Actions)
3. Clear browser cache
4. Check browser console for errors (F12)
5. Verify GA4 tag with Google Tag Assistant extension

### Want to exclude your own visits?
1. In Google Analytics, go to **Admin** → **Data Streams**
2. Click your stream → **Configure tag settings** → **Show all**
3. Click **Define internal traffic**
4. Add your IP address

## Additional Resources

- [Google Analytics 4 Documentation](https://support.google.com/analytics/answer/9304153)
- [GA4 Setup Guide](https://support.google.com/analytics/answer/9304153)
- [Understanding GA4 Reports](https://support.google.com/analytics/answer/9143382)
