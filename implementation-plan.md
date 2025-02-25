# Implementation Plan

This document outlines the step-by-step plan to fix the image upload and checkout flow issues in the LivePhoto25 project.

## Phase 1: Fix Pre-Checkout File Storage

### Step 1: Update Products Page
- Modify `src/app/products/page.tsx` to implement the async file storage solution
- Add loading states and better error handling
- Test with various file sizes and types

### Step 2: Add File Storage Utilities
- Create a new utility file `src/lib/storage/client-storage.ts`
- Implement functions for storing and retrieving files from sessionStorage
- Add validation and error handling

## Phase 2: Improve Success Page

### Step 1: Update Success Page
- Modify `src/app/payment/success/page.tsx` to implement exponential backoff
- Improve loading states and user feedback
- Enhance error handling

### Step 2: Add Order Status Tracking
- Update the order status tracking in the database
- Add more detailed status information
- Implement better error recovery

## Phase 3: Testing and Validation

### Step 1: Test Complete Flow
- Test the entire checkout flow from product selection to upload
- Verify files are properly stored and retrieved
- Test with various network conditions

### Step 2: Edge Case Testing
- Test with maximum file sizes
- Test with slow network connections
- Test with browser refreshes during the process

## Phase 4: Monitoring and Logging

### Step 1: Add Logging
- Implement detailed logging throughout the process
- Track success rates and failure points
- Set up error reporting

### Step 2: Add Analytics
- Track conversion rates through the checkout process
- Identify drop-off points
- Measure performance metrics

## Implementation Details

### File Storage Utility

```typescript
// src/lib/storage/client-storage.ts

/**
 * Store a file in sessionStorage as base64
 */
export async function storeFileInSession(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    
    reader.onload = () => {
      try {
        const base64Data = reader.result as string;
        const fileId = `${Date.now()}_${Math.random().toString(36).substring(7)}`;
        sessionStorage.setItem(`file_${fileId}`, base64Data);
        resolve(fileId);
      } catch (err) {
        reject(err);
      }
    };
    
    reader.onerror = () => {
      reject(new Error(`Failed to read file: ${file.name}`));
    };
    
    reader.readAsDataURL(file);
  });
}

/**
 * Store multiple files in sessionStorage
 */
export async function storeFilesInSession(files: File[]): Promise<string[]> {
  try {
    const fileIds = await Promise.all(files.map(file => storeFileInSession(file)));
    sessionStorage.setItem('pendingUploadFiles', JSON.stringify(fileIds));
    return fileIds;
  } catch (error) {
    console.error('Error storing files in session:', error);
    throw error;
  }
}

/**
 * Retrieve a file from sessionStorage
 */
export async function getFileFromSession(fileId: string): Promise<File | null> {
  try {
    const fileData = sessionStorage.getItem(`file_${fileId}`);
    if (!fileData) {
      return null;
    }

    // Handle both formats: with or without data URL prefix
    let base64Data = fileData;
    let contentType = 'image/jpeg'; // Default
    
    if (fileData.includes(',')) {
      const parts = fileData.split(',');
      // Extract content type if available
      if (parts[0].includes(':') && parts[0].includes(';')) {
        contentType = parts[0].split(':')[1].split(';')[0];
      }
      base64Data = parts[1];
    }
    
    const byteCharacters = atob(base64Data);
    const byteNumbers = new Array(byteCharacters.length);
    
    for (let i = 0; i < byteCharacters.length; i++) {
      byteNumbers[i] = byteCharacters.charCodeAt(i);
    }
    
    const byteArray = new Uint8Array(byteNumbers);
    const blob = new Blob([byteArray], { type: contentType });

    // Create a File object
    return new File([blob], `file_${fileId}`, { type: contentType });
  } catch (error) {
    console.error(`Error retrieving file ${fileId}:`, error);
    return null;
  }
}

/**
 * Retrieve all stored files from sessionStorage
 */
export async function getAllFilesFromSession(): Promise<File[]> {
  try {
    const storedFileIds = sessionStorage.getItem('pendingUploadFiles');
    if (!storedFileIds) {
      return [];
    }
    
    const fileIds = JSON.parse(storedFileIds) as string[];
    const filePromises = fileIds.map(id => getFileFromSession(id));
    const files = await Promise.all(filePromises);
    
    // Filter out any null results
    return files.filter((file): file is File => file !== null);
  } catch (error) {
    console.error('Error retrieving files from session:', error);
    return [];
  }
}

/**
 * Clear all stored files from sessionStorage
 */
export function clearFilesFromSession(): void {
  try {
    const storedFileIds = sessionStorage.getItem('pendingUploadFiles');
    if (storedFileIds) {
      const fileIds = JSON.parse(storedFileIds) as string[];
      fileIds.forEach(id => {
        sessionStorage.removeItem(`file_${id}`);
      });
    }
    sessionStorage.removeItem('pendingUploadFiles');
  } catch (error) {
    console.error('Error clearing files from session:', error);
  }
}
```

## Testing Plan

1. **Unit Tests**
   - Test file storage utilities
   - Test order retrieval functions
   - Test file conversion functions

2. **Integration Tests**
   - Test the complete checkout flow
   - Test webhook handling
   - Test file upload process

3. **End-to-End Tests**
   - Test the entire user journey
   - Test with real Stripe test cards
   - Test with various file types and sizes

## Rollout Strategy

1. **Development Environment**
   - Implement changes in development
   - Test thoroughly with various scenarios

2. **Staging Environment**
   - Deploy to staging
   - Perform end-to-end testing
   - Verify with real Stripe test transactions

3. **Production Rollout**
   - Deploy changes to production
   - Monitor closely for any issues
   - Have rollback plan ready if needed

## Success Metrics

1. **Reduction in Failed Uploads**
   - Target: 99% successful uploads after payment

2. **Improved User Experience**
   - Target: Clear progress indicators throughout the process
   - Target: Meaningful error messages when issues occur

3. **Performance**
   - Target: Complete upload process within 30 seconds for typical usage