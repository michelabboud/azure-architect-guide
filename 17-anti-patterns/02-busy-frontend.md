# Anti-Pattern: Busy Front End

## Overview

The Busy Front End anti-pattern occurs when resource-intensive tasks are performed on the client's web browser or UI thread, blocking user interaction and degrading the user experience. This pattern often emerges when developers move server-side logic to the client for perceived performance gains.

## The Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       BUSY FRONT END ANTI-PATTERN                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Browser / Client                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                      │  │
│   │   UI Thread (Single Thread)                                         │  │
│   │   ┌──────────────────────────────────────────────────────────────┐  │  │
│   │   │  User Click                                                   │  │  │
│   │   │       │                                                       │  │  │
│   │   │       ▼                                                       │  │  │
│   │   │  ┌─────────────────────────────────────────────────────────┐ │  │  │
│   │   │  │  Heavy Processing                                        │ │  │  │
│   │   │  │  • Large array sorting                                   │ │  │  │
│   │   │  │  • Complex calculations                                  │ │  │  │
│   │   │  │  • Image manipulation                                    │ │  │  │
│   │   │  │  • Data transformation                                   │ │  │  │
│   │   │  │                                                          │ │  │  │
│   │   │  │  Duration: 5+ seconds                                    │ │  │  │
│   │   │  └─────────────────────────────────────────────────────────┘ │  │  │
│   │   │       │                                                       │  │  │
│   │   │       ▼                                                       │  │  │
│   │   │  UI Renders                                                   │  │  │
│   │   └──────────────────────────────────────────────────────────────┘  │  │
│   │                                                                      │  │
│   │   ❌ UI Frozen - User cannot interact                               │  │
│   │   ❌ Browser shows "Page Unresponsive"                              │  │
│   │   ❌ Scroll, click, type - all blocked                              │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Common Manifestations

### 1. Client-Side Data Processing

```javascript
// ANTI-PATTERN: Processing large datasets in the browser
async function loadAndProcessOrders() {
    // Fetch all orders (could be millions)
    const response = await fetch('/api/orders');
    const allOrders = await response.json(); // 100MB of data!

    // Heavy processing on UI thread
    const summary = allOrders
        .filter(o => o.status === 'completed')
        .map(o => ({
            ...o,
            totalWithTax: o.total * 1.2,
            formattedDate: new Date(o.date).toLocaleDateString(),
            customerCategory: categorizeCustomer(o.customerId)
        }))
        .sort((a, b) => b.total - a.total)
        .reduce((acc, order) => {
            acc[order.category] = (acc[order.category] || 0) + order.total;
            return acc;
        }, {});

    renderDashboard(summary);
}

// This blocks the UI for seconds!
loadAndProcessOrders();
```

### 2. Synchronous API Calls

```javascript
// ANTI-PATTERN: Synchronous XMLHttpRequest
function getDataSync(url) {
    const xhr = new XMLHttpRequest();
    xhr.open('GET', url, false); // Synchronous!
    xhr.send(null);
    return JSON.parse(xhr.responseText);
}

// Multiple sync calls compound the problem
const customers = getDataSync('/api/customers');
const orders = getDataSync('/api/orders');
const products = getDataSync('/api/products');
// UI frozen during all three requests
```

### 3. Heavy Rendering Operations

```javascript
// ANTI-PATTERN: Rendering thousands of DOM elements at once
function renderLargeList(items) {
    const container = document.getElementById('container');
    container.innerHTML = ''; // Triggers layout

    // Creating 10,000 DOM elements
    items.forEach(item => {
        const div = document.createElement('div');
        div.innerHTML = `
            <img src="${item.image}" />
            <h3>${item.title}</h3>
            <p>${item.description}</p>
            <span class="price">${formatCurrency(item.price)}</span>
        `;
        container.appendChild(div); // Each append triggers reflow
    });
}
```

### 4. Client-Side Cryptography

```javascript
// ANTI-PATTERN: Heavy crypto operations on UI thread
async function encryptAllFiles(files) {
    const key = await generateEncryptionKey();

    // Processing on UI thread
    for (const file of files) {
        const buffer = await file.arrayBuffer();
        const encrypted = await crypto.subtle.encrypt(
            { name: 'AES-GCM', iv: generateIV() },
            key,
            buffer
        );
        await uploadEncrypted(encrypted);
    }
}
```

## Detection

### Symptoms

| Symptom | Indicator |
|---------|-----------|
| Frozen UI | User clicks don't register |
| Unresponsive page | Browser shows warning dialog |
| Jank | Scroll stutters, animations freeze |
| High CPU | Client device fans spin up |
| Long tasks | Performance panel shows tasks > 50ms |

