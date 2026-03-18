---
marp: true
theme: default
paginate: true
size: 16:9
style: |
  @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');

  section {
    background: linear-gradient(135deg, #ffffff 0%, #f8f9ff 100%);
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    font-size: 28px;
    color: #1a1a1a;
    padding: 60px 80px;
  }

  h1 {
    color: #2563eb;
    font-size: 52px;
    font-weight: 700;
    margin-bottom: 24px;
  }

  h2 {
    color: #1e40af;
    font-size: 36px;
    font-weight: 600;
    border-left: 4px solid #2563eb;
    padding-left: 20px;
    margin-top: 32px;
    margin-bottom: 20px;
  }

  h3 {
    color: #3b82f6;
    font-size: 28px;
    font-weight: 600;
    margin-top: 24px;
  }

  code {
    background: #eff6ff;
    color: #1e40af;
    padding: 2px 8px;
    border-radius: 4px;
    font-family: 'SF Mono', 'Fira Code', 'Courier New', monospace;
    font-size: 0.9em;
  }

  pre {
    background: #1e293b;
    color: #e2e8f0;
    padding: 24px;
    border-radius: 8px;
    font-size: 20px;
    line-height: 1.5;
  }

  pre code {
    background: transparent;
    color: #e2e8f0;
    padding: 0;
  }

  .columns {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 40px;
  }

  .box {
    background: #dbeafe;
    border-radius: 12px;
    padding: 24px;
    border: 2px solid #3b82f6;
  }

  .success {
    color: #16a34a;
    font-weight: 700;
  }

  .warning {
    color: #ea580c;
    font-weight: 700;
  }

  .metric {
    font-size: 42px;
    font-weight: 800;
    color: #2563eb;
  }

  ul {
    line-height: 1.8;
  }

  li {
    margin: 12px 0;
  }

  section::after {
    color: #64748b;
    font-weight: 500;
    font-size: 18px;
  }
---

# AI Agent Evaluations

## Tutorial for Harness Platform

**Ensuring Quality & Reliability at Scale**

---

## Today's Agenda

**1. Why Evaluate?**
   The business case for agent testing

**2. Evaluation Framework**
   Architecture and components

**3. Getting Started**
   Your first evaluation in 5 minutes

**4. Advanced Patterns**
   LLM-as-Judge, A/B testing, regression detection

**5. Best Practices**
   Production-grade evaluation strategies

---

# 1️⃣ Why Evaluate?

---

## The Problem: Agent Quality at Scale

<div class="columns">

<div>

### Without Evaluations

- 🎲 **Unpredictable outputs**
- 🐛 **Silent regressions**
- 💸 **Cost surprises**
- 🤷 **No performance baseline**
- 🔍 **Debugging = guesswork**

</div>

<div>

### With Evaluations

- ✅ **Consistent quality**
- 📊 **Metrics-driven decisions**
- 💰 **Cost optimization**
- 🎯 **Clear baselines**
- 🔬 **Scientific debugging**

</div>

</div>

---

## Real-World Impact

### Case Study: Customer Support Agent

**Before Evals:**
- 78% accuracy (unknown until production)
- $2,400/month in LLM costs
- 3 production incidents/month

**After Evals:**
- <span class="metric">93%</span> accuracy (measured pre-deploy)
- <span class="metric">$840/month</span> (65% cost reduction via caching)
- <span class="metric">0</span> production incidents (caught in testing)

---

## What Can We Evaluate?

**Quality Metrics:**
- Response accuracy (LLM-as-Judge)
- Task completion rate
- Tool usage correctness
- Response relevance

**Performance Metrics:**
- Latency (p50, p95, p99)
- Token usage and cost
- Cache hit rates
- Error rates

**Behavioral Metrics:**
- Consistency across runs
- Regression detection
- A/B comparison

---

# 2️⃣ Evaluation Framework

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  Evaluation Runner                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │ Test Cases │→ │   Agent    │→ │  Metrics   │   │
│  └────────────┘  └────────────┘  └────────────┘   │
└─────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────┐
│                 Evaluation Results                  │
│  • Quality Scores  • Cost Analysis  • Latency      │
│  • Pass/Fail       • Comparisons    • Regressions  │
└─────────────────────────────────────────────────────┘
```

---

## Components

### 1. Test Dataset
```python
{
  "test_id": "support_001",
  "input": "How do I reset my password?",
  "expected_output": "Password reset instructions...",
  "criteria": ["mentions reset link", "polite tone"],
  "metadata": {"category": "authentication"}
}
```

### 2. Evaluation Metrics
- **Exact Match**: Binary pass/fail
- **LLM-as-Judge**: Semantic evaluation
- **Custom Scorers**: Domain-specific logic

---

## Evaluation Flow

```
1. Load Test Dataset
   ↓
2. Run Agent on Each Test Case
   ↓
3. Collect Outputs + Metrics
   ↓
4. Score with Evaluators
   ↓
5. Generate Report
   ↓
6. Pass/Fail Decision
```

---

# 3️⃣ Getting Started

---

## Prerequisites

**1. Install the Evaluation SDK**
```bash
pip install harness-ai-evals
```

**2. Configure Your Agent**
```python
from cortex.orchestration import Agent

agent = Agent(
    model="claude-sonnet-4",
    tools=[search_tool, database_tool]
)
```

---

## Your First Evaluation (5 Minutes)

### Step 1: Create Test Dataset

```python
# tests/evals/datasets/support_queries.json
[
  {
    "id": "test_001",
    "input": "How do I reset my password?",
    "expected_behavior": "Provides password reset steps",
    "criteria": [
      "Mentions reset link or button",
      "Explains where to find it",
      "Polite and helpful tone"
    ]
  },
  {
    "id": "test_002",
    "input": "What's your refund policy?",
    "expected_behavior": "Explains refund policy accurately",
    "criteria": [
      "Mentions 30-day window",
      "Explains refund process",
      "Links to policy page"
    ]
  }
]
```

---

## Step 2: Create Evaluation Script

```python
# tests/evals/run_support_agent_eval.py
from harness_ai_evals import EvaluationRunner, LLMJudge
import json

# Load test dataset
with open("datasets/support_queries.json") as f:
    test_cases = json.load(f)

# Configure evaluator
evaluator = LLMJudge(
    model="claude-sonnet-4",
    criteria_prompt="""
    Evaluate if the response meets these criteria:
    {criteria}

    Score 0-10 and explain reasoning.
    """
)

# Run evaluation
runner = EvaluationRunner(
    agent=agent,
    test_cases=test_cases,
    evaluators=[evaluator]
)

results = runner.run()
print(results.summary())
```

---

## Step 3: Run and Review

```bash
$ python tests/evals/run_support_agent_eval.py

Running evaluations...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 2/2 100%

Evaluation Results:
┌─────────┬──────────┬────────┬─────────┐
│ Test ID │ Score    │ Passed │ Latency │
├─────────┼──────────┼────────┼─────────┤
│ test_001│ 9.5/10   │ ✓      │ 1.2s    │
│ test_002│ 8.0/10   │ ✓      │ 1.5s    │
└─────────┴──────────┴────────┴─────────┘

Overall: 2/2 passed (100%)
Avg Score: 8.75/10
Avg Latency: 1.35s
Total Cost: $0.042
```

---

## Step 4: Integrate into CI/CD

```yaml
# .harness/pipelines/agent-test.yaml
pipeline:
  name: Agent Evaluation
  stages:
    - stage:
        name: Run Evals
        type: Custom
        spec:
          execution:
            steps:
              - step:
                  type: Run
                  name: AI Agent Evaluation
                  spec:
                    command: |
                      python tests/evals/run_support_agent_eval.py

              - step:
                  type: Policy
                  name: Quality Gate
                  spec:
                    policySpec: |
                      - eval_score >= 8.0
                      - pass_rate >= 95%
                      - avg_latency < 2.0s
```

---

# 4️⃣ Advanced Patterns

---

## Pattern 1: LLM-as-Judge

**When to use:** Evaluating subjective qualities (tone, helpfulness, accuracy)

```python
from harness_ai_evals import LLMJudge

judge = LLMJudge(
    model="claude-sonnet-4",
    criteria_prompt="""
    Evaluate the assistant's response on:
    1. Accuracy (0-10): Factually correct information
    2. Helpfulness (0-10): Directly addresses user need
    3. Tone (0-10): Professional and friendly
    4. Completeness (0-10): Fully answers the question

    Provide scores and brief explanations.
    """,
    aggregation="weighted_average",
    weights={"accuracy": 0.4, "helpfulness": 0.3,
             "tone": 0.15, "completeness": 0.15}
)

results = judge.evaluate(test_cases, agent_responses)
```

---

## Pattern 2: A/B Testing

**When to use:** Comparing two agent configurations

```python
from harness_ai_evals import ABTest

# Define two configurations
config_a = Agent(model="claude-sonnet-4", temperature=0.7)
config_b = Agent(model="claude-opus-4", temperature=0.3)

# Run A/B test
ab_test = ABTest(
    variant_a=config_a,
    variant_b=config_b,
    test_cases=test_dataset,
    metrics=["quality", "latency", "cost"]
)

comparison = ab_test.run()

print(comparison.winner())  # Statistical significance test
```

---

## Pattern 2: A/B Test Results

```
A/B Test Results:
┌──────────────┬───────────┬───────────┬──────────────┐
│ Metric       │ Variant A │ Variant B │ Significance │
├──────────────┼───────────┼───────────┼──────────────┤
│ Quality      │ 8.5/10    │ 9.2/10    │ p < 0.01 ✓   │
│ Latency      │ 1.8s      │ 2.4s      │ p < 0.05 ✓   │
│ Cost/request │ $0.021    │ $0.067    │ p < 0.001 ✓  │
└──────────────┴───────────┴───────────┴──────────────┘

Recommendation: Variant A
- Better cost efficiency (68% cheaper)
- Faster response time (25% improvement)
- Slightly lower quality (0.7 points), but within threshold
```

**Winner: Variant A** - Best cost/performance tradeoff

---

## Pattern 3: Regression Detection

**When to use:** Ensuring new changes don't break existing behavior

```python
from harness_ai_evals import RegressionDetector

detector = RegressionDetector(
    baseline_results="evals/baselines/v1.2.0.json",
    threshold=0.05  # 5% tolerance
)

# Run current version
current_results = runner.run(test_cases)

# Compare against baseline
regression_report = detector.compare(current_results)

if regression_report.has_regressions():
    print(f"⚠️ Regressions detected:")
    for reg in regression_report.regressions:
        print(f"  - {reg.test_id}: "
              f"{reg.baseline_score} → {reg.current_score} "
              f"({reg.delta})")
    exit(1)  # Fail CI/CD pipeline
```

---

## Pattern 4: Cost Optimization

**When to use:** Finding the cheapest model that meets quality bar

```python
from harness_ai_evals import CostOptimizer

optimizer = CostOptimizer(
    models=["gpt-4o-mini", "claude-sonnet-4", "claude-opus-4"],
    quality_threshold=8.5,  # Minimum acceptable score
    test_cases=test_dataset
)

# Test all models
results = optimizer.run()

print(f"Recommended model: {results.best_model}")
print(f"Quality: {results.quality_score}/10")
print(f"Cost: ${results.cost_per_1k_requests}")
print(f"Savings vs. baseline: {results.savings_pct}%")
```

---

## Pattern 5: Multi-Turn Conversations

**When to use:** Evaluating conversational agents

```python
conversation_test = {
    "test_id": "conversation_001",
    "turns": [
        {
            "user": "I need help with my account",
            "expected": ["asks what kind of help", "friendly tone"]
        },
        {
            "user": "I forgot my password",
            "expected": ["provides reset instructions", "asks for email"]
        },
        {
            "user": "My email is user@example.com",
            "expected": ["confirms reset link sent", "next steps"]
        }
    ],
    "conversation_criteria": [
        "Maintains context across turns",
        "Doesn't repeat information",
        "Natural conversation flow"
    ]
}

conv_evaluator = ConversationEvaluator()
result = conv_evaluator.evaluate(agent, conversation_test)
```

---

# 5️⃣ Best Practices

---

## Building Quality Test Datasets

### ✅ Do:

- **Diverse inputs**: Cover common cases + edge cases
- **Real user data**: Anonymized production queries
- **Clear criteria**: Explicit success conditions
- **Versioned**: Track dataset changes over time
- **Balanced**: Represent actual usage distribution

### ❌ Don't:

- Over-fit to current implementation
- Test only happy paths
- Use synthetic/toy examples
- Mix multiple test objectives
- Ignore corner cases

---

## Test Dataset Structure

```python
{
  "version": "1.0.0",
  "created": "2024-03-15",
  "tests": [
    {
      "id": "auth_001",
      "category": "authentication",
      "difficulty": "easy",
      "input": "How do I log in?",
      "expected_output": "Login instructions...",
      "criteria": {
        "required": [
          "mentions login button",
          "explains where to find it"
        ],
        "optional": [
          "provides screenshot",
          "mentions mobile app"
        ]
      },
      "scoring": {
        "required_weight": 0.7,
        "optional_weight": 0.3
      }
    }
  ]
}
```

---

## Evaluation Frequency

<div class="columns">

<div>

### Pre-Deployment

**When**: Every code change
**What**: Full regression suite
**Threshold**: Zero regressions
**Time**: 5-15 minutes

</div>

<div>

### Continuous Monitoring

**When**: Every hour in production
**What**: Sampled live queries
**Threshold**: Alert if < 90%
**Time**: Real-time

</div>

</div>

---

## Quality Gates

```python
# Define quality gates for CI/CD
quality_gates = {
    "overall_score": {
        "threshold": 8.5,
        "required": True
    },
    "pass_rate": {
        "threshold": 0.95,  # 95%
        "required": True
    },
    "avg_latency": {
        "threshold": 2.0,  # seconds
        "required": True
    },
    "cost_per_request": {
        "threshold": 0.05,  # $0.05
        "required": False,
        "warning_only": True
    }
}

# CI/CD integration
if not results.meets_gates(quality_gates):
    print("❌ Quality gates failed")
    exit(1)
```

---

## Handling Flaky Tests

**Problem**: LLM outputs are non-deterministic

**Solutions**:

1. **Run multiple times**: Average scores across 3-5 runs
2. **Temperature = 0**: Reduce randomness for test stability
3. **Fuzzy matching**: Don't require exact outputs
4. **Threshold bands**: 8.0-9.0 instead of exact 8.5
5. **Statistical significance**: Require p < 0.05

```python
# Example: Multiple runs with variance check
results = []
for _ in range(5):
    r = runner.run(test_cases)
    results.append(r.average_score)

avg_score = np.mean(results)
std_dev = np.std(results)

if std_dev > 1.0:
    print(f"⚠️ High variance: {std_dev:.2f}")
```

---

## Cost Management

**Evaluation costs can add up quickly!**

### Optimization Strategies:

1. **Tiered testing**:
   - PR: 10% sample (cheap, fast)
   - Merge: 50% sample (moderate)
   - Release: 100% full suite (comprehensive)

2. **Cheaper judge models**: Use GPT-4o-mini for evaluation

3. **Caching**: Cache evaluator LLM calls

4. **Parallel execution**: Batch API requests

5. **Smart sampling**: Run full suite weekly, sample daily

---

## Monitoring in Production

```python
from harness_ai_evals import ProductionMonitor

monitor = ProductionMonitor(
    sample_rate=0.10,  # Evaluate 10% of production traffic
    evaluators=[quality_judge, safety_judge],
    alert_threshold=0.85
)

@app.post("/agent/chat")
async def chat(request: ChatRequest):
    response = await agent.run(request.message)

    # Async evaluation (non-blocking)
    monitor.evaluate_async(
        input=request.message,
        output=response,
        metadata={"user_id": request.user_id}
    )

    return response

# Monitor dashboard shows real-time quality metrics
```

---

## Evaluation Report Example

```
═══════════════════════════════════════════════════════
        HARNESS AI AGENT EVALUATION REPORT
═══════════════════════════════════════════════════════

Agent: support-agent-v2.1.0
Date: 2024-03-18 14:30:00 UTC
Dataset: production_sample_1000

┌─────────────────────┬──────────┬──────────┐
│ Metric              │ Score    │ Baseline │
├─────────────────────┼──────────┼──────────┤
│ Overall Quality     │ 9.2/10   │ 8.8/10   │
│ Pass Rate           │ 97.5%    │ 95.0%    │
│ Avg Latency         │ 1.24s    │ 1.45s    │
│ P95 Latency         │ 2.10s    │ 2.80s    │
│ Cost per Request    │ $0.018   │ $0.031   │
│ Error Rate          │ 0.3%     │ 1.2%     │
└─────────────────────┴──────────┴──────────┘

✅ All quality gates passed
📈 +4.5% quality improvement vs baseline
💰 42% cost reduction
⚡ 25% latency improvement
```

---

## Common Pitfalls

**1. Testing implementation, not behavior**
- ❌ "Agent calls `search_tool` first"
- ✅ "Agent finds accurate information"

**2. Over-reliance on exact matching**
- ❌ Expecting exact string match
- ✅ Semantic similarity or criteria-based

**3. Ignoring edge cases**
- Test empty inputs, very long inputs, ambiguous queries

**4. Not versioning test datasets**
- Track changes, understand why scores change

**5. Running evals only once**
- Continuous evaluation catches drift

---

## Advanced: Custom Evaluators

```python
from harness_ai_evals import BaseEvaluator

class ToolUsageEvaluator(BaseEvaluator):
    """Validates agent used correct tools"""

    def evaluate(self, test_case, agent_output, metadata):
        expected_tools = test_case.get("expected_tools", [])
        actual_tools = metadata.get("tools_called", [])

        # Check tool usage
        correct_tools = set(expected_tools) & set(actual_tools)
        score = len(correct_tools) / len(expected_tools)

        return EvaluationResult(
            score=score,
            passed=score >= 0.8,
            details={
                "expected": expected_tools,
                "actual": actual_tools,
                "missing": list(set(expected_tools) - set(actual_tools))
            }
        )
```

---

## Integration with Harness Platform

### Harness AI Gateway Integration

```python
from harness.ai import AIGateway

# AI Gateway automatically logs all requests
gateway = AIGateway(
    api_key=os.getenv("HARNESS_API_KEY"),
    enable_evaluation=True,
    evaluation_sample_rate=0.1
)

agent = Agent(
    model="claude-sonnet-4",
    gateway=gateway  # Automatic logging + evaluation
)

# View metrics in Harness dashboard:
# - Quality trends over time
# - Cost analysis
# - Latency percentiles
# - Error rates
```

---

## Harness Dashboards

**Automated Reporting:**

- **Quality Dashboard**: Track eval scores over time
- **Cost Dashboard**: LLM spend by agent, model, user
- **Performance Dashboard**: Latency, throughput, errors
- **Regression Alerts**: Automatic notifications on quality drops

**Access:**
Harness Platform → AI Observability → Agent Evaluations

---

## Resources

**Documentation:**
- [Evaluation SDK Docs](https://docs.harness.io/ai/evals)
- [Example Test Suites](https://github.com/harness/ai-evals-examples)
- [LLM-as-Judge Guide](https://docs.harness.io/ai/llm-judge)

**Support:**
- Slack: `#ai-platform-support`
- Email: ai-support@harness.io

**Workshops:**
- Office Hours: Tuesdays 2pm PT
- Monthly Deep Dives: First Thursday

---

# Summary

---

## Key Takeaways

**1. Start Simple**
   Create basic test dataset → Run evaluation → Iterate

**2. Automate Early**
   Integrate into CI/CD from day one

**3. Monitor Continuously**
   Production sampling catches real-world issues

**4. Use LLM-as-Judge**
   Powerful for subjective quality evaluation

**5. Track Over Time**
   Baseline → Compare → Detect regressions

---

## Next Steps

### This Week:
1. Create your first 10-test evaluation dataset
2. Run a baseline evaluation
3. Integrate into CI/CD pipeline

### This Month:
1. Expand to 100+ test cases
2. Implement LLM-as-Judge evaluation
3. Set up production monitoring
4. Create quality dashboards

### This Quarter:
1. A/B test model improvements
2. Optimize cost/quality tradeoff
3. Build regression detection suite
4. Share best practices with team

---

# Questions?

**Demo & Hands-On:**
Let's run a live evaluation together

**Contact:**
- Slack: `#ai-platform-support`
- Docs: [docs.harness.io/ai/evals](https://docs.harness.io/ai/evals)
- Office Hours: Tuesdays 2pm PT

---

# Thank You!

**Let's build reliable AI agents together** 🚀

_Scan QR code for quick start guide_

