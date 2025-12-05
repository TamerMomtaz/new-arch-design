# RootRise Technical Handoff Document
## Multi-Agent Architecture Implementation Guide

**Prepared for:** Ahmed ElGazzar, Technical DevOps Lead  
**Prepared by:** Tee, Product Creative Strategist  
**Date:** December 2024  
**Version:** 1.0

---

## A Note from Tee

Ahmed, your infrastructure work is the critical foundation that makes this ambitious multi-agent system possible. Without solid Frappe architecture, none of what follows would be executable. This document provides everything you need to implement the system. Let's build something remarkable together.

---

## Executive Summary

We're building a multi-agent diagnostic system that processes SME data through 10 specialist AI agents in approximately 2 minutes, producing investor-ready reports. The system combines your Frappe expertise with sophisticated AI orchestration via LangGraph.

**Key Metrics:**
- Full diagnostic: ~2 minutes (vs 5.5 min sequential)
- Agent timeout: 60 seconds each
- Target confidence: 0.7+ per agent

---

## 1. The Pantheon: Our Agent Naming Convention

Each agent is named after a historical pioneer. This creates brand differentiation and memorability.

### Quick Reference

| Codename | Display Name | Type | Named After |
|----------|-------------|------|-------------|
| `supervisor` | The Drucker | Orchestrator | Peter Drucker |
| `diagnostic` | The Marvin | Core | Marvin Bower |
| `finance` | The Graham | Core | Benjamin Graham |
| `export` | The Ricardo | Add-on | David Ricardo |
| `digital` | The Lovelace | Add-on | Ada Lovelace |
| `hr` | The Mayo | Add-on | Elton Mayo |
| `supply_chain` | The Ohno | Add-on | Taiichi Ohno |
| `market` | The Porter | Add-on | Michael Porter |
| `report` | The Tufte | Utility | Edward Tufte |
| `qa` | The Deming | Utility | W. Edwards Deming |

**Agent Types:**
- **Orchestrator (1):** Always active, coordinates workflow
- **Core (2):** Always included in every diagnostic
- **Add-on (5):** User selects which to activate
- **Utility (2):** Internal for reports and QA

---

## 2. Frappe DocTypes to Create

### DocType 1: Agent Config

Store agent configurations:

```
DocType Name: Agent Config
Module: RootRise

Fields:
- agent_name (Data, Required) - e.g., "The Marvin"
- agent_codename (Data, Required, Unique) - e.g., "diagnostic"
- agent_type (Select: orchestrator/core/addon/utility, Required)
- historical_figure (Data) - e.g., "Marvin Bower"
- historical_years (Data) - e.g., "1903-2003"
- tagline (Small Text)
- model_provider (Select: openai/anthropic/azure, Default: openai)
- model_name (Data, Default: gpt-4-turbo)
- temperature (Float, Default: 0.4)
- max_tokens (Int, Default: 4000)
- timeout_seconds (Int, Default: 60)
- system_prompt (Long Text, Required)
- tools_json (Code, JSON)
- output_schema (Code, JSON)
- is_active (Check, Default: 1)
- version (Data, Default: 1.0)
```

### DocType 2: Workflow Run

Track diagnostic executions:

```
DocType Name: Workflow Run
Module: RootRise

Fields:
- sme (Link to SME, Required)
- status (Select: pending/running/completed/failed/needs_review)
- selected_agents (JSON) - list of agent codenames
- report_config (JSON)
- started_at (Datetime)
- completed_at (Datetime)
- duration_seconds (Float)
- overall_score (Float)
- confidence_score (Float)
- requires_human_review (Check)
- review_reason (Data)
- reviewed_by (Link to User)
- reviewed_at (Datetime)
- report_url (Data)
- report_format (Select: pdf/docx/html)
```

### DocType 3: Agent Output Log

Store individual agent results:

```
DocType Name: Agent Output Log
Module: RootRise

Fields:
- workflow_run (Link to Workflow Run, Required)
- agent_codename (Data, Required)
- started_at (Datetime)
- completed_at (Datetime)
- duration_ms (Int)
- output_json (JSON)
- confidence_score (Float)
- model_used (Data)
- tokens_input (Int)
- tokens_output (Int)
- status (Select: success/failed/timeout/low_confidence)
- error_message (Text)
```

---

## 3. Python Utility Functions

Create file: `rootrise/utils/agents.py`

```python
import frappe
import json
from datetime import datetime

def get_agent_config(codename: str) -> dict:
    """Load agent configuration from database"""
    doc = frappe.get_doc("Agent Config", {"agent_codename": codename})
    return {
        "name": doc.agent_name,
        "codename": doc.agent_codename,
        "type": doc.agent_type,
        "historical_figure": doc.historical_figure,
        "tagline": doc.tagline,
        "model_provider": doc.model_provider,
        "model_name": doc.model_name,
        "temperature": doc.temperature,
        "max_tokens": doc.max_tokens,
        "timeout": doc.timeout_seconds,
        "system_prompt": doc.system_prompt,
        "tools": json.loads(doc.tools_json or "[]"),
        "output_schema": json.loads(doc.output_schema or "{}")
    }

def get_active_agents(agent_type: str = None) -> list:
    """Get all active agents, optionally filtered by type"""
    filters = {"is_active": 1}
    if agent_type:
        filters["agent_type"] = agent_type
    
    return frappe.get_all(
        "Agent Config",
        filters=filters,
        fields=["agent_codename", "agent_name", "agent_type", "historical_figure"]
    )

def hydrate_prompt(prompt_template: str, sme_data: dict) -> str:
    """Replace placeholders in prompt with SME data"""
    replacements = {
        "{{SME_NAME}}": sme_data.get("company_name", ""),
        "{{INDUSTRY}}": sme_data.get("industry", ""),
        "{{COUNTRY}}": sme_data.get("country", ""),
        "{{EMPLOYEE_COUNT}}": str(sme_data.get("employee_count", "")),
        "{{ANNUAL_REVENUE}}": str(sme_data.get("annual_revenue", "")),
        "{{YEARS_IN_BUSINESS}}": str(sme_data.get("years_in_business", "")),
    }
    
    result = prompt_template
    for placeholder, value in replacements.items():
        result = result.replace(placeholder, value)
    return result

def log_agent_output(workflow_run_id: str, agent_codename: str, 
                     output: dict, duration_ms: int, model_info: dict):
    """Log agent output to database"""
    doc = frappe.new_doc("Agent Output Log")
    doc.workflow_run = workflow_run_id
    doc.agent_codename = agent_codename
    doc.output_json = json.dumps(output)
    doc.confidence_score = output.get("confidence_score", 0)
    doc.duration_ms = duration_ms
    doc.model_used = model_info.get("model")
    doc.tokens_input = model_info.get("input_tokens", 0)
    doc.tokens_output = model_info.get("output_tokens", 0)
    doc.status = "success" if output.get("confidence_score", 0) >= 0.7 else "low_confidence"
    doc.completed_at = datetime.now()
    doc.insert()
    return doc.name

def create_workflow_run(sme_id: str, selected_agents: list, report_config: dict) -> str:
    """Create a new workflow run"""
    doc = frappe.new_doc("Workflow Run")
    doc.sme = sme_id
    doc.status = "pending"
    doc.selected_agents = json.dumps(selected_agents)
    doc.report_config = json.dumps(report_config)
    doc.insert()
    return doc.name

def update_workflow_status(workflow_id: str, status: str, **kwargs):
    """Update workflow run status"""
    doc = frappe.get_doc("Workflow Run", workflow_id)
    doc.status = status
    for key, value in kwargs.items():
        if hasattr(doc, key):
            setattr(doc, key, value)
    doc.save()
```

---

## 4. API Endpoints

Create file: `rootrise/api/diagnostic.py`