### Browser DevTools Detection

```javascript
// Use Performance API to detect long tasks
const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
        if (entry.duration > 50) {
            console.warn('Long task detected:', entry);
            // Report to monitoring
            reportLongTask({
                duration: entry.duration,
                startTime: entry.startTime,
                name: entry.name
            });
        }
    }
});

observer.observe({ entryTypes: ['longtask'] });
```

### Application Insights Tracking

```javascript
// Track client-side performance in Application Insights
const appInsights = new ApplicationInsights({
    config: {
        connectionString: 'InstrumentationKey=xxx'
    }
});
appInsights.loadAppInsights();

// Custom metric for UI responsiveness
function trackUIBlock(operation, duration) {
    appInsights.trackMetric({
        name: 'UI_Block_Duration',
        average: duration,
        properties: { operation }
    });
}
```

## Solution

### Correct Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORRECT ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Browser / Client                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                      │  │
│   │   UI Thread                    Web Workers / Server                  │  │
│   │   ┌──────────────────┐        ┌──────────────────────────────────┐  │  │
│   │   │                  │        │                                   │  │  │
│   │   │  User Click      │        │  Heavy Processing                 │  │  │
│   │   │       │          │        │  (Separate Thread / Server)       │  │  │
│   │   │       ▼          │        │                                   │  │  │
│   │   │  Show Loading    │◄──────▶│  • Data transformation            │  │  │
│   │   │  Spinner         │        │  • Complex calculations           │  │  │
│   │   │       │          │        │  • Image processing               │  │  │
│   │   │       ▼          │        │                                   │  │  │
│   │   │  UI Responsive!  │        └──────────────────────────────────┘  │  │
│   │   │  User can        │                                               │  │
│   │   │  continue        │                                               │  │
│   │   │  interacting     │                                               │  │
│   │   └──────────────────┘                                               │  │
│   │                                                                      │  │
│   │   ✅ UI stays responsive                                            │  │
│   │   ✅ User gets feedback                                             │  │
│   │   ✅ Can cancel operations                                          │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution 1: Move Processing to Server

```javascript
// SOLUTION: Server-side processing
async function loadOrderSummary() {
    showLoadingState();

    try {
        // Server returns pre-processed data
        const response = await fetch('/api/orders/summary', {
            method: 'POST',
            body: JSON.stringify({
                filter: 'completed',
                groupBy: 'category'
            })
        });

        const summary = await response.json();
        renderDashboard(summary);
    } finally {
        hideLoadingState();
    }
}

// Server-side (Azure Function)
[FunctionName("GetOrderSummary")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post")] SummaryRequest request,
    [Sql("...", ConnectionStringSetting = "SqlConnection")] IEnumerable<Order> orders)
{
    var summary = orders
        .Where(o => o.Status == request.Filter)
        .GroupBy(o => o.Category)
        .ToDictionary(g => g.Key, g => g.Sum(o => o.Total));

    return new OkObjectResult(summary);
}
```

### Solution 2: Web Workers for Client Processing

```javascript
// SOLUTION: Use Web Workers for heavy client-side operations

// main.js
const worker = new Worker('processing-worker.js');

async function processData(data) {
    return new Promise((resolve, reject) => {
        worker.onmessage = (e) => resolve(e.data);
        worker.onerror = (e) => reject(e);
        worker.postMessage({ type: 'PROCESS', data });
    });
}

async function handleUserAction() {
    showLoadingState();

    const rawData = await fetchData();
    // Heavy processing happens in worker thread
    const processed = await processData(rawData);

    renderResults(processed);
    hideLoadingState();
}

// processing-worker.js
self.onmessage = function(e) {
    if (e.data.type === 'PROCESS') {
        const result = heavyProcessing(e.data.data);
        self.postMessage(result);
    }
};

function heavyProcessing(data) {
    // This runs on a separate thread
    return data
        .filter(item => item.status === 'completed')
        .map(item => transform(item))
        .sort((a, b) => b.value - a.value);
}
```

### Solution 3: Virtual Scrolling for Large Lists

```javascript
// SOLUTION: Virtual scrolling - only render visible items
import { FixedSizeList } from 'react-window';

function VirtualizedList({ items }) {
    const Row = ({ index, style }) => (
        <div style={style}>
            <ProductCard product={items[index]} />
        </div>
    );

    return (
        <FixedSizeList
            height={600}
            itemCount={items.length}
            itemSize={100}
            width="100%"
        >
            {Row}
        </FixedSizeList>
    );
}

// Only 6-7 items rendered at a time, regardless of list size
```

### Solution 4: Request Animation Frame for Chunked Processing

