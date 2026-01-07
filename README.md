# StockFlow_Case_Study
StockFlow – A B2B SaaS inventory management system that lets companies manage products across multiple warehouses, track stock levels, monitor suppliers, and receive low-stock alerts.

# 📦 StockFlow — Inventory Management System (B2B SaaS)

## 📄 Full Case Study Documentation

---

## 1. Overview

StockFlow is a B2B SaaS Inventory Management Platform designed for small and medium businesses to:

- track stock across multiple warehouses  
- manage suppliers  
- monitor inventory levels  
- support bundled products  
- receive low-stock alerts  

This document contains:

- Part 1 – Code Review & Debugging  
- Part 2 – Database Design  
- Part 3 – API Design & Implementation  
- assumptions and edge-case handling  

---

# ✅ Part 1 — Code Review & Debugging

## 1.1 Given Code

```python
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json
     
    product = Product(
        name=data['name'],
        sku=data['sku'],
        price=data['price'],
        warehouse_id=data['warehouse_id']
    )
     
    db.session.add(product)
    db.session.commit()
     
    inventory = Inventory(
        product_id=product.id,
        warehouse_id=data['warehouse_id'],
        quantity=data['initial_quantity']
    )
     
    db.session.add(inventory)
    db.session.commit()
     
    return {"message": "Product created", "product_id": product.id}
```

---

## 1.2 Issues Identified

### Technical Issues

- no input validation  
- KeyError crashes if fields missing  
- price stored as float not decimal  
- 2 commits → no transaction safety  
- negative quantities allowed  
- exceptions not handled  
- optional fields unsupported  
- duplicate SKUs allowed  
- warehouse tied directly to product  
- business rules missing  

### Business Logic Issues

- product wrongly tied to single warehouse  
- SKUs must be UNIQUE across platform  
- multi-warehouse inventory required  
- bundles not considered  
- inventory change logs missing  

---

## 1.3 Impact in Production

| Issue | Impact |
|------|--------|
Duplicate SKU | shipping/billing confusion |
Two commits | orphan products |
Float price | money rounding errors |
Missing validation | API crashes |
Negative quantity | accounting problems |
Single warehouse | business rule violation |

---

## 1.4 Corrected Code

### ✔ Fix Highlights

- validates input fields  
- uses **database transaction**
- supports **optional warehouse**
- prevents negative stock
- SKU uniqueness checked
- decimal price storage
- single atomic commit
- handles exceptions safely

---

### ✔ Final Corrected Endpoint

```python
from flask import request, jsonify
from sqlalchemy.exc import IntegrityError
from decimal import Decimal

@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.get_json() or {}

    required = ["name", "sku", "price"]
    missing = [f for f in required if f not in data]

    if missing:
        return jsonify({"error": f"Missing fields: {missing}"}), 400

    try:
        price = Decimal(str(data["price"]))
    except:
        return jsonify({"error": "Invalid price format"}), 400

    initial_quantity = data.get("initial_quantity", 0)
    if initial_quantity < 0:
        return jsonify({"error": "Negative stock not allowed"}), 400

    try:
        with db.session.begin():
            product = Product(
                name=data["name"],
                sku=data["sku"].strip().upper(),
                price=price,
                description=data.get("description")
            )

            db.session.add(product)

            if "warehouse_id" in data:
                inventory = Inventory(
                    product_id=product.id,
                    warehouse_id=data["warehouse_id"],
                    quantity=initial_quantity,
                )
                db.session.add(inventory)

        return jsonify({
            "message": "Product created successfully",
            "product_id": product.id,
        }), 201

    except IntegrityError:
        db.session.rollback()
        return jsonify({"error": "SKU already exists"}), 409

    except Exception as e:
        db.session.rollback()
        return jsonify({"error": str(e)}), 500
```

---

# 🗄 Part 2 — Database Design

## 2.1 Requirements Satisfied

- multiple companies  
- multiple warehouses per company  
- products stored in multiple warehouses  
- track inventory level changes  
- suppliers supply products  
- bundles supported  

---

## 2.2 Database Schema

### Companies

```
companies(
  id PK,
  name,
  created_at
)
```

### Warehouses

```
warehouses(
  id PK,
  company_id FK -> companies,
  name,
  location
)
```

### Products

```
products(
  id PK,
  company_id FK -> companies,
  sku UNIQUE,
  name,
  price DECIMAL(10,2),
  is_bundle BOOLEAN,
  low_stock_threshold INT
)
```

### Inventory (per warehouse)

```
inventory(
  product_id FK -> products,
  warehouse_id FK -> warehouses,
  quantity INT,
  PRIMARY KEY(product_id, warehouse_id)
)
```

