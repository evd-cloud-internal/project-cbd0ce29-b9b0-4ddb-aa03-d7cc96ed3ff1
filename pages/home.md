---
name: Home
assetId: 2e150e60-f1b1-46d2-b75e-afedb1064443
type: page
---

# Property Brokers – Booking overview

---

## KPIs

```sql property_brokers_kpis
SELECT
  COUNT(*) AS total_bookings,
  countIf(status = 1) AS active_bookings,
  countIf(status = 11) AS completed_bookings,
  countIf(creation_time >= toDate(date_trunc('month', today()))) AS created_this_month
FROM kepla_bookings
WHERE playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
  AND is_deleted = false
```

{% row %}
    {% big_value data="property_brokers_kpis" value="total_bookings" title="Total bookings" fmt="num" /%}
    {% big_value data="property_brokers_kpis" value="active_bookings" title="Active" fmt="num" /%}
    {% big_value data="property_brokers_kpis" value="completed_bookings" title="Completed" fmt="num" /%}
    <!-- {% big_value data="property_brokers_kpis" value="created_this_month" title="Created this month" fmt="num" /%} -->
{% /row %}

## Bookings by week

```sql bookings_by_week
SELECT
  date_trunc('week', creation_time)::date AS week,
  COUNT(*) AS bookings
FROM kepla_bookings
WHERE playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
  AND is_deleted = false
GROUP BY week
ORDER BY week
```

{% bar_chart data="bookings_by_week" x="week" y="bookings" /%}

---

## Bookings by status

```sql bookings_by_status
SELECT
  status,
  CASE status
    WHEN 12 THEN 'Draft'
    WHEN 3 THEN 'Pending Approval'
    WHEN 14 THEN 'Approved'
    WHEN 4 THEN 'Creating Resources'
    WHEN 5 THEN 'Launching Resources'
    WHEN 1 THEN 'Active'
    WHEN 11 THEN 'Completed'
    WHEN 2 THEN 'Failed'
    WHEN 15 THEN 'Cancelled'
    WHEN 8 THEN 'Scheduled'
    ELSE 'Other'
  END AS status_label,
  COUNT(*) AS count
FROM kepla_bookings
WHERE playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
  AND is_deleted = false
GROUP BY status
ORDER BY count DESC
```

{% bar_chart data="bookings_by_status" x="status_label" y="count" /%}

---

## Bookings per month

```sql bookings_per_month
SELECT
  date_trunc('month', creation_time)::date AS month,
  COUNT(*) AS created,
  countIf(status = 11) AS completed,
  countIf(status = 1) AS active
FROM kepla_bookings
WHERE playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
  AND is_deleted = false
GROUP BY date_trunc('month', creation_time)::date
ORDER BY month
```

{% line_chart data="bookings_per_month" x="month" y=["created", "completed", "active"] /%}

---

## Bookings by promo code

```sql bookings_by_promo_code
SELECT
  CASE
    WHEN promo_code = 'AUTO-JUST-SOLD' THEN 'Auto Just Sold'
    WHEN promo_code LIKE '%VENDOR_LIFE%' THEN 'Vendor Lifestyle'
    WHEN promo_code LIKE '%VENDOR_RURA%' THEN 'Vendor Rural'
    WHEN promo_code LIKE '%VENDOR_COMM%' THEN 'Vendor Commercial'
    WHEN promo_code LIKE '%VENDOR_EXTEND%' THEN 'Vendor Extended'
    WHEN promo_code LIKE '%VENDOR%' THEN 'Vendor'
    WHEN promo_code LIKE '%SOLD%' THEN 'Sold'
    ELSE 'Other'
  END AS product,
  CASE
    WHEN promo_code = 'AUTO-JUST-SOLD' THEN 'N/A'
    WHEN promo_code LIKE '%PLATINUM%' THEN 'Platinum'
    WHEN promo_code LIKE '%GOLD%' THEN 'Gold'
    WHEN promo_code LIKE '%SILVER%' THEN 'Silver'
    WHEN promo_code LIKE '%BRONZE%' THEN 'Bronze'
    WHEN promo_code LIKE '%ELITE%' THEN 'Elite'
    ELSE 'Other'
  END AS tier,
  COUNT(*) AS count
FROM (
  SELECT JSONExtractString(playbook_resource_inputs, 'listing', 'propertysuite_item_promo_code') AS promo_code
  FROM kepla_bookings
  WHERE playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
    AND is_deleted = false
) sub
WHERE promo_code != ''
GROUP BY product, tier
ORDER BY count DESC
```