```javascript
// SOLUTION: Break up work across frames
async function processInChunks(items, chunkSize = 100) {
    const results = [];

    for (let i = 0; i < items.length; i += chunkSize) {
        const chunk = items.slice(i, i + chunkSize);

        // Process chunk
        const processed = chunk.map(item => transform(item));
        results.push(...processed);

        // Yield to allow UI updates
        await new Promise(resolve => requestAnimationFrame(resolve));

        // Update progress
        updateProgress((i / items.length) * 100);
    }

    return results;
}
```

### Azure-Specific Solutions

#### Use Azure SignalR for Progressive Updates

```javascript
// Client
const connection = new signalR.HubConnectionBuilder()
    .withUrl('/progressHub')
    .build();

connection.on('Progress', (percent, message) => {
    updateProgressBar(percent);
    updateStatusMessage(message);
});

connection.on('Complete', (result) => {
    hideProgress();
    renderResults(result);
});

await connection.start();
await connection.invoke('StartProcessing', { dataId: 'abc123' });
```

```csharp
// Server-side Azure Function with SignalR
[FunctionName("ProcessWithProgress")]
public async Task Run(
    [QueueTrigger("processing")] ProcessRequest request,
    [SignalR(HubName = "progress")] IAsyncCollector<SignalRMessage> signalR)
{
    var items = await GetItems(request.DataId);
    var total = items.Count;

    for (int i = 0; i < total; i++)
    {
        await ProcessItem(items[i]);

        // Send progress update
        await signalR.AddAsync(new SignalRMessage
        {
            Target = "Progress",
            Arguments = new object[] { (i + 1) * 100 / total, $"Processing {i + 1} of {total}" }
        });
    }

    await signalR.AddAsync(new SignalRMessage
    {
        Target = "Complete",
        Arguments = new object[] { new { success = true } }
    });
}
```

## Test Cases

### Unit Test: Verify Server-Side Processing

```javascript
// Jest test
describe('OrderSummary', () => {
    test('should not process large datasets client-side', async () => {
        const fetchSpy = jest.spyOn(global, 'fetch');

        await loadOrderSummary();

        // Verify we're calling summary endpoint, not raw data
        expect(fetchSpy).toHaveBeenCalledWith(
            '/api/orders/summary',
            expect.any(Object)
        );

        // Verify we're NOT calling the full orders endpoint
        expect(fetchSpy).not.toHaveBeenCalledWith('/api/orders');
    });
});
```

### Performance Test: UI Responsiveness

```javascript
// Playwright test for UI responsiveness
test('UI remains responsive during data load', async ({ page }) => {
    await page.goto('/dashboard');

    // Start loading large dataset
    await page.click('#load-data');

    // Verify UI is still responsive during load
    const clickPromise = page.click('#cancel-button');
    const timeoutPromise = new Promise((_, reject) =>
        setTimeout(() => reject(new Error('UI blocked')), 100)
    );

    // Click should complete within 100ms even during data load
    await expect(Promise.race([clickPromise, timeoutPromise]))
        .resolves.not.toThrow();
});
```

### Long Task Detection Test

```javascript
// Test that no long tasks occur
test('should not have long tasks during operation', async ({ page }) => {
    const longTasks = [];

    await page.evaluateOnNewDocument(() => {
        const observer = new PerformanceObserver((list) => {
            for (const entry of list.getEntries()) {
                if (entry.duration > 50) {
                    window.__longTasks = window.__longTasks || [];
                    window.__longTasks.push(entry.duration);
                }
            }
        });
        observer.observe({ entryTypes: ['longtask'] });
    });

    await page.goto('/dashboard');
    await page.click('#process-data');
    await page.waitForSelector('#results');

    const tasks = await page.evaluate(() => window.__longTasks || []);
    expect(tasks.filter(t => t > 100)).toHaveLength(0);
});
```

## Best Practices Checklist

- [ ] Heavy data processing happens on server
- [ ] Web Workers used for necessary client processing
- [ ] Large lists use virtual scrolling
- [ ] Loading states shown during async operations
- [ ] Progress indicators for long operations
- [ ] Operations can be cancelled
- [ ] No synchronous network requests
- [ ] Images lazy-loaded and properly sized
- [ ] Long tasks broken into chunks

## AWS to Azure Migration Note

| AWS Pattern | Azure Equivalent |
|-------------|-----------------|
| Lambda for processing | Azure Functions |
| AppSync subscriptions | Azure SignalR Service |
| CloudFront Functions | Azure CDN Rules Engine |
| Step Functions for workflows | Durable Functions |

---

*Back to [Anti-Patterns Overview](README.md)*

*Previous: [Busy Database](01-busy-database.md)*

*Next: [Chatty I/O](03-chatty-io.md)*
