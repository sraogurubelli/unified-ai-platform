# Traditional Control Plane

## Overview

The **Traditional Control Plane** manages policies and governance specific to SaaS-style operations including API quotas, billing, audit logging, and compliance.

```
┌──────────────────────────────────────────────────────────────┐
│              Traditional Control Plane                        │
├──────────────┬───────────────┬──────────────────────────────┤
│ API Quotas   │ Billing       │ Audit & Compliance            │
│              │               │                                │
│ • API calls  │ • Usage       │ • Action logs                  │
│ • Storage    │ • Invoicing   │ • Compliance                   │
│ • Rate limits│ • Plans       │ • Data retention               │
└──────────────┴───────────────┴────────────────────────────────┘
```

---

## API Quotas & Limits

### Quota Types

```python
class QuotaType(str, Enum):
    API_CALLS_MONTHLY = "api_calls_monthly"
    STORAGE_GB = "storage_gb"
    PROJECTS = "projects"
    USERS = "users"
    CONCURRENT_REQUESTS = "concurrent_requests"

class SubscriptionTier(str, Enum):
    FREE = "free"
    PRO = "pro"
    TEAM = "team"
    ENTERPRISE = "enterprise"

# Quota limits by tier
QUOTA_LIMITS = {
    SubscriptionTier.FREE: {
        QuotaType.API_CALLS_MONTHLY: 10_000,
        QuotaType.STORAGE_GB: 1,
        QuotaType.PROJECTS: 3,
        QuotaType.USERS: 1,
        QuotaType.CONCURRENT_REQUESTS: 5,
    },
    SubscriptionTier.PRO: {
        QuotaType.API_CALLS_MONTHLY: 100_000,
        QuotaType.STORAGE_GB: 10,
        QuotaType.PROJECTS: 10,
        QuotaType.USERS: 5,
        QuotaType.CONCURRENT_REQUESTS: 20,
    },
    SubscriptionTier.TEAM: {
        QuotaType.API_CALLS_MONTHLY: 1_000_000,
        QuotaType.STORAGE_GB: 100,
        QuotaType.PROJECTS: 50,
        QuotaType.USERS: 25,
        QuotaType.CONCURRENT_REQUESTS: 100,
    },
    SubscriptionTier.ENTERPRISE: {
        QuotaType.API_CALLS_MONTHLY: -1,  # Unlimited
        QuotaType.STORAGE_GB: -1,
        QuotaType.PROJECTS: -1,
        QuotaType.USERS: -1,
        QuotaType.CONCURRENT_REQUESTS: 500,
    },
}
```

### Quota Enforcement

```python
class QuotaService:
    """Enforce subscription quotas."""

    async def check_quota(
        self,
        tenant_id: str,
        quota_type: QuotaType
    ) -> bool:
        """Check if tenant has remaining quota."""

        # Get account tier
        account = await db.query(Account).filter_by(uid=tenant_id).first()
        tier = account.subscription_tier

        # Get quota limit
        limit = QUOTA_LIMITS[tier][quota_type]

        # Unlimited quota
        if limit == -1:
            return True

        # Get current usage
        usage = await self._get_usage(tenant_id, quota_type)

        return usage < limit

    async def _get_usage(self, tenant_id: str, quota_type: QuotaType) -> int:
        """Get current usage for quota type."""

        if quota_type == QuotaType.API_CALLS_MONTHLY:
            # Query analytics database
            result = await clickhouse.query("""
                SELECT COUNT(*) FROM api_requests
                WHERE tenant_id = :tenant_id
                  AND timestamp >= date_trunc('month', NOW())
            """, {"tenant_id": tenant_id})
            return result[0][0]

        elif quota_type == QuotaType.STORAGE_GB:
            # Query storage usage
            result = await db.query("""
                SELECT SUM(size_bytes) FROM documents
                WHERE tenant_id = :tenant_id
            """, {"tenant_id": tenant_id})
            return (result[0][0] or 0) / (1024**3)  # Convert to GB

        elif quota_type == QuotaType.PROJECTS:
            # Count projects
            return await db.query(Project).filter_by(
                tenant_id=tenant_id
            ).count()

        # ... other quota types

# Usage in API
@app.post("/api/v1/projects")
async def create_project(request: Request, ...):
    """Create project with quota check."""

    # Check quota
    has_quota = await quota_service.check_quota(
        tenant_id=request.state.tenant_id,
        quota_type=QuotaType.PROJECTS
    )

    if not has_quota:
        raise HTTPException(
            status_code=403,
            detail="Project quota exceeded. Upgrade your plan."
        )

    ...
```

