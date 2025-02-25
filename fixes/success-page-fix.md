# Success Page Improvements

## Issue

The current implementation in `payment/success/page.tsx` has several issues:

1. It uses fixed timeouts (5 seconds) to wait for the webhook to process, which may not be sufficient
2. The error handling is minimal, with limited retry logic
3. The base64 to File conversion process could be more robust

## Current Implementation (Problematic)

```typescript
// Current implementation in payment/success/page.tsx
useEffect(() => {
  const handleUpload = async () => {
    if (!sessionId) {
      toast.error('No session ID found');
      router.push('/dashboard');
      return;
    }

    try {
      // Wait longer to ensure the webhook has processed
      await new Promise(resolve => setTimeout(resolve, 5000));

      console.log('Looking for order with session ID:', sessionId);
      
      let order = await getOrder(sessionId);

      if (!order) {
        console.error('Order not found, retrying...');
        
        // Try again after an even longer delay
        await new Promise(resolve => setTimeout(resolve, 5000));
        
        order = await getOrder(sessionId);
        
        if (!order) {
          toast.error('Could not find order details');
          router.push('/dashboard');
          return;
        }
      }

      // Get stored files from sessionStorage
      const storedFiles = sessionStorage.getItem('pendingUploadFiles');
      if (!storedFiles) {
        toast.error('No files found to upload');
        router.push('/dashboard');
        return;
      }

      const fileIds = JSON.parse(storedFiles) as string[];

      // Convert all files to File objects first
      const files: File[] = await Promise.all(fileIds.map(async (fileId) => {
        const fileData = sessionStorage.getItem(`file_${fileId}`);
        if (!fileData) {
          throw new Error(`File data not found for ${fileId}`);
        }

        // Convert base64 to Blob
        const base64Data = fileData.split(',')[1];
        const byteCharacters = atob(base64Data);
        const byteNumbers = new Array(byteCharacters.length);
        
        for (let i = 0; i < byteCharacters.length; i++) {
          byteNumbers[i] = byteCharacters.charCodeAt(i);
        }
        
        const byteArray = new Uint8Array(byteNumbers);
        const blob = new Blob([byteArray], { type: 'image/jpeg' });

        // Create a File object
        return new File([blob], `${fileId}.jpg`, { type: 'image/jpeg' });
      }));

      // Upload all files in one call
      const result = await uploadOrderImages({
        orderId: order.id,
        customerEmail: order.customer_email,
        files,
        onProgress: (progress) => {
          console.log(`Upload progress: ${progress}%`);
        }
      });

      if (!result.success) {
        throw new Error(result.error || 'Upload failed');
      }

      // Clear the sessionStorage
      fileIds.forEach(fileId => {
        sessionStorage.removeItem(`file_${fileId}`);
      });
      sessionStorage.removeItem('pendingUploadFiles');

      toast.success('Files uploaded successfully');
      
      // Update order status
      const supabase = createBrowserClient(
        process.env.NEXT_PUBLIC_SUPABASE_URL!,
        process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
      );
      const { error: updateError } = await supabase
        .from('orders')
        .update({ status: 'processing' })
        .eq('id', order.id);

      if (updateError) {
        console.error('Error updating order status:', updateError);
      }

      router.push('/dashboard');
    } catch (error) {
      console.error('Error in handleUpload:', error);
      toast.error('Error processing your order');
      router.push('/dashboard');
    }
  };

  handleUpload();
}, [sessionId, router]);
```

## Improved Implementation

