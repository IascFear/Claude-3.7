# LivePhoto25 Project Fixes

This repository contains fixes and improvements for the LivePhoto25 project, which is a web application for converting images to videos.

## Project Overview

LivePhoto25 is built with:
- Next.js with TypeScript
- Tailwind CSS for styling
- Supabase for authentication and database
- Stripe for payments
- shadcn/ui for UI components

## Issues and Fixes

### 1. Image Upload and Checkout Flow

**Issue**: Files are stored in sessionStorage asynchronously before checkout, but the payment process might start before all files are properly saved.

**Fix**: Implement proper async/await pattern to ensure all files are stored before proceeding to checkout.

## Checkout Flow Documentation

1. **Product Selection and Image Upload** (products/page.tsx)
   - User selects package (images, processing, video length)
   - User uploads images via TestUpload component
   - "Proceed to Payment" button activates when all images are uploaded

2. **Pre-Payment Processing** (products/page.tsx)
   - Files stored in sessionStorage as base64 strings
   - File IDs stored in 'pendingUploadFiles'
   - Checkout session created via Stripe API

3. **Stripe Checkout Session Creation** (api/stripe/checkout/route.ts)
   - Server creates Stripe checkout session with product details
   - User redirected to Stripe's checkout page

4. **Payment Processing**
   - User completes payment on Stripe
   - Stripe redirects to success URL with session_id

5. **Webhook Processing** (api/stripe/webhook/route.ts)
   - Stripe sends webhook for successful payment
   - Server creates order record in database

6. **Success Page Processing** (payment/success/page.tsx)
   - Page waits 5 seconds for webhook processing
   - Retrieves order using session_id
   - Retrieves files from sessionStorage
   - Converts base64 strings to File objects
   - Uploads files to Supabase storage

7. **File Upload Processing** (api/storage/upload/route.ts)
   - Server validates and uploads files to Supabase
   - Returns paths to uploaded files

8. **Post-Upload Processing** (payment/success/page.tsx)
   - Updates order status to 'processing'
   - Redirects user to dashboard