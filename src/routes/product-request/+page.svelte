<script lang="ts">
  import { onMount } from 'svelte';
  import { fade } from 'svelte/transition';
  import Select from 'svelte-select';
  import { db } from '$lib/firebase';
  import { collection, addDoc, serverTimestamp } from 'firebase/firestore';
  import { currentUser } from '$lib/firebase';

  interface SelectOption {
    value: string;
    label: string;
  }

  interface ProductRow {
    sku: string;
    productName: string;
    brand: SelectOption | null;
    supplier: SelectOption | null;
    purchasePrice: string;
    rrp: string;
    exists: boolean;
  }

  // API Endpoints
  const brandsUrl = 'https://prod-06.australiasoutheast.logic.azure.com:443/workflows/58215302c1c24203886ccf481adbaac5/triggers/manual/paths/invoke?api-version=2016-06-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=RFQ4OtbS6cyjB_JzaIsowmww4KBqPQgavWLg18znE5s';
  const suppliersUrl = 'https://prod-06.australiasoutheast.logic.azure.com:443/workflows/da5c5708146642768d63293d2bbb9668/triggers/manual/paths/invoke?api-version=2016-06-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=-n0W0PxlF1G83xHYHGoEOhv3XmHXWlesbRk5NcgNT9w';
  const skuCheckUrl = 'https://prod-03.australiasoutheast.logic.azure.com:443/workflows/151bc47e0ba4447b893d1c9fea9af46f/triggers/manual/paths/invoke?api-version=2016-06-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=bRyr_oW-ud06XlU5VLhBqQ7tyU__jD3clEOGIEhax-Q';
  const notificationUrl = 'https://prod-24.australiasoutheast.logic.azure.com:443/workflows/16979e5f23434b988b37be58343e93e9/triggers/manual/paths/invoke?api-version=2016-06-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=loAkudpZIyE7_2o54CIncgVBLoXBtND6G_4Qm2MJzOE';

  // State for brands and suppliers
  let brands: SelectOption[] = [];
  let suppliers: SelectOption[] = [];
  let loadingBrands = false;
  let loadingSuppliers = false;
  let brandError = '';
  let supplierError = '';

  // Product Request Form
  let rows: ProductRow[] = [createEmptyRow()];
  let isLoading = false;
  let notification = { show: false, message: '', type: 'info' };

  // State for user
  let user: any;
  
  // Subscribe to user changes
  currentUser.subscribe((u) => {
    user = u;
  });

  function createEmptyRow(): ProductRow {
    return {
      sku: '',
      productName: '',
      brand: null,
      supplier: null,
      purchasePrice: '',
      rrp: '',
      exists: false
    };
  }

  function addRow() {
    rows = [...rows, createEmptyRow()];
  }

  function removeRow(index: number) {
    rows = rows.filter((_, i) => i !== index);
  }

  function applyToAll(field: 'brand' | 'supplier', value: SelectOption | null) {
    rows = rows.map(row => ({
      ...row,
      [field]: value
    }));
  }

  // Function to fetch brands
  async function fetchBrands() {
    console.log('Starting fetchBrands...');
    loadingBrands = true;
    brandError = '';
    try {
      console.log('Making API call to fetch brands...');
      const response = await fetch(brandsUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' }
      });
      
      console.log('API Response status:', response.status);
      const data = await response.json();
      console.log('API Response data:', data);

      if (data.status === 200 && data.message.Ack === "Success") {
        brands = data.message.Content.map((brand: { ContentID: string; ContentName: string }) => ({
          value: brand.ContentID,
          label: brand.ContentName
        }));
        console.log('Processed brands:', brands);
      } else {
        throw new Error('Failed to load brands: Invalid response format');
      }
    } catch (error) {
      console.error('Error in fetchBrands:', error);
      brandError = error instanceof Error ? error.message : 'Failed to load brands';
    } finally {
      loadingBrands = false;
      console.log('fetchBrands completed. Brands:', brands, 'Error:', brandError);
    }
  }

  // Function to fetch suppliers
  async function fetchSuppliers() {
    console.log('Starting fetchSuppliers...');
    loadingSuppliers = true;
    supplierError = '';
    try {
      console.log('Making API call to fetch suppliers...');
      const response = await fetch(suppliersUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({})
      });
      
      console.log('API Response status:', response.status);
      const data = await response.json();
      console.log('API Response data:', data);

      if (data.status === 200 && data.message.Ack === "Success") {
        suppliers = data.message.Supplier
          .filter((supplier: { SupplierID: string }) => supplier.SupplierID !== "0")
          .map((supplier: { SupplierID: string }) => ({
            value: supplier.SupplierID,
            label: supplier.SupplierID
          }))
          .sort((a: SelectOption, b: SelectOption) => a.label.localeCompare(b.label));
        console.log('Processed suppliers:', suppliers);
      } else {
        throw new Error('Failed to load suppliers: Invalid response format');
      }
    } catch (error) {
      console.error('Error in fetchSuppliers:', error);
      supplierError = error instanceof Error ? error.message : 'Failed to load suppliers';
    } finally {
      loadingSuppliers = false;
      console.log('fetchSuppliers completed. Suppliers:', suppliers, 'Error:', supplierError);
    }
  }

  // Function to send email notification
  async function sendEmailNotification(products: ProductRow[]) {
    console.log('=== Starting Email Notification Process ===');
    console.log('Current user:', user);
    
    if (!user?.email) {
      console.error('No user email available for notification');
      return;
    }

    try {
      // Create HTML table for products
      const productTable = `
        <table style="border-collapse: collapse; width: 100%; margin-top: 20px;">
          <thead>
            <tr style="background-color: #f3f4f6;">
              <th style="border: 1px solid #e5e7eb; padding: 8px; text-align: left;">SKU</th>
              <th style="border: 1px solid #e5e7eb; padding: 8px; text-align: left;">Product Name</th>
              <th style="border: 1px solid #e5e7eb; padding: 8px; text-align: left;">Brand</th>
              <th style="border: 1px solid #e5e7eb; padding: 8px; text-align: left;">Supplier</th>
              <th style="border: 1px solid #e5e7eb; padding: 8px; text-align: right;">Purchase Price</th>
              <th style="border: 1px solid #e5e7eb; padding: 8px; text-align: right;">RRP</th>
            </tr>
          </thead>
          <tbody>
            ${products.map(product => `
              <tr>
                <td style="border: 1px solid #e5e7eb; padding: 8px;">${product.sku}</td>
                <td style="border: 1px solid #e5e7eb; padding: 8px;">${product.productName}</td>
                <td style="border: 1px solid #e5e7eb; padding: 8px;">${product.brand?.label || '-'}</td>
                <td style="border: 1px solid #e5e7eb; padding: 8px;">${product.supplier?.label || '-'}</td>
                <td style="border: 1px solid #e5e7eb; padding: 8px; text-align: right;">$${parseFloat(product.purchasePrice).toFixed(2)}</td>
                <td style="border: 1px solid #e5e7eb; padding: 8px; text-align: right;">$${parseFloat(product.rrp).toFixed(2)}</td>
              </tr>
            `).join('')}
          </tbody>
        </table>
      `;

      // Create email body with user info and product table
      const emailBody = `
        <div style="font-family: Arial, sans-serif;">
          <h2>Product Request Submission</h2>
          
          <div style="margin-bottom: 20px;">
            <h3>Requestor Information:</h3>
            <p>Email: ${user.email}</p>
            <p>Name: ${user.displayName || 'N/A'}</p>
            <p>User ID: ${user.uid}</p>
            <p>Submission Time: ${new Date().toLocaleString()}</p>
          </div>

          <div>
            <h3>Requested Products:</h3>
            ${productTable}
          </div>
        </div>
      `;

      console.log('Preparing email payload');

      const response = await fetch(notificationUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          email_subject: 'Product Request Submission Confirmation',
          email_body: emailBody,
          email_send_to: user.email
        })
      });

      const data = await response.json();
      console.log('Email notification API response:', data);

      if (!response.ok) {
        throw new Error('Failed to send email notification');
      }

      console.log('Email notification sent successfully');
    } catch (error) {
      console.error('Error sending email notification:', error);
    } finally {
      console.log('=== Email Notification Process Completed ===');
    }
  }

  async function handleProductRequestSubmit() {
    isLoading = true;
    console.log('=== Starting Product Request Submission ===');
    console.log('Current user:', user);
    console.log('Number of rows to submit:', rows.length);
    
    try {
      // Validate rows
      let valid = true;
      const invalidRows = rows.filter(row => 
        !row.sku || 
        !row.productName || 
        !row.brand || 
        !row.supplier || 
        !row.purchasePrice
      );
      
      console.log('Validation check - Invalid rows:', invalidRows.length);
      if (invalidRows.length > 0) {
        console.error('Validation failed - Missing required fields:', invalidRows);
        throw new Error('Please fill in all required fields');
      }

      // Check for duplicate SKUs within the form
      const skus = rows.map(row => row.sku);
      console.log('SKUs to be submitted:', skus);
      
      const duplicates = skus.filter((sku, index) => skus.indexOf(sku) !== index);
      if (duplicates.length > 0) {
        console.error('Duplicate SKUs found in form:', duplicates);
        throw new Error('Duplicate SKUs found within the submitted requests. Please ensure each SKU is unique.');
      }

      // Check if SKUs already exist in the system
      console.log('Checking SKUs against existing system...');
      const response = await fetch(skuCheckUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ SKU: skus })
      });

      const data = await response.json();
      console.log('SKU check API response:', data);

      if (data.Ack === "Success") {
        const existingSKUs = new Set(data.Item?.map((item: any) => item.SKU) || []);
        const duplicateRows = rows.filter(row => existingSKUs.has(row.sku));
        
        if (duplicateRows.length > 0) {
          const duplicateSkus = duplicateRows.map(row => row.sku).join(', ');
          console.error('SKUs already exist in system:', duplicateSkus);
          throw new Error(`The following SKUs already exist: ${duplicateSkus}`);
        }

        console.log('=== Starting Firebase Submission ===');
        // If all validation passes, save to Firestore
        const savePromises = rows.map(row => {
          const productData = {
            sku: row.sku,
            product_name: row.productName,
            brand: row.brand?.label || '',
            primary_supplier: row.supplier?.value || '',
            purchase_price: parseFloat(row.purchasePrice) || 0,
            rrp: parseFloat(row.rrp) || 0,
            status: 'request',
            date_created: serverTimestamp(),
            requestor_uid: user?.uid || '',
            requestor_email: user?.email || ''
          };
          
          console.log('Preparing to save product data:', productData);
          return addDoc(collection(db, 'product_requests'), productData)
            .then(docRef => {
              console.log('Successfully saved product with SKU:', productData.sku, 'Document ID:', docRef.id);
              return docRef;
            })
            .catch(error => {
              console.error('Error saving product with SKU:', productData.sku, 'Error:', error);
              throw error;
            });
        });

        console.log('Waiting for all Firebase submissions to complete...');
        const results = await Promise.all(savePromises);
        console.log('All Firebase submissions completed successfully!');
        console.log('Saved document references:', results.map(ref => ref.id));

        // Send email notification after successful submission
        await sendEmailNotification(rows);

        showNotification('Product request submitted successfully', 'success');
        
        // Clear form and add new row
        rows = [createEmptyRow()];
        console.log('Form cleared and reset');
      } else {
        console.error('SKU check API returned error:', data);
        throw new Error('SKU check failed. Please try again later.');
      }
    } catch (error: any) {
      console.error('=== Submission Error ===');
      console.error('Error details:', error);
      showNotification(error.message || 'An error occurred', 'error');
    } finally {
      console.log('=== Submission Process Completed ===');
      isLoading = false;
    }
  }

  function showNotification(message: string, type: 'success' | 'error' | 'info' = 'info') {
    notification = { show: true, message, type };
    setTimeout(() => {
      notification = { ...notification, show: false };
    }, 3000);
  }

  // SVG icon for delete button
  const trashIcon = `<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" class="w-5 h-5">
    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
  </svg>`;

  // Function to handle Excel paste
  async function handlePaste(event: ClipboardEvent, rowIndex: number, field: keyof ProductRow) {
    console.log('=== Starting Paste Operation ===');
    console.log('Paste target:', { field, rowIndex });
    
    event.preventDefault();
    const clipboardData = event.clipboardData?.getData('text') || '';
    console.log('Raw clipboard data:', clipboardData);
    
    // Split the clipboard data into rows
    const pastedRows = clipboardData.split('\n')
      .map(row => row.split('\t'))
      .filter(row => row.some(cell => cell.trim() !== '')); // Filter out empty rows

    console.log('Processed rows:', pastedRows);

    if (pastedRows.length === 0) {
      console.log('No valid data to paste');
      return;
    }

    // For single cell paste
    if (pastedRows.length === 1 && pastedRows[0].length === 1) {
      const value = pastedRows[0][0].trim();
      console.log('Single cell paste:', { value, field, rowIndex });
      
      if (field === 'sku' || field === 'productName' || field === 'purchasePrice' || field === 'rrp') {
        rows[rowIndex][field] = value;
      }
      return;
    }

    // For multi-cell paste in a single column
    const values = pastedRows.map(row => row[0]?.trim() || '');
    console.log('Multi-cell paste values:', values);
    
    // Create new rows if needed
    const initialRowCount = rows.length;
    while (rowIndex + values.length > rows.length) {
      rows = [...rows, createEmptyRow()];
    }
    console.log(`Added ${rows.length - initialRowCount} new rows`);

    // Update the specific column for each row
    values.forEach((value, index) => {
      if (field === 'sku' || field === 'productName' || field === 'purchasePrice' || field === 'rrp') {
        rows[rowIndex + index][field] = value;
      }
    });

    // Show notification
    const notificationMessage = `Pasted ${values.length} values into ${field} column`;
    showNotification(notificationMessage, 'success');
    console.log(notificationMessage);
    console.log('=== Paste Operation Completed ===');
  }

  onMount(() => {
    console.log('Component mounted, fetching brands and suppliers...');
    fetchBrands();
    fetchSuppliers();
  });