```typescript
// Improved implementation for payment/success/page.tsx
useEffect(() => {
  const handleUpload = async () => {
    if (!sessionId) {
      toast.error('No session ID found');
      router.push('/dashboard');
      return;
    }

    // Create a toast ID for updating the loading state
    const loadingToastId = toast.loading('Processing your payment...');
    
    try {
      // Poll for the order with exponential backoff
      let order = null;
      let attempts = 0;
      const maxAttempts = 5;
      const baseDelay = 2000; // Start with 2 seconds
      
      while (!order && attempts < maxAttempts) {
        attempts++;
        const delay = baseDelay * Math.pow(1.5, attempts - 1); // Exponential backoff
        
        toast.loading(`Waiting for payment confirmation... (Attempt ${attempts}/${maxAttempts})`, {
          id: loadingToastId
        });
        
        // Wait before trying again
        await new Promise(resolve => setTimeout(resolve, delay));
        
        // Try to get the order
        order = await getOrder(sessionId);
        
        if (order) {
          break;
        }
        
        console.log(`Order not found, retrying in ${delay/1000} seconds...`);
      }
      
      if (!order) {
        throw new Error('Could not find order details after multiple attempts');
      }
      
      toast.loading('Order confirmed! Preparing to upload files...', {
        id: loadingToastId
      });
      
      // Get stored files from sessionStorage
      const storedFiles = sessionStorage.getItem('pendingUploadFiles');
      if (!storedFiles) {
        throw new Error('No files found to upload');
      }

      const fileIds = JSON.parse(storedFiles) as string[];
      
      if (fileIds.length === 0) {
        throw new Error('No file IDs found in storage');
      }
      
      toast.loading(`Preparing ${fileIds.length} files for upload...`, {
        id: loadingToastId
      });

      // Convert all files to File objects first
      const files: File[] = [];
      
      for (const fileId of fileIds) {
        const fileData = sessionStorage.getItem(`file_${fileId}`);
        if (!fileData) {
          throw new Error(`File data not found for ${fileId}`);
        }

        try {
          // Handle both formats: with or without data URL prefix
          let base64Data = fileData;
          if (fileData.includes(',')) {
            base64Data = fileData.split(',')[1];
          }
          
          const byteCharacters = atob(base64Data);
          const byteNumbers = new Array(byteCharacters.length);
          
          for (let i = 0; i < byteCharacters.length; i++) {
            byteNumbers[i] = byteCharacters.charCodeAt(i);
          }
          
          const byteArray = new Uint8Array(byteNumbers);
          const blob = new Blob([byteArray], { type: 'image/jpeg' });

          // Create a File object with a more unique name
          files.push(new File([blob], `order_${order.id}_${fileId}.jpg`, { type: 'image/jpeg' }));
        } catch (error) {
          console.error(`Error processing file ${fileId}:`, error);
          throw new Error(`Failed to process file ${fileId}`);
        }
      }
      
      if (files.length === 0) {
        throw new Error('Failed to process any files');
      }

      toast.loading(`Uploading ${files.length} files...`, {
        id: loadingToastId
      });
      
      // Upload all files in one call
      const result = await uploadOrderImages({
        orderId: order.id,
        customerEmail: order.customer_email,
        files,
        onProgress: (progress) => {
          toast.loading(`Uploading: ${progress}% complete`, {
            id: loadingToastId
          });
        }
      });

      if (!result.success) {
        throw new Error(result.error || 'Upload failed');
      }

      // Clear the sessionStorage
      fileIds.forEach(fileId => {
        sessionStorage.removeItem(`file_${fileId}`);
      });
      sessionStorage.removeItem('pendingUploadFiles');

      toast.success('Files uploaded successfully!', {
        id: loadingToastId
      });
      
      // Update order status
      const supabase = createBrowserClient(
        process.env.NEXT_PUBLIC_SUPABASE_URL!,
        process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
      );
      
      const { error: updateError } = await supabase
        .from('orders')
        .update({ status: 'processing' })
        .eq('id', order.id);

      if (updateError) {
        console.error('Error updating order status:', updateError);
        // Continue anyway since files were uploaded successfully
      }

      // Short delay so user can see success message
      await new Promise(resolve => setTimeout(resolve, 1500));
      router.push('/dashboard');
    } catch (error) {
      console.error('Error in handleUpload:', error);
      toast.error(error instanceof Error ? error.message : 'Error processing your order', {
        id: loadingToastId
      });
      
      // Give user time to read the error
      await new Promise(resolve => setTimeout(resolve, 3000));
      router.push('/dashboard');
    }
  };

  handleUpload();
}, [sessionId, router]);
```

## Changes Made

1. **Exponential Backoff**: Replaced fixed timeouts with exponential backoff for polling the order status.
2. **Improved Loading States**: Added detailed loading states to keep the user informed of progress.
3. **Better Error Handling**: Enhanced error handling with specific error messages.
4. **Robust File Processing**: Improved the file conversion process to handle different base64 formats.
5. **Unique File Names**: Added order ID to file names to ensure uniqueness.
6. **Progress Updates**: Added visual progress updates during file upload.

## Additional Recommendations

1. **Webhook Confirmation**: Consider implementing a direct webhook confirmation mechanism instead of polling.
2. **Resumable Uploads**: For larger files, implement resumable uploads to handle network interruptions.
3. **Fallback Mechanism**: Add a manual retry button in case the automatic process fails.
4. **Order Status Tracking**: Implement a more detailed order status tracking system.