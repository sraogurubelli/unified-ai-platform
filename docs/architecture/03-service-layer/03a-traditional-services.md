# Traditional Services

## Overview

The **Traditional Services** layer implements core SaaS business logic including user management, project management, document handling, webhooks, and analytics.

```
┌──────────────────────────────────────────────────────────────┐
│              Traditional Services Layer                       │
├──────────────┬───────────────┬──────────────────────────────┤
│ Auth Service │ Project Mgmt  │ Analytics Service             │
│ User Mgmt    │ Document Mgmt │ Webhook Service               │
│ Org Mgmt     │ Settings      │ Notification Service          │
└──────────────┴───────────────┴────────────────────────────────┘
```

---

## Authentication Service

### User Registration

```python
class AuthService:
    """Handle user authentication and registration."""

    async def register(
        self,
        email: str,
        password: str,
        display_name: str
    ) -> dict:
        """Register new user."""

        # Validate email format
        if not self._is_valid_email(email):
            raise ValidationError("Invalid email format")

        # Check if email already exists
        existing = await db.query(Principal).filter_by(email=email).first()
        if existing:
            raise ValidationError("Email already registered")

        # Hash password
        salt = bcrypt.gensalt()
        password_hash = bcrypt.hashpw(password.encode('utf-8'), salt)

        # Create principal
        principal = Principal(
            uid=f"user_{uuid.uuid4().hex[:12]}",
            email=email,
            display_name=display_name,
            principal_type=PrincipalType.USER,
            salt=password_hash.decode('utf-8')
        )

        await db.add(principal)

        # Create account for new user
        account = Account(
            uid=f"acct_{uuid.uuid4().hex[:12]}",
            name=f"{display_name}'s Account",
            subscription_tier=SubscriptionTier.FREE
        )

        await db.add(account)
        await db.commit()

        # Create membership (owner role)
        membership = Membership(
            principal_id=principal.id,
            resource_type="account",
            resource_id=account.uid,
            role=Role.OWNER
        )

        await db.add(membership)
        await db.commit()

        # Create JWT token
        token = auth_service._create_token(
            principal_id=principal.id,
            tenant_id=account.uid,
            email=email
        )

        return {
            "token": token,
            "principal": {
                "id": principal.uid,
                "email": principal.email,
                "display_name": principal.display_name
            },
            "tenant_id": account.uid
        }
```

---

## Organization & Project Management

### Project Service

```python
class ProjectService:
    """Manage projects within organizations."""

    async def create_project(
        self,
        tenant_id: str,
        organization_id: str,
        name: str,
        description: str,
        principal_id: int
    ) -> Project:
        """Create new project."""

        # Check quota
        has_quota = await quota_service.check_quota(
            tenant_id=tenant_id,
            quota_type=QuotaType.PROJECTS
        )

        if not has_quota:
            raise QuotaExceeded("Project quota exceeded")

        # Verify organization belongs to tenant
        org = await db.query(Organization).filter_by(
            uid=organization_id
        ).first()

        if not org or org.account.uid != tenant_id:
            raise NotFoundError("Organization not found")

        # Create project
        project = Project(
            uid=f"proj_{uuid.uuid4().hex[:12]}",
            organization_id=org.id,
            name=name,
            description=description
        )

        await db.add(project)
        await db.commit()

        # Audit log
        await audit_service.log_action(
            tenant_id=tenant_id,
            principal_id=principal_id,
            action="project.create",
            resource_type="project",
            resource_id=project.uid,
            metadata={"name": name}
        )

        return project

    async def list_projects(
        self,
        tenant_id: str,
        organization_id: str = None,
        skip: int = 0,
        limit: int = 100
    ) -> list[Project]:
        """List projects for tenant."""

        query = db.query(Project).join(Organization).join(Account).filter(
            Account.uid == tenant_id
        )

        if organization_id:
            query = query.filter(Organization.uid == organization_id)

        projects = await query.offset(skip).limit(limit).all()

        return projects
```

---

## Document Management

### Document Service