```python
import frappe
from frappe import _
from rootrise.utils.agents import (
    create_workflow_run, 
    get_active_agents,
    update_workflow_status
)

@frappe.whitelist()
def start_diagnostic(sme_id: str, selected_agents: list = None, report_config: dict = None):
    """
    Start a diagnostic workflow
    
    Args:
        sme_id: SME document ID
        selected_agents: List of agent codenames (optional, defaults to core only)
        report_config: Report configuration (format, language, etc.)
    
    Returns:
        workflow_run_id: ID of the created workflow run
    """
    # Validate SME exists
    if not frappe.db.exists("SME", sme_id):
        frappe.throw(_("SME not found"))
    
    # Default to core agents if none selected
    if not selected_agents:
        core_agents = get_active_agents("core")
        selected_agents = [a["agent_codename"] for a in core_agents]
    
    # Always include core agents
    core_codenames = ["diagnostic", "finance"]
    for core in core_codenames:
        if core not in selected_agents:
            selected_agents.insert(0, core)
    
    # Default report config
    if not report_config:
        report_config = {
            "format": "pdf",
            "language": "english",
            "detail_level": "executive"
        }
    
    # Create workflow run
    workflow_id = create_workflow_run(sme_id, selected_agents, report_config)
    
    # Trigger async execution (via background job)
    frappe.enqueue(
        "rootrise.agents.workflow.execute_diagnostic",
        workflow_id=workflow_id,
        queue="long",
        timeout=600  # 10 minute max
    )
    
    return {"workflow_run_id": workflow_id, "status": "started"}

@frappe.whitelist()
def get_workflow_status(workflow_id: str):
    """Get status of a workflow run"""
    doc = frappe.get_doc("Workflow Run", workflow_id)
    
    # Get agent outputs
    outputs = frappe.get_all(
        "Agent Output Log",
        filters={"workflow_run": workflow_id},
        fields=["agent_codename", "status", "confidence_score", "duration_ms"]
    )
    
    return {
        "workflow_id": workflow_id,
        "status": doc.status,
        "overall_score": doc.overall_score,
        "confidence_score": doc.confidence_score,
        "requires_human_review": doc.requires_human_review,
        "review_reason": doc.review_reason,
        "report_url": doc.report_url,
        "agent_outputs": outputs
    }

@frappe.whitelist()
def provide_human_feedback(workflow_id: str, approved: bool, feedback: str = None):
    """Submit human review feedback"""
    doc = frappe.get_doc("Workflow Run", workflow_id)
    
    if not doc.requires_human_review:
        frappe.throw(_("This workflow does not require human review"))
    
    doc.reviewed_by = frappe.session.user
    doc.reviewed_at = frappe.utils.now_datetime()
    
    if approved:
        doc.status = "completed"
        # Trigger report delivery
        frappe.enqueue(
            "rootrise.agents.workflow.deliver_report",
            workflow_id=workflow_id
        )
    else:
        doc.status = "needs_revision"
        # Store feedback for re-processing
        doc.add_comment("Comment", feedback)
    
    doc.save()
    
    return {"status": doc.status}

@frappe.whitelist()
def get_available_agents():
    """Get all available agents for selection"""
    agents = get_active_agents()
    
    # Group by type
    result = {
        "core": [],
        "addon": [],
        "utility": []
    }
    
    for agent in agents:
        agent_type = agent["agent_type"]
        if agent_type in result:
            result[agent_type].append({
                "codename": agent["agent_codename"],
                "name": agent["agent_name"],
                "historical_figure": agent["historical_figure"]
            })
    
    return result
```

---

## 5. The Four Validation Gates

### Gate 1: Data Validation (Automatic)
**When:** Before any agent processing
**Checks:** Questionnaire completeness, data types, required fields
**Fail Action:** Return to user requesting missing data

### Gate 2: Confidence Check (Automatic + Escalate)
**When:** After each agent completes
**Checks:** Confidence score >= 0.7
**Fail Action:** Flag for supervisor review