---

## Billing & Usage Tracking

### Usage Events

```python
class UsageEvent(Base):
    """Track billable usage events."""
    __tablename__ = "usage_events"

    id = Column(Integer, primary_key=True)
    tenant_id = Column(String(255), nullable=False)
    event_type = Column(String(50), nullable=False)  # 'api_call', 'storage'
    quantity = Column(Integer, nullable=False)
    unit_price = Column(Numeric(10, 4), nullable=False)
    total_cost = Column(Numeric(10, 4), nullable=False)
    timestamp = Column(DateTime(timezone=True), server_default=func.now())

    __table_args__ = (
        Index("idx_usage_tenant_timestamp", "tenant_id", "timestamp"),
    )
```

### Invoice Generation

```python
class BillingService:
    """Manage billing and invoices."""

    async def generate_monthly_invoice(
        self,
        tenant_id: str,
        year_month: str
    ) -> Invoice:
        """Generate invoice for month."""

        # Query usage events
        usage_events = await db.query(UsageEvent).filter(
            UsageEvent.tenant_id == tenant_id,
            func.date_trunc('month', UsageEvent.timestamp) == year_month
        ).all()

        # Calculate totals
        line_items = defaultdict(lambda: {"quantity": 0, "cost": 0.0})

        for event in usage_events:
            line_items[event.event_type]["quantity"] += event.quantity
            line_items[event.event_type]["cost"] += float(event.total_cost)

        # Create invoice
        invoice = Invoice(
            tenant_id=tenant_id,
            year_month=year_month,
            line_items=dict(line_items),
            total=sum(item["cost"] for item in line_items.values()),
            status="pending"
        )

        await db.add(invoice)
        await db.commit()

        return invoice
```

---

## Audit Logging

### Audit Events

```python
class AuditLog(Base):
    """Audit trail for compliance."""
    __tablename__ = "audit_logs"

    id = Column(Integer, primary_key=True)
    tenant_id = Column(String(255), nullable=False)
    principal_id = Column(Integer, ForeignKey("principals.id"))
    action = Column(String(100), nullable=False)  # 'project.create', 'user.delete'
    resource_type = Column(String(50), nullable=False)
    resource_id = Column(String(255), nullable=False)
    metadata = Column(JSON, nullable=True)  # Additional context
    ip_address = Column(String(45), nullable=True)
    user_agent = Column(String(500), nullable=True)
    timestamp = Column(DateTime(timezone=True), server_default=func.now())

    __table_args__ = (
        Index("idx_audit_tenant_timestamp", "tenant_id", "timestamp"),
        Index("idx_audit_principal", "principal_id"),
    )
```

### Audit Logging Service

```python
class AuditService:
    """Centralized audit logging."""

    async def log_action(
        self,
        tenant_id: str,
        principal_id: int,
        action: str,
        resource_type: str,
        resource_id: str,
        metadata: dict = None,
        request: Request = None
    ):
        """Log auditable action."""

        audit_log = AuditLog(
            tenant_id=tenant_id,
            principal_id=principal_id,
            action=action,
            resource_type=resource_type,
            resource_id=resource_id,
            metadata=metadata,
            ip_address=request.client.host if request else None,
            user_agent=request.headers.get("User-Agent") if request else None
        )

        await db.add(audit_log)
        await db.commit()

# Usage in API
@app.delete("/api/v1/projects/{project_id}")
async def delete_project(request: Request, project_id: str):
    """Delete project with audit logging."""

    # Delete project
    await db.delete(Project).filter_by(uid=project_id)

    # Log action
    await audit_service.log_action(
        tenant_id=request.state.tenant_id,
        principal_id=request.state.principal_id,
        action="project.delete",
        resource_type="project",
        resource_id=project_id,
        metadata={"reason": "user_request"},
        request=request
    )
```

