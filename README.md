# azure-monitor-alerting
messing about with azure monitor and alerting 


# Used qwen3.5:cloud to generate the following

## Objective/Ask
Here is a comprehensive plan and Terraform implementation to monitor a specific Windows process on an 
Azure VM, alert via email after 3 consecutive failures, and visualize the status on an Azure 
Dashboard.

### 1. Architecture Overview

1.  **Target:** Azure Windows Virtual Machine.
2.  **Agent:** Azure Monitor Agent (AMA) installed via VM Extension.
3.  **Data Collection:** Data Collection Rule (DCR) configured to ingest **Process Inventory** data 
into a Log Analytics Workspace.
4.  **Alerting:** Azure Monitor Scheduled Query Rule (V2). It runs a KQL query every 1 minute. If the 
process is missing for 3 minutes (3 probes), it triggers an Action Group.
5.  **Action:** Action Group sends an Email.
6.  **Visualization:** Azure Portal Dashboard displaying the process status.

### 2. Prerequisites

Before running the Terraform code, ensure you have the following:

1.  **Azure Subscription:** With permissions to create resources (Contributor role).
2.  **Tools Installed:**
    *   Terraform CLI (v1.0+)
    *   Azure CLI (`az`)
3.  **Authentication:** Log in via Azure CLI:
    ```bash
    az login
    az account set --subscription "<YOUR_SUBSCRIPTION_ID>"
    ```
4.  **Process Name:** Know the exact executable name you want to monitor (e.g., `notepad.exe`, 
`myapp.exe`).
5.  **Email Address:** A valid email address to receive alerts.

---

### 3. Terraform Implementation

Create a file named `main.tf`. This code creates the network, VM, Monitoring Agent, DCR, Alert, and 
Dashboard.

**Note:** This code uses the `azurerm` provider. Ensure you run `terraform init` before applying.

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

provider "azurerm" {
  features {}
}

# --- Variables ---
variable "resource_group_name" {
  description = "Name of the Resource Group"
  type        = string
  default     = "rg-process-monitor"
}

variable "location" {
  description = "Azure Region"
  type        = string
  default     = "East US"
}

variable "vm_admin_username" {
  description = "VM Admin Username"
  type        = string
  default     = "azureadmin"
}

variable "vm_admin_password" {
  description = "VM Admin Password (Must meet complexity requirements)"
  type        = string
  sensitive   = true
  # WARNING: In production, use KeyVault references instead of hardcoding
  default     = "ChangeMe123!" 
}

variable "monitor_process_name" {
  description = "The exact name of the process to monitor (e.g., notepad.exe)"
  type        = string
  default     = "notepad.exe"
}

variable "alert_email" {
  description = "Email address to receive alerts"
  type        = string
  default     = "admin@example.com"
}

# --- Resources ---

# 1. Resource Group
resource "azurerm_resource_group" "main" {
  name     = var.resource_group_name
  location = var.location
}

# 2. Networking (VNet, Subnet, NIC, Public IP)
resource "azurerm_virtual_network" "main" {
  name                = "vnet-monitor"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_subnet" "main" {
  name                 = "subnet-vm"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_public_ip" "main" {
  name                = "pip-vm"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_network_interface" "main" {
  name                = "nic-vm"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.main.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.main.id
  }
}

# 3. Windows Virtual Machine
resource "azurerm_windows_virtual_machine" "main" {
  name                = "vm-process-target"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  size                = "Standard_B2s"
  admin_username      = var.vm_admin_username
  admin_password      = var.vm_admin_password

  network_interface_ids = [azurerm_network_interface.main.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2019-Datacenter"
    version   = "latest"
  }

  # Enable Guest Patching and Monitor Agent prerequisites
  patch_mode    = "AutomaticByOS"
  patch_assessment_mode = "AutomaticByOS"
}

# 4. Log Analytics Workspace
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-process-monitor"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# 5. Data Collection Rule (Process Inventory)
resource "azurerm_monitor_data_collection_rule" "main" {
  name                = "dcr-process-inventory"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  destinations {
    log_analytics {
      name                  = "law-dest"
      workspace_resource_id = azurerm_log_analytics_workspace.main.id
    }
  }

  data_flow {
    streams      = ["Microsoft-Process"]
    destinations = ["law-dest"]
  }

  data_sources {
    process {
      name           = "process-source"
      streams        = ["Microsoft-Process"]
      sampling_frequency_in_seconds = 60
      # Monitor all processes, filtering happens in Alert Query to keep DCR generic
      # Or specify specific processes here if supported by provider version
    }
  }
}

# 6. Associate DCR to VM
resource "azurerm_monitor_data_collection_rule_association" "main" {
  name                    = "dcr-association"
  resource_id             = azurerm_windows_virtual_machine.main.id
  data_collection_rule_id = azurerm_monitor_data_collection_rule.main.id
}

# 7. Azure Monitor Agent Extension
resource "azurerm_virtual_machine_extension" "ama" {
  name                       = "AzureMonitorWindowsAgent"
  virtual_machine_id         = azurerm_windows_virtual_machine.main.id
  publisher                  = "Microsoft.Azure.Monitor"
  type                       = "AzureMonitorWindowsAgent"
  type_handler_version       = "1.1"
  auto_upgrade_minor_version = true

  # The DCR Association handles the config, but the extension must be installed
}

# 8. Action Group (Email)
resource "azurerm_monitor_action_group" "email_ag" {
  name                = "ag-process-alert"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "ProcAlert"

  email_action {
    name          = "SendToAdmin"
    email_address = var.alert_email
  }
}