```python
class DocumentService:
    """Manage document upload, storage, and retrieval."""

    async def upload_document(
        self,
        tenant_id: str,
        project_id: str,
        file: UploadFile,
        principal_id: int
    ) -> Document:
        """Upload document to project."""

        # Check storage quota
        file_size = await self._get_file_size(file)
        has_quota = await quota_service.check_quota(
            tenant_id=tenant_id,
            quota_type=QuotaType.STORAGE_GB
        )

        if not has_quota:
            raise QuotaExceeded("Storage quota exceeded")

        # Upload to S3
        s3_key = f"{tenant_id}/{project_id}/{uuid.uuid4().hex}/{file.filename}"
        await s3_client.upload_fileobj(file.file, BUCKET_NAME, s3_key)

        # Create document record
        document = Document(
            uid=f"doc_{uuid.uuid4().hex[:12]}",
            project_id=project_id,
            tenant_id=tenant_id,
            filename=file.filename,
            s3_key=s3_key,
            size_bytes=file_size,
            content_type=file.content_type,
            uploaded_by=principal_id
        )

        await db.add(document)
        await db.commit()

        # Trigger async processing (embedding generation)
        await event_bus.publish("document.uploaded", {
            "tenant_id": tenant_id,
            "document_id": document.uid,
            "s3_key": s3_key
        })

        return document

    async def get_document(
        self,
        tenant_id: str,
        document_id: str
    ) -> Document:
        """Retrieve document metadata."""

        document = await db.query(Document).filter_by(
            uid=document_id,
            tenant_id=tenant_id
        ).first()

        if not document:
            raise NotFoundError("Document not found")

        return document

    async def download_document(
        self,
        tenant_id: str,
        document_id: str
    ) -> str:
        """Generate presigned URL for document download."""

        document = await self.get_document(tenant_id, document_id)

        # Generate presigned URL (valid for 1 hour)
        url = s3_client.generate_presigned_url(
            'get_object',
            Params={'Bucket': BUCKET_NAME, 'Key': document.s3_key},
            ExpiresIn=3600
        )

        return url
```

---

## Webhook Service

### Webhook Management

```python
class WebhookService:
    """Manage webhooks for event notifications."""

    async def create_webhook(
        self,
        tenant_id: str,
        url: str,
        events: list[str],
        secret: str = None
    ) -> Webhook:
        """Register webhook endpoint."""

        # Validate URL
        if not self._is_valid_url(url):
            raise ValidationError("Invalid webhook URL")

        # Create webhook
        webhook = Webhook(
            uid=f"hook_{uuid.uuid4().hex[:12]}",
            tenant_id=tenant_id,
            url=url,
            events=events,
            secret=secret or secrets.token_urlsafe(32),
            active=True
        )

        await db.add(webhook)
        await db.commit()

        return webhook

    async def trigger_webhook(
        self,
        tenant_id: str,
        event: str,
        payload: dict
    ):
        """Trigger webhooks for event."""

        # Get active webhooks for event
        webhooks = await db.query(Webhook).filter(
            Webhook.tenant_id == tenant_id,
            Webhook.active == True,
            Webhook.events.contains([event])
        ).all()

        for webhook in webhooks:
            # Send async webhook request
            await self._send_webhook(webhook, event, payload)

    async def _send_webhook(
        self,
        webhook: Webhook,
        event: str,
        payload: dict
    ):
        """Send HTTP POST to webhook URL."""

        # Create signature
        signature = hmac.new(
            webhook.secret.encode('utf-8'),
            json.dumps(payload).encode('utf-8'),
            hashlib.sha256
        ).hexdigest()

        headers = {
            "Content-Type": "application/json",
            "X-Webhook-Event": event,
            "X-Webhook-Signature": signature
        }

        try:
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    webhook.url,
                    json=payload,
                    headers=headers,
                    timeout=10.0
                )

                response.raise_for_status()

                # Log success
                logger.info(
                    "webhook_delivered",
                    webhook_id=webhook.uid,
                    event=event,
                    status_code=response.status_code
                )

        except Exception as e:
            # Log failure
            logger.error(
                "webhook_failed",
                webhook_id=webhook.uid,
                event=event,
                error=str(e)
            )

            # Retry logic (use Celery or similar)
            await self._schedule_retry(webhook, event, payload)
```

