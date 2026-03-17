---
marp: true
theme: default
paginate: true
size: 16:9
style: |
  @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap');

  section {
    background: linear-gradient(135deg, #ffffff 0%, #fff8f0 100%);
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    font-size: 28px;
    color: #1a1a1a;
    padding: 60px 80px;
    line-height: 1.6;
  }

  h1 {
    color: #d97757;
    font-size: 52px;
    font-weight: 700;
    letter-spacing: -0.02em;
    margin-bottom: 24px;
    line-height: 1.2;
  }

  h2 {
    color: #c85a3c;
    font-size: 36px;
    font-weight: 600;
    border-left: 4px solid #d97757;
    padding-left: 20px;
    margin-top: 32px;
    margin-bottom: 20px;
  }

  h3 {
    color: #8b5a44;
    font-size: 28px;
    font-weight: 600;
    margin-top: 24px;
  }

  strong {
    color: #c85a3c;
    font-weight: 600;
  }

  code {
    background: #fff4e6;
    color: #c85a3c;
    padding: 2px 8px;
    border-radius: 4px;
    font-family: 'SF Mono', 'Fira Code', 'Courier New', monospace;
    font-size: 0.9em;
  }

  .columns {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 32px;
  }

  .impact {
    color: #dc2626;
    font-weight: 800;
    font-size: 40px;
  }

  .success {
    color: #16a34a;
    font-weight: 800;
    font-size: 40px;
  }

  /* Two-column box layout */
  .box-container {
    display: grid;
    grid-template-columns: 1fr auto 1fr;
    gap: 40px;
    align-items: center;
    margin: 40px 0;
  }

  .box {
    background: #dbeafe;
    border-radius: 12px;
    padding: 32px;
    min-height: 380px;
  }

  .box.orange {
    background: #fed7aa;
  }

  .box h3 {
    color: #1e3a8a;
    font-size: 32px;
    font-weight: 600;
    margin-top: 0;
    margin-bottom: 24px;
    text-align: center;
    border: none;
    padding: 0;
  }

  .box.orange h3 {
    color: #8b5a44;
  }

  .box ul {
    list-style: none;
    padding: 0;
    margin: 0;
  }

  .box li {
    margin: 20px 0;
    font-size: 26px;
    line-height: 1.4;
  }

  .arrow {
    font-size: 80px;
    color: #60a5fa;
    text-align: center;
  }

  .tagline {
    background: #dbeafe;
    padding: 24px 40px;
    border-radius: 12px;
    text-align: center;
    font-size: 30px;
    font-weight: 600;
    color: #1e3a8a;
    margin-top: 40px;
  }

  /* Title slide special styling */
  section:nth-child(1) h1 {
    font-size: 64px;
    background: linear-gradient(135deg, #d97757 0%, #c85a3c 100%);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  section::after {
    color: #c85a3c;
    font-weight: 500;
    font-size: 18px;
  }
---

# TallyGo

## AI Revenue Automation Platform for Logistics

**Solving 10%+ Revenue Leakage Through Vertical AI + Knowledge Graphs**

March 16, 2025

---

## Today's Roadmap

**1. The Problem**
   Revenue loss in fragmented operations

**2. The Solution**
   Proprietary Billing Graph + Vertical AI

**3. The Platform**
   Event-driven architecture at scale

**4. Execution**
   30-Day Pilot → $2M ARR

---

# 1️⃣ The Problem

## Revenue Loss Starts in Operations

---

<div class="box-container">

<div class="box">

### Fragmented Operations

- 🗄️ **TMS**
- 📧 **Emails**
- 📄 **PDFs / PODs**
- 📋 **Contracts**
- 💰 **Carrier invoices**
- 🔗 **AP portals**

</div>

<div class="arrow">→</div>

<div class="box orange">

### Business Impact

- 🚫 **Invoice rejections**
- 💸 **Missed charges**
- ✅ **Manual reconciliation**
- ⏰ **Slow collections**

</div>

</div>

<div class="tagline">
Billing fails when operational evidence is disconnected
</div>

---

## The Opportunity

**Mid-size 3PL (Industry Benchmark)**:

- 📊 **Invoice rejection rate**: 50%
- 💰 **Revenue leakage**: 10%+ ($1M annually)
- ⏱️ **Days Sales Outstanding**: 100+ days
- 👥 **Manual billing effort**: 200 hours/month

---

# 2️⃣ The Solution

---

## Knowledge Graph: AI + Graph Reasoning

![Knowledge Graph Architecture](images/KnowledgeGraph_Light.png)

---

## Example: Detention Charge Detection

<div style="text-align: center; margin-top: 10px;">
<img src="images/KnowledgeGraphExample.png" style="max-width: 75%; max-height: 450px; object-fit: contain;" />
</div>

---

## Vertical AI Advantage

<div class="columns">

<div style="background: linear-gradient(135deg, #fee2e2 0%, #fecaca 100%); padding: 35px; border-radius: 16px; border: 3px solid #dc2626; min-height: 380px;">
<div style="text-align: center; font-size: 44px; margin-bottom: 15px;">❌</div>
<h3 style="text-align: center; color: #dc2626; font-size: 32px; margin-top: 0; border: none; padding: 0;">Generic AI</h3>
<div style="margin-top: 25px; font-size: 22px; line-height: 1.7;">
<div style="margin: 14px 0;">📊 <strong>Accuracy</strong>: 80-85%</div>
<div style="margin: 14px 0;">💰 <strong>Cost</strong>: $0.03/invoice</div>
<div style="margin: 14px 0;">📚 <strong>Training</strong>: General internet</div>
<div style="margin: 14px 0; color: #dc2626; font-weight: 600;">❌ Not audit-grade</div>
</div>
</div>

<div style="background: linear-gradient(135deg, #dcfce7 0%, #bbf7d0 100%); padding: 35px; border-radius: 16px; border: 3px solid #16a34a; min-height: 380px;">
<div style="text-align: center; font-size: 44px; margin-bottom: 15px;">✅</div>
<h3 style="text-align: center; color: #16a34a; font-size: 32px; margin-top: 0; border: none; padding: 0;">TallyGo AI</h3>
<div style="margin-top: 25px; font-size: 22px; line-height: 1.7;">
<div style="margin: 14px 0;">📊 <strong>Accuracy</strong>: <span style="color: #16a34a; font-weight: 800; font-size: 26px;">95%+</span></div>
<div style="margin: 14px 0;">💰 <strong>Cost</strong>: <span style="color: #16a34a; font-weight: 800; font-size: 26px;">$0.003/invoice</span></div>
<div style="margin: 14px 0;">📚 <strong>Training</strong>: 10K+ logistics invoices</div>
<div style="margin: 14px 0; color: #16a34a; font-weight: 600;">✅ Audit-grade precision</div>
</div>
</div>

</div>

<div class="tagline">
10-15% accuracy lift = CFO trust | 90% cost reduction | 18-24 months to replicate
</div>

---

# 3️⃣ The Platform

---

## Connect Operations → Billing → Revenue

![Platform Architecture](images/ConnectPlatform.png)

---

## Technology Architecture

![Technology Stack](images/final_architecture_slide.png)

---

# 4️⃣ Execution

---

## Build → Scale → Learn

![Execution Roadmap](images/execution_slide_with_intelligence.png)

---

## Team & Customer Success

![Team and Customer Strategy](images/execution_slide_short.png)