# 9. Alert Rule (3 Failed Probes)
# Logic: Check every 1 min. If process not seen in last 3 mins, trigger.
resource "azurerm_monitor_scheduled_query_rules_alert_v2" "process_alert" {
  name                = "alert-process-down"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  evaluation_frequency = "PT1M" # 1 Minute
  window_duration      = "PT3M" # 3 Minutes (3 Probes)
  severity             = 3
  enabled              = true

  scopes = [azurerm_log_analytics_workspace.main.id]

  criteria {
    query          = <<-KQL
      Process 
      | where Name == "${var.monitor_process_name}" 
      | summarize LastSeen = max(TimeGenerated) by Computer 
      | where LastSeen < ago(3m)
    KQL
    time_aggregation_method = "Count"
    operator                = "GreaterThanOrEqual"
    threshold               = 1
    failing_periods {
      evaluation_frequency      = "PT1M"
      number_of_evaluation_periods = 3
    }
  }

  action {
    action_group_id = azurerm_monitor_action_group.email_ag.id
  }
}

# 10. Azure Dashboard
resource "azurerm_dashboard" "main" {
  name                = "dashboard-process-status"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location

  dashboard_properties = <<DASHBOARD
{
  "lenses": {
    "0": {
      "order": 0,
      "parts": {
        "0": {
          "position": { "x": 0, "y": 0, "colSpan": 4, "rowSpan": 2 },
          "metadata": {
            "inputs": [
              { "name": "Query", "value": "Process | where Name == '${var.monitor_process_name}' | 
summarize Count = count()" },
              { "name": "Workspace", "value": "${azurerm_log_analytics_workspace.main.id}" }
            ],
            "type": "Extension/HubsExtension/PartType/MonitorQueryPart"
          }
        },
        "1": {
          "position": { "x": 4, "y": 0, "colSpan": 4, "rowSpan": 2 },
          "metadata": {
            "inputs": [
              { "name": "AlertRuleId", "value": 
"${azurerm_monitor_scheduled_query_rules_alert_v2.process_alert.id}" }
            ],
            "type": "Extension/HubsExtension/PartType/AlertsListPart"
          }
        }
      }
    }
  },
  "metadata": {
    "model": {
      "timeRange": { "value": { "relative": { "duration": 24, "timeUnit": 1 } } }
    }
  }
}
DASHBOARD
}
```

### 4. Explanation of Key Components

1.  **Data Collection Rule (DCR):**
    *   The `streams = ["Microsoft-Process"]` setting tells the Azure Monitor Agent to inventory 
running processes on the VM and send them to Log Analytics.
    *   This populates the `Process` table in your Log Analytics Workspace.

2.  **Alert Logic (The "3 Probes"):**
    *   **KQL Query:** Checks the `Process` table. It looks for the specific process name and checks 
if the `TimeGenerated` is older than 3 minutes. If the process is running, `TimeGenerated` will be 
recent. If it stops, the timestamp stops updating.
    *   **Evaluation Frequency:** `PT1M` (1 Minute).
    *   **Window Duration:** `PT3M` (3 Minutes).
    *   **Failing Periods:** Configured to require 3 consecutive evaluations to trigger. This 
satisfies the "3 probes fail" requirement.

3.  **Dashboard:**
    *   The `azurerm_dashboard` resource deploys a JSON definition.
    *   **Tile 1:** A query tile showing the current count of the process (1 = Running, 0 = Stopped).
    *   **Tile 2:** An alert list tile showing the status of the specific alert rule created.

### 5. Deployment Steps

1.  **Initialize Terraform:**
    ```bash
    terraform init
    ```
2.  **Review Plan:**
    ```bash
    terraform plan -out=tfplan
    ```
    *Check that the email and process name variables are correct.*
3.  **Apply:**
    ```bash
    terraform apply tfplan
    ```
4.  **Wait for Agent:**
    *   After the VM is created, it takes approximately **5-10 minutes** for the Azure Monitor Agent 
to install, apply the DCR, and send the first Process Inventory data to Log Analytics.

### 6. Testing the Alert

1.  **Log into the VM:** Use RDP to connect to the created VM (`vm-process-target`).
2.  **Start the Process:** Open the process you defined (default is `notepad.exe`).
3.  **Verify Dashboard:** Go to the Azure Portal -> Dashboards -> `dashboard-process-status`. You 
should see the Count = 1.
4.  **Stop the Process:** Close `notepad.exe`.
5.  **Wait:** Wait for 3 to 5 minutes.
6.  **Check Email:** You should receive an email from "Microsoft Azure Monitor" indicating the alert 
has fired.
7.  **Check Dashboard:** The Alert List tile should show "Fired".

### 7. Cost & Cleanup

*   **Costs:** You will incur costs for the VM, Public IP, and Log Analytics data ingestion (Process 
inventory is lightweight but not free).
*   **Cleanup:** To remove all resources and stop billing:
    ```bash
    terraform destroy
    ```

### 8. Troubleshooting

*   **No Data in Log Analytics:** Ensure the `AzureMonitorWindowsAgent` extension status is 
"Provisioning succeeded" in the VM Extensions blade.
*   **Alert Not Firing:** Check the "Failed" tab in the Alert Rule. Run the KQL query manually in the 
Log Analytics Workspace to ensure it returns results when the process is stopped.
*   **Process Name:** Ensure you use the executable name (e.g., `java.exe`, `w3wp.exe`), not the 
service name. You can verify the name by running `Get-Process` in PowerShell on the VM.
