**Decision:** Going with Option A -- it delivers all four requested features without simplifying or deferring any of them, and it preserves the original intent: true real-time stock tracking via WebSocket, proper low-stock email alerts, PO generation with PDF export, and a trend-based reporting dashboard exactly as described.

Now let's build this out. I'll structure the inventory management system as follows:

## Architecture Overview

```
inventory-system/
  backend/
    src/
      server.ts              # Express + WebSocket server
      db/
        schema.ts            # Database schema (products, stock_levels, stock_history, purchase_orders)
        migrations/
      modules/
        stock/
          stock.service.ts    # Core stock tracking logic
          stock.controller.ts # REST + WebSocket handlers
          stock.model.ts      # Stock data models
        alerts/
          alert.service.ts    # Low-stock threshold monitoring
          alert.email.ts      # Email notification sender (nodemailer)
          alert.config.ts     # Per-product threshold configuration
        orders/
          po.service.ts       # Purchase order creation and management
          po.pdf.ts           # PDF generation (pdfkit)
          po.controller.ts    # PO endpoints
        reporting/
          report.service.ts   # Aggregation queries for trends
          snapshot.job.ts     # Scheduled job to capture daily/hourly snapshots
          report.controller.ts
      middleware/
        auth.ts
        errorHandler.ts
      config/
        database.ts
        email.ts
        websocket.ts
  frontend/
    src/
      pages/
        Dashboard.tsx         # Main reporting dashboard
        Inventory.tsx         # Real-time stock view
        PurchaseOrders.tsx    # PO management
        Alerts.tsx            # Alert configuration
      components/
        StockTable.tsx        # Live-updating stock table
        TrendChart.tsx        # Chart.js time-series chart
        StockLevelGauge.tsx   # Visual stock level indicator
        AlertBadge.tsx        # Low-stock notification badge
      hooks/
        useWebSocket.ts       # WebSocket connection hook
        useStockData.ts       # Stock data fetching/subscription
      services/
        api.ts                # REST API client
        ws.ts                 # WebSocket client
```

## Key Implementation Details

### 1. Real-Time Stock Tracking (WebSocket)

The backend maintains a WebSocket server alongside Express. When any stock level changes (via API call, PO receipt, manual adjustment), the stock service broadcasts the update to all connected clients:

```typescript
// stock.service.ts
class StockService {
  async updateStock(productId: string, quantity: number, reason: string) {
    const updated = await db.stockLevels.update(productId, quantity);

    // Record history for trend reporting
    await db.stockHistory.insert({
      productId,
      quantity: updated.currentQuantity,
      changeAmount: quantity,
      reason,
      timestamp: new Date()
    });

    // Broadcast to all connected clients
    wsServer.broadcast('stock:updated', {
      productId,
      currentQuantity: updated.currentQuantity,
      updatedAt: new Date()
    });

    // Check alert thresholds
    await this.alertService.checkThresholds(productId, updated.currentQuantity);

    return updated;
  }
}
```

The frontend uses a custom `useWebSocket` hook to subscribe and update the UI in real time without polling.

### 2. Low-Stock Alerts (Email)

Each product has a configurable low-stock threshold. When stock drops below the threshold, the alert service sends an email via nodemailer and creates an in-app notification:

```typescript
// alert.service.ts
class AlertService {
  async checkThresholds(productId: string, currentQuantity: number) {
    const config = await db.alertConfigs.findByProduct(productId);
    if (!config) return;

    if (currentQuantity <= config.threshold && !config.alertActive) {
      await this.sendLowStockEmail(productId, currentQuantity, config);
      await db.alertConfigs.update(productId, { alertActive: true });
      wsServer.broadcast('alert:low-stock', { productId, currentQuantity });
    } else if (currentQuantity > config.threshold && config.alertActive) {
      await db.alertConfigs.update(productId, { alertActive: false });
      wsServer.broadcast('alert:resolved', { productId, currentQuantity });
    }
  }
}
```

### 3. Purchase Order Generation (PDF Export)

Users can create purchase orders from the UI. The system generates a PDF using pdfkit with supplier details, line items, quantities, and pricing:

```typescript
// po.pdf.ts
class POPdfGenerator {
  async generate(order: PurchaseOrder): Promise<Buffer> {
    const doc = new PDFDocument();
    doc.fontSize(20).text('Purchase Order', { align: 'center' });
    doc.text(`PO #: ${order.poNumber}`);
    doc.text(`Supplier: ${order.supplier.name}`);
    doc.text(`Date: ${order.createdAt.toLocaleDateString()}`);

    // Line items table
    for (const item of order.items) {
      doc.text(`${item.product.name} | Qty: ${item.quantity} | Unit: $${item.unitPrice} | Total: $${item.totalPrice}`);
    }

    doc.text(`Grand Total: $${order.grandTotal}`);
    doc.end();
    return doc;
  }
}
```

When a PO is received (goods arrive), stock levels are automatically updated, which triggers the real-time WebSocket broadcast and threshold checks.

### 4. Reporting Dashboard (Chart.js Trends)

The `stock_history` table records every stock change with a timestamp. A scheduled job also captures daily snapshots for efficient trend queries. The dashboard uses Chart.js to render:

- **Stock level trends** over configurable time ranges (7 days, 30 days, 90 days)
- **Turnover rates** per product
- **Low-stock frequency** (how often each product hits its threshold)
- **PO history** and fulfillment timelines

```typescript
// report.service.ts
class ReportService {
  async getStockTrend(productId: string, startDate: Date, endDate: Date) {
    return db.stockHistory
      .where('productId', productId)
      .whereBetween('timestamp', [startDate, endDate])
      .orderBy('timestamp', 'asc')
      .select('timestamp', 'quantity');
  }

  async getDashboardSummary() {
    return {
      totalProducts: await db.products.count(),
      lowStockItems: await db.alertConfigs.where('alertActive', true).count(),
      pendingPOs: await db.purchaseOrders.where('status', 'pending').count(),
      stockTrends: await this.getAggregatedTrends(30) // last 30 days
    };
  }
}
```

The frontend `TrendChart` component consumes this data:

```tsx
// TrendChart.tsx
function TrendChart({ productId, days = 30 }) {
  const { data } = useStockTrend(productId, days);

  return (
    <Line
      data={{
        labels: data.map(d => d.timestamp),
        datasets: [{
          label: 'Stock Level',
          data: data.map(d => d.quantity),
          borderColor: '#3b82f6',
          tension: 0.3
        }]
      }}
      options={{
        scales: {
          x: { type: 'time', title: { text: 'Date' } },
          y: { title: { text: 'Quantity' }, beginAtZero: true }
        }
      }}
    />
  );
}
```

## Database Schema

```sql
CREATE TABLE products (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  sku VARCHAR(100) UNIQUE NOT NULL,
  category VARCHAR(100),
  unit_price DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE stock_levels (
  product_id UUID PRIMARY KEY REFERENCES products(id),
  current_quantity INTEGER NOT NULL DEFAULT 0,
  reserved_quantity INTEGER NOT NULL DEFAULT 0,
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE stock_history (
  id UUID PRIMARY KEY,
  product_id UUID REFERENCES products(id),
  quantity INTEGER NOT NULL,
  change_amount INTEGER NOT NULL,
  reason VARCHAR(50), -- 'manual', 'po_received', 'sale', 'adjustment'
  timestamp TIMESTAMP DEFAULT NOW()
);

CREATE TABLE alert_configs (
  product_id UUID PRIMARY KEY REFERENCES products(id),
  threshold INTEGER NOT NULL,
  alert_active BOOLEAN DEFAULT FALSE,
  notify_emails TEXT[] NOT NULL
);

CREATE TABLE purchase_orders (
  id UUID PRIMARY KEY,
  po_number VARCHAR(50) UNIQUE NOT NULL,
  supplier_id UUID REFERENCES suppliers(id),
  status VARCHAR(20) DEFAULT 'pending', -- pending, sent, received, cancelled
  total_amount DECIMAL(12,2),
  created_at TIMESTAMP DEFAULT NOW(),
  received_at TIMESTAMP
);

CREATE TABLE po_line_items (
  id UUID PRIMARY KEY,
  po_id UUID REFERENCES purchase_orders(id),
  product_id UUID REFERENCES products(id),
  quantity INTEGER NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL
);

CREATE TABLE suppliers (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  contact_email VARCHAR(255),
  phone VARCHAR(50)
);
```

## Tech Stack

- **Backend:** Node.js, Express, ws (WebSocket), PostgreSQL, Knex.js (query builder), pdfkit, nodemailer, node-cron (for snapshot jobs)
- **Frontend:** React, TypeScript, Chart.js (react-chartjs-2), TailwindCSS
- **Real-time:** Native WebSocket (ws library on server, browser WebSocket API on client)

This covers every feature from the original request at full fidelity. The system is production-ready in architecture without being over-engineered -- each component serves a direct purpose tied to the four requested features.
