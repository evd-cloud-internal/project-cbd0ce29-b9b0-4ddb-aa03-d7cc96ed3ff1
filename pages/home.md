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
    {% big_value data="property_brokers_kpis" value="created_this_month" title="Created this month" fmt="num" /%}
{% /row %}

## Bookings by day

```sql bookings_by_day
SELECT
  date_trunc('day', creation_time)::date AS date,
  COUNT(*) AS count
FROM kepla_bookings
WHERE playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
  AND is_deleted = false
GROUP BY date_trunc('day', creation_time)::date
ORDER BY date
```

{% bar_chart data="bookings_by_day" x="date" y="count" /%}

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

## 
```sql bookings_by_promo_code
SELECT
  CASE
    WHEN promo_code LIKE '%VENDOR_LIFE%' THEN 'Vendor Lifestyle'
    WHEN promo_code LIKE '%VENDOR_RURA%' THEN 'Vendor Rural'
    WHEN promo_code LIKE '%VENDOR_COMM%' THEN 'Vendor Commercial'
    WHEN promo_code LIKE '%VENDOR_EXTEND%' THEN 'Vendor Extended'
    WHEN promo_code LIKE '%VENDOR%' THEN 'Vendor'
    WHEN promo_code LIKE '%SOLD%' THEN 'Sold'
    ELSE 'Other'
  END AS promo_group,
  CASE
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
GROUP BY promo_group, tier
ORDER BY promo_group, count DESC
```

{% bar_chart data="bookings_by_promo_code" x="promo_group" y="count" series="tier" /%}

```sql bookings_by_promo_group
SELECT
  CASE
    WHEN promo_code LIKE '%VENDOR_LIFE%' THEN 'Vendor Lifestyle'
    WHEN promo_code LIKE '%VENDOR_RURA%' THEN 'Vendor Rural'
    WHEN promo_code LIKE '%VENDOR_COMM%' THEN 'Vendor Commercial'
    WHEN promo_code LIKE '%VENDOR_EXTEND%' THEN 'Vendor Extended'
    WHEN promo_code LIKE '%VENDOR%' THEN 'Vendor'
    WHEN promo_code LIKE '%SOLD%' THEN 'Sold'
    ELSE 'Other'
  END AS promo_group,
  COUNT(*) AS count
FROM (
  SELECT JSONExtractString(playbook_resource_inputs, 'listing', 'propertysuite_item_promo_code') AS promo_code
  FROM kepla_bookings
  WHERE playbook_id = '3769b0e1-fa6e-4a23-8d38-c04263eaf361'
    AND is_deleted = false
) sub
WHERE promo_code != ''
GROUP BY promo_group
ORDER BY count DESC
```

{% pie_chart data="bookings_by_promo_group" category="promo_group" value="count" /%}

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
<!-- 
{% horizontal_bar_chart data="bookings_by_tenant" x="bookings" y="tenant" /%} -->

{% table data="bookings_by_tenant" /%}

---

## Leads by booking (not setup yet)

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

---

