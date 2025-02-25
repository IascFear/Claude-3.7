# Image Upload Fix

## Issue

The current implementation in `products/page.tsx` stores files in sessionStorage using an asynchronous FileReader without waiting for the operations to complete. This means the checkout process might proceed before all files are properly saved, leading to missing files when trying to upload them after payment.

## Current Implementation (Problematic)

```typescript
// Current implementation in products/page.tsx
const handleProceedToPayment = useCallback(async () => {
  if (files.length !== selectedPackage.imageCount.count) {
    toast.error(`Please upload exactly ${selectedPackage.imageCount.count} image${selectedPackage.imageCount.count > 1 ? 's' : ''}`);
    return;
  }

  try {
    // Store files in sessionStorage before redirecting
    const fileIds = files.map(file => file.name);
    sessionStorage.setItem('pendingUploadFiles', JSON.stringify(fileIds));
    
    // Store each file in sessionStorage
    for (const file of files) {
      const reader = new FileReader();
      reader.readAsDataURL(file);
      reader.onload = () => {
        const base64Data = (reader.result as string).split(',')[1];
        sessionStorage.setItem(`file_${file.name}`, base64Data);
      };
    }

    const session = await createCheckoutSession({
      product: {
        id: selectedPackage.product.id,
        name: selectedPackage.product.name,
        price: selectedPackage.imageCount.price,
        imageCount: selectedPackage.imageCount.count
      },
      processing: selectedPackage.processing,
      videoLength: selectedPackage.videoLength
    });

    if (session?.url) {
      router.push(session.url);
    }
  } catch (error) {
    console.error("Error creating checkout session:", error);
    toast.error("Failed to create checkout session");
  }
}, [files, selectedPackage, router]);
```

## Fixed Implementation

```typescript
// Fixed implementation for products/page.tsx
const handleProceedToPayment = useCallback(async () => {
  if (files.length !== selectedPackage.imageCount.count) {
    toast.error(`Please upload exactly ${selectedPackage.imageCount.count} image${selectedPackage.imageCount.count > 1 ? 's' : ''}`);
    return;
  }

  try {
    // Show loading state
    toast.loading("Preparing your files for checkout...");
    
    // Store files in sessionStorage before redirecting
    const fileIds = files.map(file => file.name);
    sessionStorage.setItem('pendingUploadFiles', JSON.stringify(fileIds));
    
    // Store each file in sessionStorage using promises
    const storeFilePromises = files.map(file => {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        
        reader.onload = () => {
          try {
            const base64Data = reader.result as string;
            sessionStorage.setItem(`file_${file.name}`, base64Data);
            resolve(file.name);
          } catch (err) {
            reject(err);
          }
        };
        
        reader.onerror = () => {
          reject(new Error(`Failed to read file: ${file.name}`));
        };
        
        reader.readAsDataURL(file);
      });
    });
    
    // Wait for all files to be stored in sessionStorage
    await Promise.all(storeFilePromises);
    
    // Verify all files were stored correctly
    const storedFileIds = JSON.parse(sessionStorage.getItem('pendingUploadFiles') || '[]');
    const missingFiles = storedFileIds.filter(id => !sessionStorage.getItem(`file_${id}`));
    
    if (missingFiles.length > 0) {
      throw new Error(`Failed to store ${missingFiles.length} files`);
    }
    
    // Dismiss loading toast
    toast.dismiss();
    
    // Proceed with checkout
    const session = await createCheckoutSession({
      product: {
        id: selectedPackage.product.id,
        name: selectedPackage.product.name,
        price: selectedPackage.imageCount.price,
        imageCount: selectedPackage.imageCount.count
      },
      processing: selectedPackage.processing,
      videoLength: selectedPackage.videoLength
    });

    if (session?.url) {
      router.push(session.url);
    }
  } catch (error) {
    console.error("Error creating checkout session:", error);
    toast.error("Failed to create checkout session");
  }
}, [files, selectedPackage, router]);
```

## Changes Made

1. **Asynchronous File Storage**: Converted the file storage process to use Promises and `Promise.all()` to ensure all files are stored before proceeding.
2. **Verification Step**: Added verification to confirm all files were properly stored in sessionStorage.
3. **Loading State**: Added a loading toast to indicate to the user that their files are being prepared.
4. **Error Handling**: Improved error handling for file reading and storage operations.

## Additional Recommendations

1. **File Size Limit**: Consider implementing a check for the total size of files to ensure they don't exceed sessionStorage limits.
2. **Alternative Storage**: For larger files, consider using IndexedDB instead of sessionStorage for more reliable storage.
3. **Unique File IDs**: Generate unique IDs for files instead of using file names to avoid potential conflicts.