---

## Analytics Service

### Usage Analytics

```python
class AnalyticsService:
    """Provide usage analytics and reporting."""

    async def get_project_stats(
        self,
        tenant_id: str,
        project_id: str,
        date_range: str = "30d"
    ) -> dict:
        """Get project usage statistics."""

        # Parse date range
        if date_range == "7d":
            start_date = datetime.utcnow() - timedelta(days=7)
        elif date_range == "30d":
            start_date = datetime.utcnow() - timedelta(days=30)
        else:
            start_date = datetime.utcnow() - timedelta(days=90)

        # Query analytics database (ClickHouse)
        stats = await clickhouse.query("""
            SELECT
                date_trunc('day', timestamp) AS day,
                COUNT(*) AS api_calls,
                COUNT(DISTINCT principal_id) AS active_users,
                AVG(latency_ms) AS avg_latency
            FROM api_requests
            WHERE tenant_id = :tenant_id
              AND project_id = :project_id
              AND timestamp >= :start_date
            GROUP BY day
            ORDER BY day
        """, {
            "tenant_id": tenant_id,
            "project_id": project_id,
            "start_date": start_date
        })

        return {
            "project_id": project_id,
            "date_range": date_range,
            "time_series": [
                {
                    "date": row[0].isoformat(),
                    "api_calls": row[1],
                    "active_users": row[2],
                    "avg_latency_ms": float(row[3])
                }
                for row in stats
            ]
        }

    async def get_tenant_dashboard(
        self,
        tenant_id: str
    ) -> dict:
        """Get tenant-wide dashboard metrics."""

        # Query multiple metrics in parallel
        total_api_calls, total_storage, active_projects, monthly_cost = await asyncio.gather(
            self._get_total_api_calls(tenant_id),
            self._get_total_storage(tenant_id),
            self._get_active_projects(tenant_id),
            self._get_monthly_cost(tenant_id)
        )

        return {
            "tenant_id": tenant_id,
            "total_api_calls": total_api_calls,
            "total_storage_gb": total_storage,
            "active_projects": active_projects,
            "monthly_cost_usd": monthly_cost
        }
```

---

## Notification Service

### Email Notifications

```python
class NotificationService:
    """Send notifications to users."""

    async def send_email(
        self,
        to_email: str,
        subject: str,
        body: str,
        template: str = None
    ):
        """Send email notification."""

        # Use email provider (SendGrid, AWS SES, etc.)
        message = {
            "to": to_email,
            "from": "noreply@cortex-ai.com",
            "subject": subject,
            "html": body
        }

        if template:
            # Use email template
            message["template_id"] = template

        try:
            await sendgrid_client.send(message)

            logger.info(
                "email_sent",
                to_email=to_email,
                subject=subject
            )

        except Exception as e:
            logger.error(
                "email_failed",
                to_email=to_email,
                error=str(e)
            )

    async def notify_quota_exceeded(
        self,
        tenant_id: str,
        quota_type: QuotaType
    ):
        """Notify user when quota is exceeded."""

        # Get account owner
        account = await db.query(Account).filter_by(uid=tenant_id).first()
        owner_membership = await db.query(Membership).filter_by(
            resource_type="account",
            resource_id=account.uid,
            role=Role.OWNER
        ).first()

        principal = await db.query(Principal).filter_by(
            id=owner_membership.principal_id
        ).first()

        # Send notification
        await self.send_email(
            to_email=principal.email,
            subject=f"Quota Exceeded: {quota_type.value}",
            body=f"""
                <h1>Quota Exceeded</h1>
                <p>Your {quota_type.value} quota has been exceeded.</p>
                <p>Upgrade your plan to continue using the service.</p>
            """,
            template="quota_exceeded"
        )
```

---

**Next**: [AI Services](./03b-ai-services.md) | [Service Integration](./03c-service-integration.md) | [Back to Overview](../00-parallel-stacks-overview.md)