{% bar_chart
    data="bookings_by_promo_code"
    x="product"
    y="count"
    series="tier"
    title="Product × Tier breakdown"
    series_order=["Platinum", "Gold", "Silver", "Bronze", "Elite", "N/A"]
    order="count desc"
    chart_options={
        series_colors={
            "Platinum"="#6366f1"
            "Gold"="#eab308"
            "Silver"="#94a3b8"
            "Bronze"="#d97706"
            "Elite"="#ec4899"
            "N/A"="#a3a3a3"
        }
    }
/%}

```sql bookings_by_product
SELECT
  CASE
    WHEN promo_code = 'AUTO-JUST-SOLD' THEN 'Auto Just Sold'
    WHEN promo_code LIKE '%VENDOR_LIFE%' THEN 'Vendor Lifestyle'
    WHEN promo_code LIKE '%VENDOR_RURA%' THEN 'Vendor Rural'
    WHEN promo_code LIKE '%VENDOR_COMM%' THEN 'Vendor Commercial'
    WHEN promo_code LIKE '%VENDOR_EXTEND%' THEN 'Vendor Extended'
    WHEN promo_code LIKE '%VENDOR%' THEN 'Vendor'
    WHEN promo_code LIKE '%SOLD%' THEN 'Sold'
    ELSE 'Other'
  END AS product,
  COUNT(*) AS count
FROM (
  SELECT JSONExtractString(playbook_resource_inputs, 'listing', 'propertysuite_item_promo_code') AS promo_code
  FROM kepla_bookings
  WHERE playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
    AND is_deleted = false
) sub
WHERE promo_code != ''
GROUP BY product
ORDER BY count DESC
```

{% bar_chart
    data="bookings_by_product"
    x="product"
    y="count"
    title="Total bookings by product"
    order="count desc"
/%}

---

## Most bookings by agent

```sql bookings_by_tenant
SELECT
  COALESCE(a.name, b.tenant_id) AS tenant,
  COUNT(*) AS bookings
FROM kepla_bookings b
LEFT JOIN kepla_accounts a ON b.tenant_id = a.account_tenant_id AND a.is_deleted = false
WHERE b.playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
  AND b.is_deleted = false
GROUP BY tenant
ORDER BY bookings DESC
LIMIT 50
```

```sql bookings_by_tenant_product
SELECT tenant, product, bookings, total_bookings
FROM (
  SELECT
    COALESCE(a.name, b.tenant_id) AS tenant,
    CASE
      WHEN promo_code = 'AUTO-JUST-SOLD' THEN 'Auto Just Sold'
      WHEN promo_code LIKE '%VENDOR_LIFE%' THEN 'Vendor Lifestyle'
      WHEN promo_code LIKE '%VENDOR_RURA%' THEN 'Vendor Rural'
      WHEN promo_code LIKE '%VENDOR_COMM%' THEN 'Vendor Commercial'
      WHEN promo_code LIKE '%VENDOR_EXTEND%' THEN 'Vendor Extended'
      WHEN promo_code LIKE '%VENDOR%' THEN 'Vendor'
      WHEN promo_code LIKE '%SOLD%' THEN 'Sold'
      WHEN promo_code = '' THEN 'No Promo Code'
      ELSE 'Other'
    END AS product,
    COUNT(*) AS bookings,
    SUM(COUNT(*)) OVER (PARTITION BY COALESCE(a.name, b.tenant_id)) AS total_bookings
  FROM kepla_bookings b
  LEFT JOIN kepla_accounts a ON b.tenant_id = a.account_tenant_id AND a.is_deleted = false
  CROSS JOIN (
    SELECT JSONExtractString(b2.playbook_resource_inputs, 'listing', 'propertysuite_item_promo_code') AS promo_code,
           b2.id
    FROM kepla_bookings b2
    WHERE b2.playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
      AND b2.is_deleted = false
  ) pc
  WHERE b.playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
    AND b.is_deleted = false
    AND b.id = pc.id
  GROUP BY tenant, product
  HAVING tenant IN (
    SELECT COALESCE(a2.name, b2.tenant_id)
    FROM kepla_bookings b2
    LEFT JOIN kepla_accounts a2 ON b2.tenant_id = a2.account_tenant_id AND a2.is_deleted = false
    WHERE b2.playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
      AND b2.is_deleted = false
    GROUP BY COALESCE(a2.name, b2.tenant_id)
    ORDER BY COUNT(*) DESC
    LIMIT 25
  )
) sub
ORDER BY total_bookings DESC, tenant, bookings DESC
```

