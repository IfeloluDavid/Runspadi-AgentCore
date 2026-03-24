# Runspadi Multi-Agent Delivery System — AgentCore Edition

**AWS Partner:** Nerdshub | **Customer:** Runspadi Nigeria | **Region:** us-east-1  
**Framework:** Amazon Bedrock AgentCore + Strands Agents  
**Model:** Claude 3.5 Sonnet v2 (`us.anthropic.claude-3-5-sonnet-20241022-v2:0`)

---

## Architecture Overview

Six autonomous agents deployed as independent **AgentCore Runtimes**, each with its own entrypoint, system prompt, and tool set. Agents share state via DynamoDB and communicate results back to the orchestrating notebook or API Gateway.

```
┌──────────────────────────────────────────────────────────────┐
│              Runspadi AgentCore Architecture                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Order Reception Agent   → validates address, creates job   │
│  Rider Assignment Agent  → finds nearest rider, assigns     │
│  Customer Comms Agent    → sends SMS at every stage         │
│  Monitoring Agent        → polls every 60s, detects delays  │
│  Escalation Agent        → creates tickets, alerts humans   │
│  Analytics Agent         → daily ops report via email       │
│                                                              │
│  All agents: Claude 3.5 Sonnet v2 via Amazon Bedrock        │
│  State:      DynamoDB (orders, riders, escalations)         │
│  SMS:        Termii API                                      │
│  Maps:       Google Maps Distance Matrix + Geocoding        │
│  Alerts:     Amazon SNS → dispatcher phone                  │
│  Reports:    Amazon SES → ops@runspadi.com.ng               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Project Structure

```
runspadi-agentcore/
├── agents/
│   ├── order_reception_agent.py     # Stage 1
│   ├── rider_assignment_agent.py    # Stage 2
│   ├── customer_comms_agent.py      # Stage 3
│   ├── monitoring_agent.py          # Stage 4 (cron every 60s)
│   ├── escalation_agent.py          # Stage 5 (on-demand)
│   └── analytics_agent.py           # Stage 7 (daily 23:59)
├── tools/
│   └── shared_tools.py              # All @tool functions (DynamoDB, SMS, Maps, SES)
├── infrastructure/
│   └── setup_infra.py               # Creates all AWS resources
├── notebooks/
│   └── deploy_all_agents.ipynb      # ← START HERE
├── requirements.txt
└── README.md
```

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Python | 3.10 or newer |
| AWS credentials | `aws configure` with AdministratorAccess or custom policy |
| Bedrock model access | Enable Claude 3.5 Sonnet v2 in us-east-1 Bedrock console |
| Google Maps API key | Enable Geocoding + Distance Matrix APIs |
| Termii account | Nigeria-focused SMS gateway (termii.com) |
| SES verified email | Verify `reports@runspadi.com.ng` in SES |

---

## Quick Start

### Option A — Jupyter Notebook (Recommended)

```bash
# 1. Clone / download this project
cd runspadi-agentcore

# 2. Install dependencies
pip install -r requirements.txt

# 3. Open the notebook
jupyter notebook notebooks/deploy_all_agents.ipynb

# Run cells top to bottom:
#   Cell 1  → Install deps
#   Cell 2  → Setup infra + env
#   Cell 3  → Deploy all 6 agents
#   Cell 4  → Seed test riders
#   Cell 5  → Run E2E simulation
#   Cell 6  → Check runtime status
#   Cell 7  → HTTP invocation test
```

### Option B — CLI

```bash
pip install -r requirements.txt

# Step 1: Create all AWS resources
python infrastructure/setup_infra.py

# Step 2: Update real API keys in Secrets Manager
aws secretsmanager update-secret \
  --secret-id runspadi/google-maps-api-key \
  --secret-string '{"api_key":"YOUR_REAL_GOOGLE_MAPS_KEY"}' \
  --region us-east-1

aws secretsmanager update-secret \
  --secret-id runspadi/termii-api-key \
  --secret-string '{"api_key":"YOUR_REAL_TERMII_KEY"}' \
  --region us-east-1

# Step 3: Enable Bedrock model access
# AWS Console → Bedrock → Model access → Request Claude 3.5 Sonnet v2

# Step 4: Deploy each agent
source .env

for agent in order_reception rider_assignment customer_comms monitoring escalation analytics; do
  agentcore configure -e agents/${agent}_agent.py \
    --agent-name runspadi-${agent} \
    --region us-east-1 \
    --execution-role-arn $AGENTCORE_EXECUTION_ROLE_ARN \
    --disable-memory
  agentcore launch
done
```

---

## AWS Competency Coverage (QCHK 001-006)

| Requirement | Implementation |
|-------------|----------------|
| **QCHK-001 Inference** | Claude 3.5 Sonnet v2 via Amazon Bedrock, on-demand pricing |
| **QCHK-002 SDKs/Tooling** | `bedrock-agentcore` SDK + `strands-agents` framework |
| **QCHK-003 Interoperability** | Cross-agent coordination via shared DynamoDB state; Google Maps, Termii, SES integrations |
| **QCHK-004 Security** | Dedicated IAM role (least privilege), Secrets Manager for API keys, CloudWatch logging |
| **QCHK-005 Responsible AI** | Human-in-the-loop escalation, severity classification, customer data handled per Nigerian privacy norms |
| **QCHK-006 Compute** | AgentCore Runtime (serverless, consumption-based, microVM isolation per session) |

---

## Environment Variables (.env)

Generated automatically by `infrastructure/setup_infra.py`. Key variables:

| Variable | Description |
|----------|-------------|
| `AWS_REGION` | Deployment region (us-east-1) |
| `BEDROCK_MODEL_ID` | Claude 3.5 Sonnet v2 model ID |
| `ORDERS_TABLE` | DynamoDB orders table name |
| `RIDERS_TABLE` | DynamoDB riders table name |
| `ESCALATIONS_SNS_ARN` | SNS topic for dispatcher alerts |
| `AGENTCORE_EXECUTION_ROLE_ARN` | IAM role ARN for all runtimes |
| `SES_FROM_ADDRESS` | Verified SES sender email |

---

## Monitoring

- **CloudWatch Logs:** `/runspadi/agents` (90-day retention)
- **AgentCore Observability:** Built-in tracing per invocation
- **Daily Report:** Analytics Agent emails `ops@runspadi.com.ng` at 23:59 WAT

---

**Document Version:** 1.0 | **Region:** us-east-1 | **Status:** Ready to deploy