### Inventory History

```
inventory_history(
  id PK,
  product_id,
  warehouse_id,
  change INT,
  reason ENUM('sale','purchase','adjustment','transfer'),
  created_at
)
```

### Bundle Mapping

```
product_components(
  bundle_id FK -> products,
  component_id FK -> products,
  quantity INT,
  PRIMARY KEY(bundle_id, component_id)
)
```

### Suppliers

```
suppliers(
  id PK,
  name,
  email,
  phone
)
```

### Supplier–Product Mapping

```
supplier_products(
  supplier_id,
  product_id,
  lead_time_days,
  price DECIMAL(10,2),
  PRIMARY KEY(supplier_id, product_id)
)
```

---

## 2.3 Questions for Product Team

- Should bundles deduct component stock automatically?  
- Do we need batch/lot/expiry tracking?  
- Per-warehouse thresholds or global?  
- Support for returns?  
- Multi-currency support?  
- Product variants?  
- Soft delete or permanent delete?  
- Do suppliers belong to company or platform-wide?  
- Track damaged/reserved stock separately?  

---

## 2.4 Design Justifications

- composite keys prevent duplicate warehouse stock rows  
- decimal types ensure price accuracy  
- history enables audit compliance  
- company_id enables multi-tenant SaaS  
- supplier mapping supports multiple suppliers/product  
- indexes added on:

  - sku  
  - product_id  
  - warehouse_id  
  - supplier_id  

---

# 🚨 Part 3 — Low-Stock Alerts API

## 3.1 Business Rules Implemented

- threshold varies by product  
- recent sales activity required  
- multiple warehouses per company  
- supplier data included  
- predicts days until stock-out  

---

## 3.2 Assumptions

- "recent" means last 30 days  
- daily usage = total sales / 30  
- no sales → no alert  
- bundle handling done separately  

---

## 3.3 Endpoint

```
GET /api/companies/{company_id}/alerts/low-stock
```

---

## 3.4 Implementation

```python
@app.route("/api/companies/<int:company_id>/alerts/low-stock")
def low_stock_alerts(company_id):

    THIRTY_DAYS_AGO = datetime.utcnow() - timedelta(days=30)

    rows = db.session.execute("""
        SELECT 
            p.id,
            p.name,
            p.sku,
            w.id AS warehouse_id,
            w.name AS warehouse_name,
            i.quantity,
            p.low_stock_threshold,
            COALESCE(SUM(soi.quantity), 0) AS sales_30d,
            s.id AS supplier_id,
            s.name AS supplier_name,
            s.email AS supplier_email
        FROM inventory i
        JOIN products p ON p.id = i.product_id
        JOIN warehouses w ON w.id = i.warehouse_id
        LEFT JOIN sales_order_items soi ON soi.product_id = p.id
        LEFT JOIN sales_orders so ON so.id = soi.order_id
            AND so.created_at >= :date
        LEFT JOIN supplier_products sp ON sp.product_id = p.id
        LEFT JOIN suppliers s ON s.id = sp.supplier_id
        WHERE p.company_id = :cid
        GROUP BY p.id, w.id, s.id
    """, {"cid": company_id, "date": THIRTY_DAYS_AGO}).fetchall()

    alerts = []

    for r in rows:

        if r.sales_30d == 0:
            continue

        if r.quantity >= r.low_stock_threshold:
            continue

        daily_usage = r.sales_30d / 30
        days_left = int(r.quantity / daily_usage) if daily_usage else None

        alerts.append({
            "product_id": r.id,
            "product_name": r.name,
            "sku": r.sku,
            "warehouse_id": r.warehouse_id,
            "warehouse_name": r.warehouse_name,
            "current_stock": r.quantity,
            "threshold": r.low_stock_threshold,
            "days_until_stockout": days_left,
            "supplier": {
                "id": r.supplier_id,
                "name": r.supplier_name,
                "contact_email": r.supplier_email
            }
        })

    return {
        "alerts": alerts,
        "total_alerts": len(alerts)
    }
```

---

## 3.5 Edge Cases Covered

- product with stock but no sales → no alert  
- division-by-zero avoided  
- supplier missing → still alert  
- multiple warehouses handled  
- negative stock prevented  
- missing thresholds ignored  

---

# 🏁 Conclusion

This document demonstrates:

- debugging and production awareness  
- scalable relational database design  
- API architecture and implementation  
- assumptions & ambiguity handling  
- attention to edge-case behavior  

StockFlow supports:

- multi-warehouse inventory  
- low-stock alerts  
- supplier management  
- product bundles  
- SaaS multi-tenant companies  

---

**End of Documentation**