{% horizontal_bar_chart
    data="bookings_by_tenant_product"
    x="bookings"
    y="tenant"
    series="product"
    title="Top 25 agents – product breakdown"
    order="total_bookings desc"
    height=600
/%}

{% table data="bookings_by_tenant" /%}

```sql agent_product_breakdown
SELECT
  COALESCE(a.name, b.tenant_id) AS agent,
  CASE
    WHEN promo_code = 'AUTO-JUST-SOLD' THEN 'Auto Just Sold'
    WHEN promo_code LIKE '%VENDOR_LIFE%' THEN 'Vendor Lifestyle'
    WHEN promo_code LIKE '%VENDOR_RURA%' THEN 'Vendor Rural'
    WHEN promo_code LIKE '%VENDOR_COMM%' THEN 'Vendor Commercial'
    WHEN promo_code LIKE '%VENDOR_EXTEND%' THEN 'Vendor Extended'
    WHEN promo_code LIKE '%VENDOR%' THEN 'Vendor'
    WHEN promo_code LIKE '%SOLD%' THEN 'Sold'
    WHEN promo_code = '' THEN 'No Promo Code'
    ELSE 'Other'
  END AS product,
  COUNT(*) AS bookings
FROM kepla_bookings b
LEFT JOIN kepla_accounts a ON b.tenant_id = a.account_tenant_id AND a.is_deleted = false
CROSS JOIN (
  SELECT JSONExtractString(b2.playbook_resource_inputs, 'listing', 'propertysuite_item_promo_code') AS promo_code,
         b2.id
  FROM kepla_bookings b2
  WHERE b2.playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
    AND b2.is_deleted = false
) pc
WHERE b.playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
  AND b.is_deleted = false
  AND b.id = pc.id
  AND promo_code != ''
GROUP BY agent, product
ORDER BY product, bookings DESC
```

### Most bookings by Vendor

{% table data="agent_product_breakdown" where="product = 'Vendor'" order="bookings desc" /%}

### Most bookings by Vendor Lifestyle

{% table data="agent_product_breakdown" where="product = 'Vendor Lifestyle'" order="bookings desc" /%}

### Most bookings by Vendor Rural

{% table data="agent_product_breakdown" where="product = 'Vendor Rural'" order="bookings desc" /%}

### Most bookings by Vendor Commercial

{% table data="agent_product_breakdown" where="product = 'Vendor Commercial'" order="bookings desc" /%}

### Most bookings by Vendor Extended

{% table data="agent_product_breakdown" where="product = 'Vendor Extended'" order="bookings desc" /%}

### Most bookings by Sold

{% table data="agent_product_breakdown" where="product = 'Sold'" order="bookings desc" /%}

### Most bookings by Auto Just Sold

{% table data="agent_product_breakdown" where="product = 'Auto Just Sold'" order="bookings desc" /%}

---

<!-- ## Leads by booking (not setup yet)

```sql leads_by_booking
SELECT
  b.id AS booking_id,
  b.display_name,
  CASE b.status
    WHEN 12 THEN 'Draft'
    WHEN 3 THEN 'Pending Approval'
    WHEN 14 THEN 'Approved'
    WHEN 4 THEN 'Creating Resources'
    WHEN 5 THEN 'Launching Resources'
    WHEN 1 THEN 'Active'
    WHEN 11 THEN 'Completed'
    WHEN 2 THEN 'Failed'
    WHEN 15 THEN 'Cancelled'
    WHEN 8 THEN 'Scheduled'
    ELSE 'Other'
  END AS status,
  COUNT(DISTINCT ml.external_lead_id) AS leads
FROM kepla_bookings b
LEFT JOIN kepla_meta_leads ml ON b.id = ml.booking_id
WHERE b.playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
  AND b.is_deleted = false
GROUP BY b.id, b.display_name, b.status
ORDER BY leads DESC, b.display_name
```

{% table data="leads_by_booking" page_size=20 /%}

--- -->

