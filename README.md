### Temporal

[Temporal](https://temporal.io) is a distributed workflow orchestration platform that enables developers to build, run, and scale reliable, long-running workflows with fault tolerance, retries, and state management. It ensures workflows persist through failures and provides tools for monitoring, debugging, and replaying executions.

### Ansible

Ansible is an open-source automation tool used for configuring systems, deploying software, and orchestrating IT tasks. It uses YAML-based playbooks to define tasks and operates over SSH, requiring no agents on target systems. Ansible simplifies infrastructure management by automating repetitive tasks like server provisioning, application deployment, and configuration updates.


### Ansible without/without Temporal

Temporal addresses challenges in automating workflows by providing a robust framework for managing complex, long-running processes that require reliability, scalability, and fault tolerance. Without Temporal, running Ansible playbooks or similar automation tasks can face issues like failed executions due to transient errors, lack of visibility into ongoing processes, inability to resume workflows after system crashes, and difficulty in coordinating multiple dependent tasks. Temporal ensures that workflows can recover gracefully from failures, maintain state across retries, and allow dynamic adjustments based on runtime conditions. It also enables seamless integration of different tools and systems within a single workflow, making it easier to orchestrate multi-step operations that involve not just Ansible but potentially other infrastructure or application management tools. By handling the complexities of task coordination, retries, and state management, Temporal allows developers to focus on defining the logic of their workflows rather than worrying about the underlying execution mechanics.

```mermaid
graph TD
    %% Custom Styling
    classDef temporal fill:#4CAF50,stroke:#333,stroke-width:2px,color:#fff;
    classDef ansible fill:#FF9800,stroke:#333,stroke-width:2px,color:#fff;
    classDef problem fill:#F44336,stroke:#333,stroke-width:2px,color:#fff;
    classDef solution fill:#2196F3,stroke:#333,stroke-width:2px,color:#fff;

    %% Nodes
    A[Challenges Without Temporal]:::problem --> B["Failed Executions<br>(Transient Errors)"];
    A --> C["Lack of Visibility<br>(No Monitoring)"];
    A --> D["Inability to Resume<br>(Crash Recovery)"];
    A --> E["Coordination Issues<br>(Dependent Tasks)"];

    F[Temporal Framework]:::temporal --> G["Retry Mechanisms<br>(Fault Tolerance)"];
    F --> H["State Management<br>(Persist State Across Failures)"];
    F --> I["Dynamic Adjustments<br>(Runtime Conditions)"];
    F --> J["Multi-Tool Integration<br>(Ansible + Others)"];

    K[Benefits of Temporal]:::solution --> L["Graceful Recovery<br>(From Failures)"];
    K --> M["Maintain State<br>(Across Retries)"];
    K --> N["Orchestrate Multi-Step<br>(Complex Operations)"];
    K --> O["Focus on Logic<br>(Not Execution Mechanics)"];

    %% Flow
    B -->|Solved By| F
    C -->|Solved By| F
    D -->|Solved By| F
    E -->|Solved By| F

    F -->|Enables| K

    %% Subgraph for Grouping Temporal Features
    subgraph TemporalFeatures
        direction TB
        G[Retry Mechanisms]
        H[State Management]
        I[Dynamic Adjustments]
        J[Multi-Tool Integration]
    end
```

### Simple workflow

[![GitLab](https://e.radikal.host/2025/05/17/Screenshot-2025-05-17-at-13.26.31.png)](https://radikal.host/i/IGWUQg)

[![Temporal](https://e.radikal.host/2025/05/17/Screenshot-2025-05-17-at-07.52.56.png)](https://radikal.host/i/Ir0Q5K)

```yaml
ansible_workflows:
  stage: deploy
  tags:
    - shell
  script:
    - python3 ansible.py --playbook apt_update.yml
```

#### Long-Running Workflows with Checkpointing

Problem: Ansible playbooks fail on long-running tasks (e.g., cloud provisioning, multi-hour deployments).
Solution:
Use Temporal workflows to automatically resume from failures.
Checkpoint progress (e.g., "50% of VMs deployed") even if the process crashes.
```python
@workflow.defn
class AnsibleWorkflow:
    async def run(self, playbook: str):
        while not workflow.is_replaying():
            result = await workflow.execute_activity(
                run_ansible_playbook,
                args=[playbook],
                start_to_close_timeout=timedelta(hours=6),
                heartbeat_timeout=timedelta(minutes=5),
            )
            if result.failed:
                await asyncio.sleep(300)  # Retry after 5 minutes
```

```mermaid
sequenceDiagram
    participant User
    participant Temporal
    participant Ansible
    User->>Temporal: Start Workflow
    Temporal->>Ansible: Run Playbook
    Ansible-->>Temporal: Progress (50% done)
    Note over Temporal: Crash occurs
    Temporal->>Temporal: Resume from checkpoint
    Temporal->>Ansible: Retry failed tasks
    Ansible-->>Temporal: Task completed
    Temporal-->>User: Workflow completed
```


#### Dynamic Parallel Execution

Problem: Ansible’s async is limited—hard to manage 1000s of nodes dynamically.
Solution:
Fan-out/fan-in workflows (e.g., deploy 500 VMs in parallel, then aggregate results).
```python
async def deploy_vms():
    vm_list = await get_vm_list_from_api()
    results = await asyncio.gather(
        *[run_ansible_playbook(f"deploy_vm.yml --extra-vars 'vm_id={vm.id}'") for vm in vm_list]
    )
    return sum(results)
```

```mermaid
graph TD
    A[Start Workflow] --> B[Fetch VM List]
    B --> C[Fan-Out: Deploy VMs in Parallel]
    C --> D[Deploy VM 1]
    C --> E[Deploy VM 2]
    C --> F[Deploy VM 3]
    D --> G[Aggregate Results]
    E --> G
    F --> G
    G --> H[Workflow Completed]
```

#### Human-in-the-Loop Approvals

Problem: Ansible Tower requires manual static approvals.
Solution:
Dynamic pauses (e.g., "Wait for Slack approval before deleting prod DB").
```python
@workflow.defn
class ProdDeploymentWorkflow:
    async def run(self):
        await run_ansible_playbook("deploy_staging.yml")
        if workflow.is_replaying():
            await workflow.wait_for_signal("prod_approval")
        else:
            await send_slack_approval_request()
            await workflow.wait_for_signal("prod_approval")  # Blocks until approved
        await run_ansible_playbook("deploy_prod.yml")
```

```mermaid
sequenceDiagram
    participant Workflow
    participant Slack
    participant User
    Workflow->>Slack: Send Approval Request
    Slack->>User: Notify for Approval
    User->>Slack: Approve
    Slack->>Workflow: Signal Approval
    Workflow->>Workflow: Proceed to Next Step
```

#### Cross-Cloud Orchestration

Problem: Ansible alone can’t coordinate AWS + Azure + GCP workflows.
Solution:
Temporal workflows orchestrate multi-cloud playbooks:
```python
async def migrate_to_aws():
    await run_ansible_playbook("azure_shutdown.yml")
    await run_ansible_playbook("aws_provision.yml")
    if await check_aws_health():
        await run_ansible_playbook("cutover_dns.yml")
```

```mermaid
graph TD
    A[Start Workflow] --> B[Shutdown Azure Resources]
    B --> C[Provision AWS Resources]
    C --> D[Check AWS Health]
    D --> E[Update DNS]
    E --> F[Workflow Completed]
```

#### Event-Driven Ansible (EDA) on Steroids

Problem: Ansible EDA reacts to simple events (e.g., webhooks).
Solution:
Complex event chains (e.g., "If CPU > 90% for 5min, scale + notify PagerDuty + ticket").
```python
async def auto_scale_workflow():
    while True:
        metrics = await fetch_cloud_metrics()
        if metrics.cpu > 90:
            await run_ansible_playbook("scale_out.yml")
            await page_team()
            await workflow.sleep(timedelta(minutes=5))  # Cooldown
```

```mermaid
sequenceDiagram
    participant Monitor
    participant Workflow
    participant Ansible
    participant PagerDuty
    Monitor->>Workflow: CPU > 90% for 5min
    Workflow->>Ansible: Scale Out
    Workflow->>PagerDuty: Notify Team
    Workflow->>Workflow: Cooldown for 5min
```

#### Stateful Workflows with Recovery

Problem: Ansible has no memory of past runs.
Solution:
Temporal remembers state (e.g., "Retry only failed nodes after outage").
```python
async def patch_workflow():
    hosts = await get_hosts()
    for host in hosts:
        try:
            await run_ansible_playbook(f"patch.yml -l {host}")
        except ActivityError:
            await log_failed_host(host)
    if workflow.is_replaying():
        await retry_failed_hosts()  # Only retries what crashed
```

```mermaid
graph TD
    A[Start Workflow] --> B[Get Host List]
    B --> C[Run Playbook on Host 1]
    B --> D[Run Playbook on Host 2]
    B --> E[Run Playbook on Host 3]
    C --> F[Log Failed Host]
    D --> F
    E --> F
    F --> G[Retry Failed Hosts]
    G --> H[Workflow Completed]
```

#### Time Travel Debugging

Problem: Debugging Ansible failures is painful.
Solution:
Replay workflows exactly (e.g., "See why playbook failed 3 days ago").
```python
# Temporal UI shows full history + inputs/outputs for every step.
```

```mermaid
sequenceDiagram
    participant User
    participant Temporal
    User->>Temporal: Replay Workflow
    Temporal->>Temporal: Load History
    Temporal->>Temporal: Re-execute Steps
    Temporal-->>User: Show Failure Details
```

#### Cross-Tool Chaining

Problem: Ansible can’t seamlessly call Terraform + Kubernetes.
Solution:
Mix tools in one workflow:

```python
async def deploy_full_stack():
    await run_terraform("apply")
    await run_kubectl("apply -f k8s/")
    await run_ansible_playbook("configure_ingress.yml")
```

```mermaid
graph TD
    A[Start Workflow] --> B[Run Terraform Apply]
    B --> C[Run Kubectl Apply]
    C --> D[Run Ansible Playbook]
    D --> E[Workflow Completed]
```
### Dynamic Ansible inventory

Here’s how to create a dynamic Ansible inventory using a Temporal workflow that fetches host data from a CMDB via REST API, ensuring real-time, fault-tolerant infrastructure management:
```
Temporal Workflow → Fetches CMDB Data (REST API) → Generates Dynamic Inventory → Runs Ansible
```

```python
#!/usr/bin/env python3
import requests
import json
import argparse

def fetch_cmdb_hosts(cmdb_api_url: str, token: str) -> dict:
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.get(f"{cmdb_api_url}/hosts", headers=headers)
    response.raise_for_status()
    return response.json()

def generate_inventory(cmdb_data: dict) -> dict:
    inventory = {
        "_meta": {"hostvars": {}},
        "all": {"hosts": []},
        "web": {"hosts": []},
        "db": {"hosts": []}
    }
    
    for host in cmdb_data["hosts"]:
        inventory["all"]["hosts"].append(host["name"])
        inventory["_meta"]["hostvars"][host["name"]] = {
            "ansible_host": host["ip"],
            "ansible_user": host["user"],
            "ansible_become": True
        }
        if host["role"] == "web":
            inventory["web"]["hosts"].append(host["name"])
        elif host["role"] == "db":
            inventory["db"]["hosts"].append(host["name"])
    
    return inventory

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--list", action="store_true", help="List all hosts")
    args = parser.parse_args()
    
    if args.list:
        cmdb_data = fetch_cmdb_hosts(
            cmdb_api_url="https://cmdb.example.com/api",
            token="your-api-token"
        )
        print(json.dumps(generate_inventory(cmdb_data)))
```

```mermaid
graph TD
    A[Start Workflow] --> B[Fetch CMDB Data]
    B --> C[Generate Dynamic Inventory]
    C --> D[Run Ansible Playbook]
    D --> E[Workflow Completed]
```


#### GitLab

To integrate the Temporal-based Ansible workflow with a **GitLab webhook** , you can configure GitLab to trigger the workflow whenever specific events occur (e.g., a push to a branch, a merge request, or a tag creation).

```mermaid
sequenceDiagram
    participant GitLab
    participant WebhookHandler
    participant Temporal
    participant Ansible
    GitLab->>WebhookHandler: Send webhook payload (Push to main)
    WebhookHandler->>WebhookHandler: Validate payload
    WebhookHandler->>Temporal: Trigger workflow
    Temporal->>Ansible: Execute playbook (apt_update.yml)
    Ansible-->>Temporal: Playbook completed
    Temporal-->>WebhookHandler: Workflow completed
    WebhookHandler-->>GitLab: Acknowledge webhook
```

#### Main use cases

```mermaid
graph TD
    A[Use Cases] --> B[Multi-Cloud Deployments]
    A --> C[Mission-Critical Workflows]
    A --> D[Self-Healing Infrastructure]
    B --> E[AWS + Azure + GCP]
    C --> F[Banking Migrations]
    D --> G[Auto-Remediation]
```


---

- Complex multi-cloud deployments
- Mission-critical workflows (e.g., banking migrations)
- Self-healing infra (e.g., auto-remediation)

---
### Terraform

Using Terraform and Temporal together offers a powerful combination for infrastructure automation, addressing gaps that arise when using either tool in isolation. Terraform excels at declarative infrastructure provisioning but requires custom providers for new or niche services, which can be time-consuming to develop and maintain. On the other hand, Temporal provides a robust orchestration layer for automating workflows, including those involving APIs, without the need for custom providers. For example, you can use Temporal to seamlessly automate interactions with services like Equinix Metal, GoDaddy, or VMware via their REST APIs, orchestrating complex, stateful workflows that Terraform alone cannot handle. While Terraform focuses on defining and managing infrastructure as code, Temporal complements it by enabling dynamic, fault-tolerant, and long-running operations, such as retries, approvals, and cross-service coordination. Together, they allow teams to leverage Terraform's strength in infrastructure provisioning while relying on Temporal for workflow automation, creating a more flexible and scalable solution than using either tool alone.

```mermaid
[![Terraform](https://e.radikal.host/2025/05/17/schema.png)](https://radikal.host/i/IGzdeE)
```