---

## Compliance & Data Retention

### Data Retention Policies

```python
class RetentionPolicy(Base):
    """Data retention configuration."""
    __tablename__ = "retention_policies"

    id = Column(Integer, primary_key=True)
    tenant_id = Column(String(255), nullable=False)
    resource_type = Column(String(50), nullable=False)
    retention_days = Column(Integer, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())

class RetentionService:
    """Enforce data retention policies."""

    async def cleanup_old_data(self):
        """Delete data beyond retention period."""

        policies = await db.query(RetentionPolicy).all()

        for policy in policies:
            cutoff = datetime.utcnow() - timedelta(days=policy.retention_days)

            if policy.resource_type == "conversation":
                # Delete old conversations
                await db.execute("""
                    DELETE FROM conversations
                    WHERE tenant_id = :tenant_id
                      AND created_at < :cutoff
                """, {
                    "tenant_id": policy.tenant_id,
                    "cutoff": cutoff
                })

            elif policy.resource_type == "audit_log":
                # Archive old audit logs to S3 before deletion
                old_logs = await db.query(AuditLog).filter(
                    AuditLog.tenant_id == policy.tenant_id,
                    AuditLog.timestamp < cutoff
                ).all()

                # Archive to S3
                await self._archive_to_s3(old_logs, policy.tenant_id)

                # Delete from database
                await db.delete(AuditLog).filter(
                    AuditLog.tenant_id == policy.tenant_id,
                    AuditLog.timestamp < cutoff
                )
```

---

## Subscription Management

### Subscription Tiers

```python
class Subscription(Base):
    """Account subscription."""
    __tablename__ = "subscriptions"

    id = Column(Integer, primary_key=True)
    account_id = Column(Integer, ForeignKey("accounts.id"), nullable=False)
    tier = Column(Enum(SubscriptionTier), nullable=False)
    billing_period = Column(String(20), nullable=False)  # 'monthly', 'annual'
    starts_at = Column(DateTime(timezone=True), nullable=False)
    expires_at = Column(DateTime(timezone=True), nullable=True)
    auto_renew = Column(Boolean, default=True)
    payment_method_id = Column(String(255), nullable=True)
    status = Column(String(20), default="active")  # 'active', 'canceled', 'expired'
```

### Upgrade/Downgrade

```python
class SubscriptionService:
    """Manage subscription changes."""

    async def upgrade_subscription(
        self,
        tenant_id: str,
        new_tier: SubscriptionTier
    ):
        """Upgrade subscription tier."""

        account = await db.query(Account).filter_by(uid=tenant_id).first()
        subscription = await db.query(Subscription).filter_by(
            account_id=account.id
        ).first()

        # Calculate prorated cost
        prorated_cost = await self._calculate_prorated_cost(
            current_tier=subscription.tier,
            new_tier=new_tier,
            days_remaining=self._days_until_renewal(subscription)
        )

        # Charge prorated amount
        await payment_service.charge(
            account_id=account.id,
            amount=prorated_cost,
            description=f"Upgrade to {new_tier}"
        )

        # Update subscription
        subscription.tier = new_tier
        account.subscription_tier = new_tier
        await db.commit()

        # Log audit event
        await audit_service.log_action(
            tenant_id=tenant_id,
            principal_id=None,  # System action
            action="subscription.upgrade",
            resource_type="subscription",
            resource_id=str(subscription.id),
            metadata={"from_tier": subscription.tier, "to_tier": new_tier}
        )
```

---

**Next**: [AI Control Plane](./02b-ai-control-plane.md) | [Shared Control Plane](./02c-control-plane-shared.md) | [Back to Overview](../00-parallel-stacks-overview.md)