</script>

<div class="min-h-screen bg-gray-100 py-8 px-2 sm:px-3">
  <div class="max-w-[98%] mx-auto bg-white shadow p-6" transition:fade>
    <h2 class="text-2xl font-bold mb-6 text-gray-900">Product Request</h2>
    
    <!-- Product Request Form -->
    <div class="space-y-6">
      <div class="flex justify-between items-center sticky top-0 bg-white/80 backdrop-blur-sm py-4 z-50 border-b border-gray-200 shadow-sm">
        <button
          on:click={addRow}
          class="bg-blue-600 text-white py-2 px-4 rounded hover:bg-blue-700"
        >
          Add Row
        </button>
        <button
          on:click={handleProductRequestSubmit}
          class="bg-green-600 text-white py-2 px-4 rounded hover:bg-green-700"
          disabled={isLoading}
        >
          {isLoading ? 'Submitting...' : 'Submit'}
        </button>
      </div>

      <!-- Product Rows -->
      <div class="overflow-visible">
        <!-- Headers -->
        <div class="hidden md:grid md:grid-cols-[1fr_1fr_1fr_1fr_1fr_1fr_40px] md:gap-4 md:px-6 md:py-3 text-sm font-medium text-gray-500 uppercase tracking-wider bg-gray-50 rounded-t-lg">
          <div>SKU</div>
          <div>Product Name</div>
          <div>
            Brand
            <button
              on:click={() => applyToAll('brand', rows[0]?.brand)}
              class="ml-2 text-blue-600 hover:text-blue-800 text-xs"
            >
              Apply to All
            </button>
          </div>
          <div>
            Supplier
            <button
              on:click={() => applyToAll('supplier', rows[0]?.supplier)}
              class="ml-2 text-blue-600 hover:text-blue-800 text-xs"
            >
              Apply to All
            </button>
          </div>
          <div>Purchase Price</div>
          <div>RRP</div>
          <div></div>
        </div>

        <!-- Rows -->
        <div class="divide-y divide-gray-200">
          {#each rows as row, i}
            <div class="bg-white md:hover:bg-gray-50 transition-colors">
              <div class="md:grid md:grid-cols-[1fr_1fr_1fr_1fr_1fr_1fr_40px] md:gap-4 md:items-center p-4 md:px-6 md:py-4">
                <!-- SKU -->
                <div class="mb-4 md:mb-0">
                  <label class="block md:hidden text-sm font-medium text-gray-700 mb-1">SKU</label>
                  <input
                    type="text"
                    bind:value={row.sku}
                    on:paste={(e) => handlePaste(e, i, 'sku')}
                    class="block w-full border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                    placeholder="SKU"
                  />
                </div>

                <!-- Product Name -->
                <div class="mb-4 md:mb-0">
                  <label class="block md:hidden text-sm font-medium text-gray-700 mb-1">Product Name</label>
                  <input
                    type="text"
                    bind:value={row.productName}
                    on:paste={(e) => handlePaste(e, i, 'productName')}
                    class="block w-full border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                    placeholder="Product Name"
                  />
                </div>

                <!-- Brand -->
                <div class="mb-4 md:mb-0">
                  <label class="block md:hidden text-sm font-medium text-gray-700 mb-1">Brand</label>
                  {#if loadingBrands}
                    <div class="animate-pulse bg-gray-200 h-9 rounded"></div>
                  {:else if brandError}
                    <div class="text-red-600 text-sm">{brandError}</div>
                  {:else}
                    <Select
                      items={brands}
                      bind:value={row.brand}
                      placeholder="Select Brand"
                      containerStyles="position: relative;"
                      portal={null}
                    />
                  {/if}
                </div>

                <!-- Supplier -->
                <div class="mb-4 md:mb-0">
                  <label class="block md:hidden text-sm font-medium text-gray-700 mb-1">Supplier</label>
                  {#if loadingSuppliers}
                    <div class="animate-pulse bg-gray-200 h-9 rounded"></div>
                  {:else if supplierError}
                    <div class="text-red-600 text-sm">{supplierError}</div>
                  {:else}
                    <Select
                      items={suppliers}
                      bind:value={row.supplier}
                      placeholder="Select Supplier"
                      containerStyles="position: relative;"
                      portal={null}
                    />
                  {/if}
                </div>

                <!-- Purchase Price -->
                <div class="mb-4 md:mb-0 input-wrapper">
                  <label class="block md:hidden text-sm font-medium text-gray-700 mb-1">Purchase Price</label>
                  <input
                    type="number"
                    bind:value={row.purchasePrice}
                    on:paste={(e) => handlePaste(e, i, 'purchasePrice')}
                    class="block w-full border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                    placeholder="Purchase Price"
                    step="0.01"
                  />
                </div>

                <!-- RRP -->
                <div class="mb-4 md:mb-0">
                  <label class="block md:hidden text-sm font-medium text-gray-700 mb-1">RRP</label>
                  <input
                    type="number"
                    bind:value={row.rrp}
                    on:paste={(e) => handlePaste(e, i, 'rrp')}
                    class="block w-full border-gray-300 rounded-md shadow-sm focus:ring-blue-500 focus:border-blue-500 sm:text-sm"
                    placeholder="RRP"
                    step="0.01"
                  />
                </div>

                <!-- Action -->
                <div class="text-right md:text-center">
                  <button
                    on:click={() => removeRow(i)}
                    class="md:p-1 md:rounded hover:bg-red-50 transition-colors inline-flex items-center justify-center"
                    disabled={rows.length === 1}
                    title="Remove row"
                  >
                    <!-- Mobile view: Text button -->
                    <span class="md:hidden bg-red-600 text-white px-3 py-1 rounded text-sm hover:bg-red-700">
                      Remove
                    </span>
                    
                    <!-- Desktop view: Icon only -->
                    <svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor" 
                      class="hidden md:block w-4 h-4 text-red-600 hover:text-red-900">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                    </svg>
                    <span class="sr-only">Remove row</span>
                  </button>
                </div>
              </div>
            </div>
          {/each}
        </div>
      </div>
    </div>
  </div>

  <!-- Notification -->
  {#if notification.show}
    <div
      class="fixed bottom-4 right-4 px-4 py-2 rounded shadow-lg"
      class:bg-green-500={notification.type === 'success'}
      class:bg-red-500={notification.type === 'error'}
      class:bg-blue-500={notification.type === 'info'}
      transition:fade
    >
      <p class="text-white">{notification.message}</p>
    </div>
  {/if}

  <!-- Loading Overlay -->
  {#if isLoading}
    <div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div class="bg-white p-6 rounded-lg shadow-xl">
        <div class="animate-spin rounded-full h-12 w-12 border-4 border-blue-500 border-t-transparent mx-auto"></div>
        <p class="mt-4 text-gray-700">Processing...</p>
      </div>
    </div>
  {/if}
</div>

<!-- Add portal container at the end of the body -->
<div id="select-portal" />

<style>
  :global(body) {
    background-color: #f3f4f6;
  }

  :global(.svelte-select) {
    --height: 38px;
    --border: 1px solid #d1d5db;
    --border-hover: 1px solid #3b82f6;
    --border-radius: 0.375rem;
    --background: white;
    --font-size: 0.875rem;
    --padding: 0 0.75rem;
    --placeholder-color: #9ca3af;
    width: 100%;
    position: relative;
  }

  :global(.svelte-select .selectContainer) {
    border: var(--border);
    border-radius: var(--border-radius);
    height: var(--height);
    background: var(--background);
    min-height: var(--height);
    padding: 0;
  }

  :global(.svelte-select .items) {
    position: absolute;
    top: 100%;
    left: 0;
    right: 0;
    border: var(--border);
    border-radius: var(--border-radius);
    background: var(--background);
    margin-top: 4px;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
    z-index: 999;
    max-height: 300px;
    overflow-y: auto;
  }

  :global(.svelte-select .item) {
    font-size: var(--font-size);
    line-height: 1.25;
    padding: 0.5rem 0.75rem;
    white-space: normal;
    word-break: break-word;
  }

  :global(.svelte-select .item.hover) {
    background-color: #f3f4f6;
  }

  :global(.svelte-select .item.active) {
    background-color: #dbeafe;
    color: #1e40af;
  }

  #select-portal {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 0;
    overflow: visible;
    z-index: 999;
  }

  /* Add styles for the select wrapper */
  .select-wrapper {
    position: relative;
    z-index: 10;
  }
  
  .select-wrapper:focus-within {
    z-index: 20;
  }

  /* Ensure other elements don't overlap */
  .input-wrapper {
    position: relative;
    z-index: 1;
  }

  :global(.svelte-select .value-container) {
    padding: var(--padding);
    height: var(--height);
    line-height: var(--height);
    font-size: var(--font-size);
    display: flex;
    align-items: center;
  }

  :global(.svelte-select .selected-item) {
    display: flex;
    align-items: center;
    height: 100%;
    line-height: normal;
  }

  :global(.svelte-select input) {
    font-size: var(--font-size);
    padding: var(--padding);
    height: calc(var(--height) - 2px);
    display: flex;
    align-items: center;
  }

  :global(.svelte-select .placeholder) {
    color: var(--placeholder-color);
    font-size: var(--font-size);
  }

  :global(button:disabled) {
    opacity: 0.5;
    cursor: not-allowed;
  }

  :global(.sr-only) {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border-width: 0;
  }
</style> 