### Gate 3: Conflict Resolution (Auto + HITL)
**When:** After all agents, before synthesis
**Triggers for Human Review:**
- Contradictory recommendations
- Financial impact > $50K
- Custom agent output

### Gate 4: Quality Assurance (Auto + HITL)
**When:** After report generation
**Checks:** Completeness, accuracy, actionability
**Max Revisions:** 3 (then force-deliver with caveats)

### HITL Triggers Summary
Human review required when:
- First-time SME (no historical context)
- Investment readiness score 4-5 (donor-facing)
- Conflicting agent outputs
- Critical scores < 40
- Exceptional scores > 85
- Financial recommendations > $50K

---

## 6. Model Recommendations

| Agent | Recommended Model | Temperature | Notes |
|-------|-------------------|-------------|-------|
| The Drucker | gpt-4-turbo | 0.3 | Consistency critical |
| The Marvin | gpt-4-turbo | 0.4 | Balanced |
| The Graham | gpt-4-turbo | 0.3 | Financial precision |
| The Ricardo | gpt-4-turbo | 0.4 | Balanced |
| The Lovelace | gpt-4-turbo | 0.5 | Slightly creative |
| The Mayo | gpt-4-turbo | 0.4 | Balanced |
| The Ohno | gpt-4-turbo | 0.3 | Process precision |
| The Porter | gpt-4-turbo | 0.5 | Strategic thinking |
| The Tufte | gpt-4-turbo | 0.4 | Report quality |
| The Deming | gpt-3.5-turbo | 0.2 | Cost-efficient QA |

---

## 7. Testing Strategy

### Phase 1: Unit Tests
- Test each agent independently with mock SME data
- Verify output schema compliance
- Test timeout handling

### Phase 2: Integration Tests
- Test parallel execution
- Test validation gates
- Test conflict resolution

### Phase 3: End-to-End
- Test with Natura Foods Processing (demo SME)
- Full workflow from input to report
- Measure timing and performance

### Test SME Data: Natura Foods Processing
```json
{
  "company_name": "Natura Foods Processing",
  "industry": "Food Manufacturing",
  "country": "Egypt",
  "employee_count": 45,
  "annual_revenue": 2500000,
  "years_in_business": 8,
  "export_status": "exploring",
  "digital_maturity": "basic"
}
```

---

## 8. Implementation Timeline

### Week 1-2 (Your Focus)
- [ ] Create Agent Config DocType
- [ ] Create Workflow Run DocType
- [ ] Create Agent Output Log DocType
- [ ] Set up API endpoints
- [ ] Load initial agent configurations

### Week 3-4 (Collaborative)
- [ ] Integrate LangGraph workflow
- [ ] Implement core agents (Marvin, Graham)
- [ ] Test parallel execution

### Week 5-6
- [ ] Add-on agents implementation
- [ ] Validation gates
- [ ] HITL integration

### Week 7-8
- [ ] Report generation (Tufte)
- [ ] QA system (Deming)
- [ ] Arabic support

### Week 9-10
- [ ] End-to-end testing
- [ ] Demo preparation
- [ ] Performance optimization

---

## 9. Questions for You

1. **Frappe Background Jobs:** What's the best queue strategy for our 10-minute timeout workflows?

2. **Database Performance:** Should we index `workflow_run` by `sme` and `status` for the dashboard queries?

3. **API Authentication:** How should we handle API auth for external LangGraph service calls?

4. **File Storage:** Where should generated reports be stored? Frappe files or S3?

---

## 10. Resources

- **Design Document:** rootrise-advanced-agent-design.md
- **Architecture Visual:** rootrise-pantheon-architecture.html
- **Prompt Templates:** rootrise-prompt-templates-for-ahmed.md
- **Team Presentation:** rootrise-pantheon-presentation.pptx

---

## Contact

Questions? Reach out anytime. Let's build this together.

**Tee** - Product Creative Strategist

---

*"The infrastructure you're building is the foundation everything else depends on."*
