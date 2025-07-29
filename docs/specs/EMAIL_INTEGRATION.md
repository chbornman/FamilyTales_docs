# FamilyTales Email Integration Specification

## Overview

FamilyTales uses SendGrid as the primary email service provider for all transactional and notification emails. This document provides comprehensive specifications for email integration, including setup, templates, implementation details, and best practices.

## Table of Contents

1. [SendGrid Integration Setup](#sendgrid-integration-setup)
2. [Email Templates](#email-templates)
3. [Implementation Details](#implementation-details)
4. [Email Analytics and Tracking](#email-analytics-and-tracking)
5. [Best Practices](#best-practices)

## SendGrid Integration Setup

### Account Configuration

1. **API Key Setup**
   ```yaml
   # Environment Variables
   SENDGRID_API_KEY: "SG.xxxxxxxxxx"
   SENDGRID_FROM_EMAIL: "hello@familytales.app"
   SENDGRID_FROM_NAME: "FamilyTales"
   SENDGRID_REPLY_TO: "support@familytales.app"
   SENDGRID_WEBHOOK_SECRET: "whsec_xxxxxxxxxx"
   ```

2. **Domain Authentication**
   - Configure SPF records
   - Set up DKIM authentication
   - Implement DMARC policy
   - Verify sender domain

3. **IP Warming Strategy**
   - Start with 50 emails/day
   - Double volume every 3 days
   - Monitor reputation metrics
   - Reach full volume in 2 weeks

### API Client Implementation

```go
package email

import (
    "github.com/sendgrid/sendgrid-go"
    "github.com/sendgrid/sendgrid-go/helpers/mail"
)

type EmailService struct {
    client *sendgrid.Client
    config EmailConfig
}

type EmailConfig struct {
    APIKey          string
    FromEmail       string
    FromName        string
    ReplyTo         string
    WebhookSecret   string
    TemplateIDs     map[string]string
}

func NewEmailService(config EmailConfig) *EmailService {
    return &EmailService{
        client: sendgrid.NewSendClient(config.APIKey),
        config: config,
    }
}
```

## Email Templates

### Design System

All email templates follow the FamilyTales brand guide with a warm, book-inspired aesthetic:

- **Colors**: Book White (#FDFCFA), Deep Brown (#6B4423), Warm Amber (#D4A574)
- **Typography**: Crimson Text for headers, Source Sans Pro for body
- **Layout**: Center-aligned, max-width 600px, responsive design

### Base HTML Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{subject}}</title>
    <!--[if mso]>
    <noscript>
        <xml>
            <o:OfficeDocumentSettings>
                <o:PixelsPerInch>96</o:PixelsPerInch>
            </o:OfficeDocumentSettings>
        </xml>
    </noscript>
    <![endif]-->
    <style>
        /* Reset styles */
        body, table, td, a { -webkit-text-size-adjust: 100%; -ms-text-size-adjust: 100%; }
        table, td { mso-table-lspace: 0pt; mso-table-rspace: 0pt; }
        img { -ms-interpolation-mode: bicubic; border: 0; outline: none; }

        /* Base styles */
        body {
            margin: 0 !important;
            padding: 0 !important;
            background-color: #F9F5F0 !important;
            font-family: 'Source Sans Pro', Arial, sans-serif;
        }

        /* Container styles */
        .email-container {
            max-width: 600px;
            margin: 0 auto;
            background-color: #FDFCFA;
            border-radius: 8px;
            box-shadow: 0 4px 12px rgba(107, 68, 35, 0.15);
        }

        /* Header styles */
        .email-header {
            background-color: #6B4423;
            padding: 40px 30px;
            text-align: center;
            border-radius: 8px 8px 0 0;
        }

        .logo {
            font-family: 'Crimson Text', Georgia, serif;
            font-size: 32px;
            color: #FDFCFA;
            text-decoration: none;
            letter-spacing: 1px;
        }

        /* Content styles */
        .email-content {
            padding: 40px 30px;
            color: #6B4423;
            line-height: 1.8;
        }

        h1 {
            font-family: 'Crimson Text', Georgia, serif;
            font-size: 28px;
            color: #6B4423;
            margin: 0 0 20px;
            font-weight: 600;
        }

        h2 {
            font-family: 'Crimson Text', Georgia, serif;
            font-size: 24px;
            color: #6B4423;
            margin: 30px 0 15px;
            font-weight: 600;
        }

        p {
            margin: 0 0 20px;
            font-size: 18px;
            color: #6B4423;
        }

        /* Button styles */
        .button {
            display: inline-block;
            padding: 14px 32px;
            background-color: #D4A574;
            color: #FDFCFA !important;
            text-decoration: none;
            border-radius: 4px;
            font-weight: 600;
            font-size: 18px;
            margin: 20px 0;
            transition: background-color 0.3s ease;
        }

        .button:hover {
            background-color: #C49564;
        }

        .button-secondary {
            background-color: #E8E2DB;
            color: #6B4423 !important;
        }

        .button-secondary:hover {
            background-color: #D8D2CB;
        }

        /* Card styles */
        .card {
            background-color: #F9F5F0;
            border: 1px solid #E8E2DB;
            border-radius: 8px;
            padding: 20px;
            margin: 20px 0;
        }

        /* Footer styles */
        .email-footer {
            background-color: #F9F5F0;
            padding: 30px;
            text-align: center;
            border-radius: 0 0 8px 8px;
            font-size: 14px;
            color: #B8B2A7;
        }

        .email-footer a {
            color: #D4A574;
            text-decoration: none;
        }

        /* Responsive styles */
        @media screen and (max-width: 600px) {
            .email-container {
                width: 100% !important;
                border-radius: 0 !important;
            }
            .email-header {
                padding: 30px 20px !important;
            }
            .email-content {
                padding: 30px 20px !important;
            }
            h1 {
                font-size: 24px !important;
            }
            h2 {
                font-size: 20px !important;
            }
        }
    </style>
</head>
<body>
    <div class="email-container">
        <!-- Header -->
        <div class="email-header">
            <a href="{{app_url}}" class="logo">
                FamilyTales
            </a>
        </div>

        <!-- Content -->
        <div class="email-content">
            {{content}}
        </div>

        <!-- Footer -->
        <div class="email-footer">
            <p>© {{current_year}} FamilyTales. All rights reserved.</p>
            <p>
                <a href="{{preferences_url}}">Email Preferences</a> • 
                <a href="{{unsubscribe_url}}">Unsubscribe</a> • 
                <a href="mailto:{{support_email}}">Contact Support</a>
            </p>
            <p>1234 Memory Lane, San Francisco, CA 94102</p>
        </div>
    </div>
</body>
</html>
```

### 1. Family Invitation Email

**Template ID**: `d-family-invitation`

```html
<!-- Email Content -->
<h1>You're invited to join {{family_name}}!</h1>

<div style="text-align: center; margin: 30px 0;">
    <img src="{{inviter_avatar}}" alt="{{inviter_name}}" 
         style="width: 80px; height: 80px; border-radius: 50%; border: 3px solid #E8E2DB;">
</div>

<p>{{inviter_name}} has invited you to join their family's story collection on FamilyTales.</p>

{{#if personal_message}}
<div class="card">
    <p style="font-family: 'Amatic SC', cursive; font-size: 24px; margin: 0; color: #D4A574;">
        "{{personal_message}}"
    </p>
    <p style="text-align: right; margin: 10px 0 0; font-style: italic;">
        - {{inviter_name}}
    </p>
</div>
{{/if}}

<h2>What is FamilyTales?</h2>
<p>FamilyTales helps families preserve and share their precious memories through:</p>
<ul style="margin: 20px 0; padding-left: 20px;">
    <li style="margin: 10px 0;">📖 Digital memory books with photos and stories</li>
    <li style="margin: 10px 0;">🎙️ Audio recordings that capture voices and emotions</li>
    <li style="margin: 10px 0;">👨‍👩‍👧‍👦 Secure family sharing and collaboration</li>
    <li style="margin: 10px 0;">📚 Beautiful printed books of your memories</li>
</ul>

<p>As a <strong>{{role}}</strong> in {{family_name}}, you'll be able to:</p>
<ul style="margin: 20px 0; padding-left: 20px;">
    {{#if_eq role "editor"}}
    <li style="margin: 10px 0;">Add and edit stories and memories</li>
    <li style="margin: 10px 0;">Upload photos and recordings</li>
    <li style="margin: 10px 0;">Comment on family stories</li>
    {{/if_eq}}
    {{#if_eq role "viewer"}}
    <li style="margin: 10px 0;">View all family stories and photos</li>
    <li style="margin: 10px 0;">Listen to audio recordings</li>
    <li style="margin: 10px 0;">Add comments to stories</li>
    {{/if_eq}}
</ul>

<div style="text-align: center; margin: 40px 0;">
    <a href="{{accept_url}}" class="button">Accept Invitation</a>
    <br>
    <a href="{{decline_url}}" style="color: #B8B2A7; font-size: 14px;">Decline invitation</a>
</div>

<p style="font-size: 14px; color: #B8B2A7; margin-top: 40px;">
    This invitation will expire in 7 days. If you have any questions, 
    feel free to reply to this email or contact our support team.
</p>
```

### 2. Welcome Email for New Users

**Template ID**: `d-welcome-new-user`

```html
<!-- Email Content -->
<h1>Welcome to FamilyTales, {{user_name}}!</h1>

<p>We're delighted to have you join our community of families preserving their precious memories.</p>

<div class="card" style="background-color: #FFF9F5; border-color: #D4A574;">
    <h2 style="margin-top: 0;">🎉 Let's get you started!</h2>
    <p>First, please verify your email address to activate your account:</p>
    <div style="text-align: center;">
        <a href="{{verification_url}}" class="button">Verify Email Address</a>
    </div>
</div>

<h2>Three simple steps to begin:</h2>

<div style="margin: 30px 0;">
    <div style="display: flex; align-items: flex-start; margin: 20px 0;">
        <div style="background-color: #D4A574; color: #FDFCFA; width: 40px; height: 40px; 
                    border-radius: 50%; display: flex; align-items: center; justify-content: center; 
                    font-weight: bold; margin-right: 20px; flex-shrink: 0;">1</div>
        <div>
            <h3 style="margin: 0 0 10px;">Create Your First Memory Book</h3>
            <p style="margin: 0;">Start with a theme like "Family Vacations" or "Grandma's Recipes"</p>
        </div>
    </div>

    <div style="display: flex; align-items: flex-start; margin: 20px 0;">
        <div style="background-color: #D4A574; color: #FDFCFA; width: 40px; height: 40px; 
                    border-radius: 50%; display: flex; align-items: center; justify-content: center; 
                    font-weight: bold; margin-right: 20px; flex-shrink: 0;">2</div>
        <div>
            <h3 style="margin: 0 0 10px;">Add Your First Story</h3>
            <p style="margin: 0;">Write it down or record your voice - both are precious</p>
        </div>
    </div>

    <div style="display: flex; align-items: flex-start; margin: 20px 0;">
        <div style="background-color: #D4A574; color: #FDFCFA; width: 40px; height: 40px; 
                    border-radius: 50%; display: flex; align-items: center; justify-content: center; 
                    font-weight: bold; margin-right: 20px; flex-shrink: 0;">3</div>
        <div>
            <h3 style="margin: 0 0 10px;">Invite Family Members</h3>
            <p style="margin: 0;">Share the joy of preserving memories together</p>
        </div>
    </div>
</div>

<div style="text-align: center; margin: 40px 0;">
    <a href="{{getting_started_url}}" class="button-secondary">View Getting Started Guide</a>
</div>

<h2>Need help?</h2>
<p>We're here to support you every step of the way:</p>
<ul style="margin: 20px 0; padding-left: 20px;">
    <li style="margin: 10px 0;">📚 <a href="{{help_center_url}}" style="color: #D4A574;">Visit our Help Center</a></li>
    <li style="margin: 10px 0;">💬 Reply to this email with any questions</li>
    <li style="margin: 10px 0;">🎥 <a href="{{tutorial_url}}" style="color: #D4A574;">Watch our quick tutorial</a></li>
</ul>

<p style="font-style: italic; color: #B8B2A7; margin-top: 40px; text-align: center;">
    "Every family has a story to tell. We're honored to help you tell yours."
</p>
```

### 3. OCR Completion Notification

**Template ID**: `d-ocr-completion`

```html
<!-- Email Content -->
<h1>Your handwritten memories are now digital!</h1>

<div style="text-align: center; margin: 30px 0;">
    <div style="background-color: #A3B88C; color: #FDFCFA; width: 80px; height: 80px; 
                border-radius: 50%; display: flex; align-items: center; justify-content: center; 
                margin: 0 auto; font-size: 40px;">✓</div>
</div>

<p>Great news! We've finished converting your handwritten documents into searchable digital text.</p>

<div class="card">
    <h3 style="margin-top: 0;">📄 Document Details</h3>
    <p><strong>Document:</strong> {{document_name}}</p>
    <p><strong>Pages processed:</strong> {{page_count}}</p>
    <p><strong>Processing time:</strong> {{processing_time}}</p>
    <p><strong>Confidence score:</strong> {{confidence_score}}%</p>
</div>

{{#if preview_text}}
<div class="card" style="background-color: #FFF9F5;">
    <h3 style="margin-top: 0;">Preview of converted text:</h3>
    <p style="font-family: 'Amatic SC', cursive; font-size: 20px; color: #6B4423;">
        "{{preview_text}}..."
    </p>
</div>
{{/if}}

<h2>What's next?</h2>
<p>Now that your text is digitized, you can:</p>
<ul style="margin: 20px 0; padding-left: 20px;">
    <li style="margin: 10px 0;">✏️ Review and edit the converted text for accuracy</li>
    <li style="margin: 10px 0;">🔍 Search through your memories instantly</li>
    <li style="margin: 10px 0;">📖 Add these stories to your memory books</li>
    <li style="margin: 10px 0;">🎙️ Record audio narration to bring them to life</li>
</ul>

<div style="text-align: center; margin: 40px 0;">
    <a href="{{review_url}}" class="button">Review Converted Text</a>
</div>

{{#if tips}}
<div class="card" style="background-color: #F0F8FF; border-color: #87CEEB;">
    <h3 style="margin-top: 0;">💡 Tips for best results:</h3>
    <ul style="margin: 10px 0; padding-left: 20px;">
        <li style="margin: 10px 0;">Review names and dates carefully - these are most prone to errors</li>
        <li style="margin: 10px 0;">Add context or notes while the memories are fresh</li>
        <li style="margin: 10px 0;">Consider recording the story in your own voice</li>
    </ul>
</div>
{{/if}}
```

### 4. Memory Book Shared Notification

**Template ID**: `d-memory-book-shared`

```html
<!-- Email Content -->
<h1>{{sharer_name}} shared a memory book with you!</h1>

<div class="card" style="text-align: center; padding: 30px;">
    <img src="{{book_cover_image}}" alt="{{book_title}}" 
         style="width: 200px; height: 280px; border-radius: 8px; 
                box-shadow: 0 4px 12px rgba(107, 68, 35, 0.15);">
    <h2 style="margin: 20px 0 10px;">{{book_title}}</h2>
    <p style="color: #B8B2A7; margin: 0;">{{story_count}} stories • {{photo_count}} photos</p>
</div>

{{#if personal_message}}
<div class="card" style="background-color: #FFF9F5;">
    <p style="font-family: 'Amatic SC', cursive; font-size: 24px; margin: 0; color: #D4A574;">
        "{{personal_message}}"
    </p>
    <p style="text-align: right; margin: 10px 0 0; font-style: italic;">
        - {{sharer_name}}
    </p>
</div>
{{/if}}

<h2>What's inside:</h2>
<div style="margin: 30px 0;">
    {{#each featured_stories}}
    <div class="card" style="margin: 15px 0;">
        <h3 style="margin: 0 0 10px;">{{title}}</h3>
        <p style="margin: 0; color: #B8B2A7;">{{summary}}</p>
    </div>
    {{/each}}
</div>

<p>You now have <strong>{{access_level}}</strong> access to this memory book, which means you can:</p>
<ul style="margin: 20px 0; padding-left: 20px;">
    {{#if_eq access_level "view"}}
    <li style="margin: 10px 0;">📖 Read all stories and view photos</li>
    <li style="margin: 10px 0;">🎧 Listen to audio recordings</li>
    <li style="margin: 10px 0;">💬 Leave comments and reactions</li>
    {{/if_eq}}
    {{#if_eq access_level "contribute"}}
    <li style="margin: 10px 0;">📖 Read all stories and view photos</li>
    <li style="margin: 10px 0;">🎧 Listen to audio recordings</li>
    <li style="margin: 10px 0;">💬 Leave comments and reactions</li>
    <li style="margin: 10px 0;">➕ Add your own stories and photos</li>
    {{/if_eq}}
</ul>

<div style="text-align: center; margin: 40px 0;">
    <a href="{{book_url}}" class="button">View Memory Book</a>
</div>

{{#if sharing_expires}}
<p style="font-size: 14px; color: #B8B2A7; text-align: center;">
    This shared access expires on {{expiry_date}}
</p>
{{/if}}
```

### 5. Payment Receipt

**Template ID**: `d-payment-receipt`

```html
<!-- Email Content -->
<h1>Thank you for your payment!</h1>

<p>Your subscription to FamilyTales has been successfully processed.</p>

<div class="card" style="background-color: #F9F5F0;">
    <h2 style="margin-top: 0;">Receipt Details</h2>
    <table style="width: 100%; border-collapse: collapse;">
        <tr>
            <td style="padding: 10px 0; border-bottom: 1px solid #E8E2DB;">
                <strong>Receipt Number:</strong>
            </td>
            <td style="padding: 10px 0; border-bottom: 1px solid #E8E2DB; text-align: right;">
                {{receipt_number}}
            </td>
        </tr>
        <tr>
            <td style="padding: 10px 0; border-bottom: 1px solid #E8E2DB;">
                <strong>Date:</strong>
            </td>
            <td style="padding: 10px 0; border-bottom: 1px solid #E8E2DB; text-align: right;">
                {{payment_date}}
            </td>
        </tr>
        <tr>
            <td style="padding: 10px 0; border-bottom: 1px solid #E8E2DB;">
                <strong>Plan:</strong>
            </td>
            <td style="padding: 10px 0; border-bottom: 1px solid #E8E2DB; text-align: right;">
                {{plan_name}}
            </td>
        </tr>
        <tr>
            <td style="padding: 10px 0; border-bottom: 1px solid #E8E2DB;">
                <strong>Billing Period:</strong>
            </td>
            <td style="padding: 10px 0; border-bottom: 1px solid #E8E2DB; text-align: right;">
                {{billing_period}}
            </td>
        </tr>
        <tr>
            <td style="padding: 10px 0;">
                <strong>Amount Paid:</strong>
            </td>
            <td style="padding: 10px 0; text-align: right; font-size: 24px; color: #D4A574;">
                <strong>{{amount}}</strong>
            </td>
        </tr>
    </table>
</div>

<h2>Your {{plan_name}} includes:</h2>
<ul style="margin: 20px 0; padding-left: 20px;">
    {{#each plan_features}}
    <li style="margin: 10px 0;">✓ {{this}}</li>
    {{/each}}
</ul>

<div class="card" style="background-color: #FFF9F5; border-color: #D4A574;">
    <p style="margin: 0;">
        <strong>Next billing date:</strong> {{next_billing_date}}<br>
        <strong>Payment method:</strong> {{payment_method}}
    </p>
</div>

<div style="text-align: center; margin: 40px 0;">
    <a href="{{download_receipt_url}}" class="button-secondary">Download Receipt (PDF)</a>
</div>

<p style="font-size: 14px; color: #B8B2A7;">
    Need to update your billing information or change your plan? 
    <a href="{{billing_url}}" style="color: #D4A574;">Manage your subscription</a>
</p>
```

### 6. Gift Subscription Email

**Template ID**: `d-gift-subscription`

```html
<!-- Email Content -->
<h1>🎁 You've received a gift!</h1>

<div style="text-align: center; margin: 30px 0;">
    <div style="background-color: #D4A574; padding: 30px; border-radius: 8px; 
                display: inline-block; box-shadow: 0 4px 12px rgba(107, 68, 35, 0.15);">
        <p style="color: #FDFCFA; margin: 0; font-size: 20px;">
            {{gift_giver_name}} has gifted you
        </p>
        <p style="color: #FDFCFA; margin: 10px 0 0; font-size: 32px; font-weight: bold;">
            {{subscription_duration}} of FamilyTales Premium
        </p>
    </div>
</div>

{{#if gift_message}}
<div class="card" style="background-color: #FFF9F5;">
    <p style="font-family: 'Amatic SC', cursive; font-size: 24px; margin: 0; color: #D4A574;">
        "{{gift_message}}"
    </p>
    <p style="text-align: right; margin: 10px 0 0; font-style: italic;">
        - {{gift_giver_name}}
    </p>
</div>
{{/if}}

<h2>Your Premium subscription includes:</h2>
<div style="margin: 30px 0;">
    <div class="card">
        <h3 style="margin-top: 0;">📚 Unlimited Memory Books</h3>
        <p style="margin: 0;">Create as many books as your family needs</p>
    </div>
    <div class="card">
        <h3 style="margin-top: 0;">🎙️ Unlimited Audio Recording</h3>
        <p style="margin: 0;">Capture every voice, every story</p>
    </div>
    <div class="card">
        <h3 style="margin-top: 0;">📸 Unlimited Photo Storage</h3>
        <p style="margin: 0;">Never worry about running out of space</p>
    </div>
    <div class="card">
        <h3 style="margin-top: 0;">📖 Annual Printed Book</h3>
        <p style="margin: 0;">One beautiful hardcover book each year</p>
    </div>
</div>

<div style="text-align: center; margin: 40px 0;">
    {{#if existing_user}}
    <a href="{{activate_gift_url}}" class="button">Activate Your Gift</a>
    {{else}}
    <a href="{{create_account_url}}" class="button">Create Your Account</a>
    {{/if}}
</div>

<p style="text-align: center; color: #B8B2A7;">
    Your gift subscription will be active for {{subscription_duration}} from the date of activation.
</p>

{{#if_not existing_user}}
<div class="card" style="background-color: #F0F8FF; border-color: #87CEEB; margin-top: 40px;">
    <h3 style="margin-top: 0;">New to FamilyTales?</h3>
    <p>FamilyTales helps families preserve their precious memories through digital storytelling. 
    With your gift subscription, you can start capturing and sharing your family's unique story today!</p>
</div>
{{/if_not}}
```

## Implementation Details

### API Integration with SendGrid

#### Send Function with Retry Logic

```go
func (s *EmailService) SendTemplatedEmail(
    to string, 
    templateID string, 
    data map[string]interface{},
) error {
    message := mail.NewV3Mail()
    
    from := mail.NewEmail(s.config.FromName, s.config.FromEmail)
    message.SetFrom(from)
    message.SetReplyTo(mail.NewEmail("Support", s.config.ReplyTo))
    
    personalization := mail.NewPersonalization()
    personalization.AddTos(mail.NewEmail("", to))
    
    // Add base template data
    baseData := map[string]interface{}{
        "app_name":         "FamilyTales",
        "app_url":          "https://app.familytales.app",
        "current_year":     time.Now().Year(),
        "support_email":    s.config.ReplyTo,
        "preferences_url":  fmt.Sprintf("https://app.familytales.app/preferences?email=%s", to),
        "unsubscribe_url":  "{{unsubscribe}}",
    }
    
    // Merge base data with provided data
    for key, value := range baseData {
        personalization.SetDynamicTemplateData(key, value)
    }
    for key, value := range data {
        personalization.SetDynamicTemplateData(key, value)
    }
    
    message.AddPersonalizations(personalization)
    message.SetTemplateID(templateID)
    
    // Add tracking settings
    trackingSettings := mail.NewTrackingSettings()
    clickTracking := mail.NewClickTracking()
    clickTracking.SetEnable(true)
    clickTracking.SetEnableText(true)
    trackingSettings.SetClickTracking(clickTracking)
    
    openTracking := mail.NewOpenTracking()
    openTracking.SetEnable(true)
    trackingSettings.SetOpenTracking(openTracking)
    
    message.SetTrackingSettings(trackingSettings)
    
    // Add custom headers
    message.SetHeader("X-Entity-Ref-ID", generateRefID())
    
    // Retry logic
    maxRetries := 3
    for i := 0; i < maxRetries; i++ {
        response, err := s.client.Send(message)
        if err == nil && response.StatusCode < 400 {
            return nil
        }
        
        if i < maxRetries-1 {
            backoff := time.Duration(math.Pow(2, float64(i))) * time.Second
            time.Sleep(backoff)
        } else {
            return fmt.Errorf("failed after %d retries: %w", maxRetries, err)
        }
    }
    
    return nil
}
```

### Template Management

#### Dynamic Template Creation

```go
type TemplateManager struct {
    templates map[string]EmailTemplate
}

type EmailTemplate struct {
    ID          string
    Name        string
    Subject     string
    Variables   []string
    TestData    map[string]interface{}
}

func (tm *TemplateManager) ValidateTemplateData(
    templateID string, 
    data map[string]interface{},
) error {
    template, exists := tm.templates[templateID]
    if !exists {
        return fmt.Errorf("template %s not found", templateID)
    }
    
    // Check required variables
    for _, variable := range template.Variables {
        if _, exists := data[variable]; !exists {
            return fmt.Errorf("missing required variable: %s", variable)
        }
    }
    
    return nil
}
```

### Dynamic Content Insertion

#### Personalization Engine

```go
type PersonalizationEngine struct {
    userService     UserService
    familyService   FamilyService
    preferenceCache *cache.Cache
}

func (pe *PersonalizationEngine) PersonalizeEmail(
    userID string,
    baseData map[string]interface{},
) map[string]interface{} {
    user, _ := pe.userService.GetUser(userID)
    preferences, _ := pe.getPreferences(userID)
    
    // Add user-specific data
    data := make(map[string]interface{})
    for k, v := range baseData {
        data[k] = v
    }
    
    data["user_name"] = user.DisplayName
    data["first_name"] = user.FirstName
    data["timezone"] = preferences.Timezone
    data["locale"] = preferences.Locale
    
    // Add time-based greetings
    greeting := pe.getTimeBasedGreeting(preferences.Timezone)
    data["greeting"] = greeting
    
    return data
}

func (pe *PersonalizationEngine) getTimeBasedGreeting(timezone string) string {
    loc, _ := time.LoadLocation(timezone)
    hour := time.Now().In(loc).Hour()
    
    switch {
    case hour < 12:
        return "Good morning"
    case hour < 17:
        return "Good afternoon"
    default:
        return "Good evening"
    }
}
```

### Unsubscribe Handling

#### Unsubscribe Service

```go
type UnsubscribeService struct {
    db              *sql.DB
    emailService    *EmailService
    preferenceCache *cache.Cache
}

func (us *UnsubscribeService) HandleUnsubscribe(token string) error {
    // Validate token
    userID, err := us.validateUnsubscribeToken(token)
    if err != nil {
        return err
    }
    
    // Update preferences
    _, err = us.db.Exec(`
        UPDATE email_preferences 
        SET all_emails_unsubscribed = true,
            updated_at = CURRENT_TIMESTAMP
        WHERE user_id = $1
    `, userID)
    
    if err != nil {
        return err
    }
    
    // Clear cache
    us.preferenceCache.Delete(userID)
    
    // Log unsubscribe event
    us.logUnsubscribeEvent(userID, "user_initiated")
    
    return nil
}

func (us *UnsubscribeService) HandleGranularUnsubscribe(
    userID string,
    preferences EmailPreferences,
) error {
    // Update specific preferences
    _, err := us.db.Exec(`
        UPDATE email_preferences 
        SET new_story_notifications = $1,
            comment_notifications = $2,
            mention_notifications = $3,
            weekly_digest = $4,
            monthly_summary = $5,
            birthday_reminders = $6,
            anniversary_reminders = $7,
            product_updates = $8,
            tips_and_tricks = $9,
            notification_frequency = $10,
            updated_at = CURRENT_TIMESTAMP
        WHERE user_id = $11
    `, 
        preferences.NewStoryNotifications,
        preferences.CommentNotifications,
        preferences.MentionNotifications,
        preferences.WeeklyDigest,
        preferences.MonthlySummary,
        preferences.BirthdayReminders,
        preferences.AnniversaryReminders,
        preferences.ProductUpdates,
        preferences.TipsAndTricks,
        preferences.NotificationFrequency,
        userID,
    )
    
    return err
}
```

### Bounce and Complaint Handling

#### Webhook Handler

```go
func (s *EmailService) HandleSendGridWebhook(w http.ResponseWriter, r *http.Request) {
    // Verify webhook signature
    signature := r.Header.Get("X-Twilio-Email-Event-Webhook-Signature")
    timestamp := r.Header.Get("X-Twilio-Email-Event-Webhook-Timestamp")
    
    if !s.verifyWebhookSignature(r.Body, signature, timestamp) {
        http.Error(w, "Invalid signature", http.StatusUnauthorized)
        return
    }
    
    var events []SendGridEvent
    if err := json.NewDecoder(r.Body).Decode(&events); err != nil {
        http.Error(w, "Invalid payload", http.StatusBadRequest)
        return
    }
    
    for _, event := range events {
        switch event.Event {
        case "bounce":
            s.handleBounce(event)
        case "dropped":
            s.handleDropped(event)
        case "spam_report":
            s.handleSpamReport(event)
        case "unsubscribe":
            s.handleUnsubscribe(event)
        case "open":
            s.trackOpen(event)
        case "click":
            s.trackClick(event)
        }
    }
    
    w.WriteHeader(http.StatusOK)
}

func (s *EmailService) handleBounce(event SendGridEvent) {
    bounceType := "soft"
    if event.Type == "bounce" && event.BounceType == "hard" {
        bounceType = "hard"
    }
    
    // Record bounce
    _, err := s.db.Exec(`
        INSERT INTO email_bounces (email_address, bounce_type, reason, sendgrid_message_id)
        VALUES ($1, $2, $3, $4)
        ON CONFLICT (email_address) DO UPDATE
        SET bounce_count = email_bounces.bounce_count + 1,
            last_bounce_at = CURRENT_TIMESTAMP,
            reason = EXCLUDED.reason
    `, event.Email, bounceType, event.Reason, event.MessageID)
    
    if err != nil {
        log.Error("Failed to record bounce", err)
        return
    }
    
    // Check if we should suppress
    var bounceCount int
    s.db.QueryRow(`
        SELECT bounce_count FROM email_bounces WHERE email_address = $1
    `, event.Email).Scan(&bounceCount)
    
    if bounceType == "hard" || bounceCount >= 3 {
        s.suppressEmail(event.Email, "bounce")
    }
}

func (s *EmailService) handleSpamReport(event SendGridEvent) {
    // Immediately suppress
    s.suppressEmail(event.Email, "spam_report")
    
    // Log spam report
    _, err := s.db.Exec(`
        INSERT INTO spam_reports (email_address, reported_at, sendgrid_message_id)
        VALUES ($1, $2, $3)
    `, event.Email, time.Now(), event.MessageID)
    
    if err != nil {
        log.Error("Failed to record spam report", err)
    }
    
    // Alert if spam rate is high
    s.checkSpamRate()
}

func (s *EmailService) suppressEmail(email string, reason string) {
    _, err := s.db.Exec(`
        INSERT INTO email_suppressions (email_address, suppression_type, reason)
        VALUES ($1, $2, $3)
        ON CONFLICT (email_address) DO UPDATE
        SET suppression_type = EXCLUDED.suppression_type,
            reason = EXCLUDED.reason,
            suppressed_at = CURRENT_TIMESTAMP
    `, email, reason, reason)
    
    if err != nil {
        log.Error("Failed to suppress email", err)
    }
}
```

## Email Analytics and Tracking

### Metrics Collection

```go
type EmailMetrics struct {
    MessageID       string
    TemplateID      string
    RecipientEmail  string
    RecipientUserID string
    SentAt          time.Time
    DeliveredAt     *time.Time
    OpenedAt        *time.Time
    OpenCount       int
    ClickedAt       *time.Time
    ClickCount      int
    ClickedLinks    []ClickedLink
}

type ClickedLink struct {
    URL       string
    ClickedAt time.Time
    UserAgent string
    IPAddress string
}

func (s *EmailService) trackEmailMetrics(event SendGridEvent) {
    switch event.Event {
    case "delivered":
        s.updateMetric(event.MessageID, "delivered_at", time.Now())
    case "open":
        s.incrementMetric(event.MessageID, "open_count")
        s.updateMetric(event.MessageID, "opened_at", time.Now())
    case "click":
        s.incrementMetric(event.MessageID, "click_count")
        s.updateMetric(event.MessageID, "clicked_at", time.Now())
        s.trackClickedLink(event)
    }
}
```

### Analytics Dashboard Queries

```sql
-- Email performance by template
SELECT 
    template_id,
    COUNT(*) as sent_count,
    COUNT(delivered_at) as delivered_count,
    COUNT(opened_at) as opened_count,
    COUNT(clicked_at) as clicked_count,
    AVG(CASE WHEN delivered_at IS NOT NULL THEN 1 ELSE 0 END) as delivery_rate,
    AVG(CASE WHEN opened_at IS NOT NULL THEN 1 ELSE 0 END) as open_rate,
    AVG(CASE WHEN clicked_at IS NOT NULL THEN 1 ELSE 0 END) as click_rate
FROM email_metrics
WHERE sent_at >= NOW() - INTERVAL '30 days'
GROUP BY template_id;

-- Best send times
SELECT 
    EXTRACT(hour FROM sent_at) as hour,
    EXTRACT(dow FROM sent_at) as day_of_week,
    AVG(CASE WHEN opened_at IS NOT NULL THEN 1 ELSE 0 END) as open_rate
FROM email_metrics
WHERE sent_at >= NOW() - INTERVAL '90 days'
GROUP BY hour, day_of_week
ORDER BY open_rate DESC;

-- User engagement segments
SELECT 
    recipient_user_id,
    COUNT(*) as emails_received,
    AVG(CASE WHEN opened_at IS NOT NULL THEN 1 ELSE 0 END) as avg_open_rate,
    AVG(CASE WHEN clicked_at IS NOT NULL THEN 1 ELSE 0 END) as avg_click_rate,
    CASE 
        WHEN AVG(CASE WHEN opened_at IS NOT NULL THEN 1 ELSE 0 END) > 0.5 THEN 'Highly Engaged'
        WHEN AVG(CASE WHEN opened_at IS NOT NULL THEN 1 ELSE 0 END) > 0.2 THEN 'Moderately Engaged'
        ELSE 'Low Engagement'
    END as engagement_segment
FROM email_metrics
WHERE sent_at >= NOW() - INTERVAL '30 days'
GROUP BY recipient_user_id;
```

## Best Practices

### 1. Deliverability Best Practices

- **Authentication**: Ensure SPF, DKIM, and DMARC are properly configured
- **Warm-up**: Gradually increase sending volume for new IPs
- **List Hygiene**: Regularly clean email lists and remove bounces
- **Engagement**: Monitor and improve open/click rates
- **Content**: Avoid spam trigger words and maintain good text/image ratio

### 2. Performance Optimization

```go
// Batch sending for large campaigns
func (s *EmailService) SendBatchEmails(recipients []EmailRecipient, templateID string) error {
    const batchSize = 1000 // SendGrid limit
    
    // Use worker pool for parallel processing
    workerPool := make(chan []EmailRecipient, 10)
    errors := make(chan error, 10)
    
    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for batch := range workerPool {
                if err := s.sendBatch(batch, templateID); err != nil {
                    errors <- err
                }
            }
        }()
    }
    
    // Send batches to workers
    for i := 0; i < len(recipients); i += batchSize {
        end := i + batchSize
        if end > len(recipients) {
            end = len(recipients)
        }
        workerPool <- recipients[i:end]
    }
    
    close(workerPool)
    wg.Wait()
    close(errors)
    
    // Check for errors
    for err := range errors {
        if err != nil {
            return err
        }
    }
    
    return nil
}
```

### 3. Security Considerations

- **Token Security**: Use cryptographically secure tokens for all email links
- **Rate Limiting**: Implement per-user and IP-based rate limits
- **Data Protection**: Never log sensitive email content
- **HTTPS Only**: All email links must use HTTPS
- **Header Injection**: Sanitize all user input in email content

### 4. Testing Strategy

```go
// Email testing framework
type EmailTester struct {
    mockClient *MockSendGridClient
    templates  map[string]EmailTemplate
}

func (et *EmailTester) TestEmailTemplate(templateID string, testData map[string]interface{}) error {
    // Validate required fields
    template := et.templates[templateID]
    for _, field := range template.RequiredFields {
        if _, exists := testData[field]; !exists {
            return fmt.Errorf("missing required field: %s", field)
        }
    }
    
    // Test rendering
    rendered, err := et.renderTemplate(templateID, testData)
    if err != nil {
        return fmt.Errorf("template rendering failed: %w", err)
    }
    
    // Validate HTML
    if err := et.validateHTML(rendered); err != nil {
        return fmt.Errorf("invalid HTML: %w", err)
    }
    
    // Check for broken links
    if err := et.checkLinks(rendered); err != nil {
        return fmt.Errorf("broken links found: %w", err)
    }
    
    // Test responsiveness
    if err := et.testResponsiveness(rendered); err != nil {
        return fmt.Errorf("responsiveness issues: %w", err)
    }
    
    return nil
}
```

### 5. Monitoring and Alerting

```yaml
# Prometheus alerts for email service
groups:
  - name: email_alerts
    rules:
      - alert: HighBounceRate
        expr: rate(email_bounces_total[1h]) > 0.05
        annotations:
          summary: "High email bounce rate detected"
          
      - alert: SendGridAPIErrors
        expr: rate(sendgrid_api_errors_total[5m]) > 0.01
        annotations:
          summary: "SendGrid API errors increasing"
          
      - alert: EmailQueueBacklog
        expr: email_queue_size > 10000
        annotations:
          summary: "Email queue backlog detected"
          
      - alert: LowOpenRate
        expr: avg_over_time(email_open_rate[24h]) < 0.15
        annotations:
          summary: "Email open rates below threshold"
```

This comprehensive email integration specification provides a complete framework for implementing a robust, scalable, and user-friendly email system for FamilyTales.