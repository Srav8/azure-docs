---
title: Troubleshooting common Azure Automation issues | Microsoft Docs
description: This article provides information to help troubleshoot and fix common Azure Automation errors.
services: automation
ms.service: automation
author: georgewallace
ms.author: gwallace
ms.date: 03/16/2018
ms.topic: article
manager: carmonm
tags: top-support-issue
keywords: automation error, troubleshooting, issue
---
# Troubleshooting common issues in Azure Automation 
This article provides help troubleshooting common errors you might experience in Azure Automation and suggests possible solutions to resolve them.

## Authentication errors when working with Azure Automation runbooks
### Scenario: Sign in to Azure Account failed
**Error:**
You receive the error "Unknown_user_type: Unknown User Type" when working with the Add-AzureAccount or Login-AzureRmAccount cmdlets.

**Reason for the error:**
This error occurs if the credential asset name is not valid or if the username and password that you used to set up the Automation credential asset are not valid.

**Troubleshooting tips:**
In order to determine what's wrong, take the following steps:  

1. Make sure that you don’t have any special characters, including the **@** character in the Automation credential asset name that you are using to connect to Azure.  
2. Check that you can use the username and password that are stored in the Azure Automation credential in your local PowerShell ISE editor. You can do this by running the following cmdlets in the PowerShell ISE:  

        $Cred = Get-Credential  
        #Using Azure Service Management   
        Add-AzureAccount –Credential $Cred  
        #Using Azure Resource Manager  
        Login-AzureRmAccount –Credential $Cred
3. If your authentication fails locally, this means that you haven’t set up your Azure Active Directory credentials properly. Refer to [Authenticating to Azure using Azure Active Directory](https://azure.microsoft.com/blog/azure-automation-authenticating-to-azure-using-azure-active-directory/) blog post to get the Azure Active Directory account set up correctly.  

### Scenario: Unable to find the Azure subscription
**Error:**
You receive the error "The subscription named ``<subscription name>`` cannot be found" when working with the Select-AzureSubscription or Select-AzureRmSubscription cmdlets.

**Reason for the error:**
This error occurs if the subscription name is not valid or if the Azure Active Directory user who is trying to get the subscription details is not configured as an admin of the subscription.

**Troubleshooting tips:**
In order to determine if you have properly authenticated to Azure and have access to the subscription you are trying to select, take the following steps:  

1. Make sure that you run the **Add-AzureAccount** before running the **Select-AzureSubscription** cmdlet.  
2. If you still see this error message, modify your code by adding the **Get-AzureSubscription** cmdlet following the **Add-AzureAccount** cmdlet and then execute the code. Now verify if the output of Get-AzureSubscription contains your subscription details.  

   * If you don't see any subscription details in the output, this means that the subscription isn’t initialized yet.  
   * If you do see the subscription details in the output, confirm that you are using the correct subscription name or ID with the **Select-AzureSubscription** cmdlet.   

### Scenario: Authentication to Azure failed because multi-factor authentication is enabled
**Error:**
You receive the error "Add-AzureAccount: AADSTS50079: Strong authentication enrollment (proof-up) is required" when authenticating to Azure with your Azure username and password.

**Reason for the error:**
If you have multi-factor authentication on your Azure account, you can't use an Azure Active Directory user to authenticate to Azure. Instead, you need to use a certificate or a service principal to authenticate to Azure.

**Troubleshooting tips:**
To use a certificate with the Azure classic deployment model cmdlets, refer to [creating and adding a certificate to manage Azure services.](http://blogs.technet.com/b/orchestrator/archive/2014/04/11/managing-azure-services-with-the-microsoft-azure-automation-preview-service.aspx) To use a service principal with Azure Resource Manager cmdlets, refer to [creating service principal using Azure portal](../azure-resource-manager/resource-group-create-service-principal-portal.md) and [authenticating a service principal with Azure Resource Manager.](../azure-resource-manager/resource-group-authenticate-service-principal.md)

## Common errors when working with runbooks
### Scenario: The runbook job start was attempted three times, but it failed to start each time
**Error:**
Your runbook fails with the error ""The job was tried three times but it failed."

**Reason for the error:**
This error can be caused by the following reasons:  

1. Memory Limit. There are documented limits on how much memory allocated to a Sandbox  [Automation service limits](../azure-subscription-service-limits.md#automation-limits) so a job may fail it if it is using more than 400 MB of memory. 

2. Module Incompatible. This can occur if module dependencies are not correct and if they are not, your runbook typically returns a "Command not found" or "Cannot bind parameter" message. 

**Troubleshooting tips:**
Any of the following solutions fix the problem:

* Suggested methods to work within the memory limit are to split the workload between multiple runbooks, not process as much data in memory, not to write unnecessary output from your runbooks, or consider how many checkpoints you write into your PowerShell workflow runbooks.  

* Update your Azure modules by following the steps [How to update Azure PowerShell modules in Azure Automation](automation-update-azure-modules.md).  


### Scenario: Runbook fails because of deserialized object
**Error:**
Your runbook fails with the error "Cannot bind parameter ``<ParameterName>``. Cannot convert the ``<ParameterType>`` value of type Deserialized ``<ParameterType>`` to type ``<ParameterType>``".

**Reason for the error:**
If your runbook is a PowerShell Workflow, it stores complex objects in a deserialized format in order to persist your runbook state if the workflow is suspended.

**Troubleshooting tips:**
Any of the following three solutions fix this problem:

1. If you are piping complex objects from one cmdlet to another, wrap these cmdlets in an InlineScript.
2. Pass the name or value that you need from the complex object instead of passing the entire object.
3. Use a PowerShell runbook instead of a PowerShell Workflow runbook.

### Scenario: Runbook job failed because the allocated quota exceeded
**Error:**
Your runbook job fails with the error "The quota for the monthly total job run time has been reached for this subscription".

**Reason for the error:**
This error occurs when the job execution exceeds the 500-minute free quota for your account. This quota applies to all types of job execution tasks such as testing a job, starting a job from the portal, executing a job by using webhooks and scheduling a job to execute by using either the Azure portal or in your datacenter. To learn more about pricing for Automation, see [Automation pricing](https://azure.microsoft.com/pricing/details/automation/).

**Troubleshooting tips:**
If you want to use more than 500 minutes of processing per month, you need to change your subscription from the Free tier to the Basic tier. You can upgrade to the Basic tier by taking the following steps:  

1. Sign in to your Azure subscription  
2. Select the Automation account you wish to upgrade  
3. Click on **Settings** > **Pricing**.
4. Click **Enable** on page bottom to upgrade your account to the **Basic** tier.

### Scenario: Cmdlet not recognized when executing a runbook
**Error:**
Your runbook job fails with the error "``<cmdlet name>``: The term ``<cmdlet name>`` is not recognized as the name of a cmdlet, function, script file, or operable program."

**Reason for the error:**
This error is caused when the PowerShell engine cannot find the cmdlet you are using in your runbook. This could be because the module containing the cmdlet is missing from the account, there is a name conflict with a runbook name, or the cmdlet also exists in another module and Automation cannot resolve the name.

**Troubleshooting tips:**
Any of the following solutions fix the problem:  

* Check that you have entered the cmdlet name correctly.  
* Make sure the cmdlet exists in your Automation account and that there are no conflicts. To verify if the cmdlet is present, open a runbook in edit mode and search for the cmdlet you want to find in the library or run **Get-Command ``<CommandName>``**. Once you have validated that the cmdlet is available to the account, and that there are no name conflicts with other cmdlets or runbooks, add it to the canvas and ensure that you are using a valid parameter set in your runbook.  
* If you do have a name conflict and the cmdlet is available in two different modules, you can resolve this by using the fully qualified name for the cmdlet. For example, you can use **ModuleName\CmdletName**.  
* If you are executing the runbook on-premises in a hybrid worker group, then make sure that the module/cmdlet is installed on the machine that hosts the hybrid worker.

### Scenario: A long running runbook consistently fails with the exception: "The job cannot continue running because it was repeatedly evicted from the same checkpoint".
**Reason for the error:**
This behavior is by design due to the "Fair Share" monitoring of processes within Azure Automation, which automatically suspends a runbook if it executes longer than three hours. However, the error message returned does not provide "what next" options. A runbook can be suspended for a number of reasons. Suspends happen mostly due to errors. For example, an uncaught exception in a runbook, a network failure, or a crash on the Runbook Worker running the runbook, all cause the runbook to be suspended and start from its last checkpoint when resumed.

**Troubleshooting tips:**
The documented solution to avoid this issue is to use Checkpoints in a workflow. To learn more, refer to [Learning PowerShell Workflows](automation-powershell-workflow.md#checkpoints). A more thorough explanation of "Fair Share" and Checkpoint can be found in this blog article [Using Checkpoints in Runbooks](https://azure.microsoft.com/blog/azure-automation-reliable-fault-tolerant-runbook-execution-using-checkpoints/).

## Common errors when importing modules
### Scenario: Module fails to import or cmdlets can't be executed after importing
**Error:**
A module fails to import or imports successfully, but no cmdlets are extracted.

**Reason for the error:**
Some common reasons that a module might not successfully import to Azure Automation are:

* The structure does not match the structure that Automation needs it to be in.
* The module is dependent on another module that has not been deployed to your Automation account.
* The module is missing its dependencies in the folder.
* The **New-AzureRmAutomationModule** cmdlet is being used to upload the module, and you have not given the full storage path or have not loaded the module by using a publicly accessible URL.

**Troubleshooting tips:**
Any of the following solutions fix the problem:

* Make sure that the module follows the following format:
  ModuleName.Zip **->** ModuleName or Version Number **->** (ModuleName.psm1, ModuleName.psd1)
* Open the .psd1 file and see if the module has any dependencies. If it does, upload these modules to the Automation account.
* Make sure that any referenced .dlls are present in the module folder.

## Common errors when working with Desired State Configuration (DSC)
### Scenario: Node is in failed status with a "Not found" error
**Error:**
The node has a report with **Failed** status and containing the error "The attempt to get the action from server https://``<url>``//accounts/``<account-id>``/Nodes(AgentId=``<agent-id>``)/GetDscAction failed because a valid configuration ``<guid>`` cannot be found."

**Reason for the error:**
This error typically occurs when the node is assigned to a configuration name (for example, ABC) instead of a node configuration name (for example, ABC.WebServer).

**Troubleshooting tips:**

* Make sure that you are assigning the node with "node configuration name" and not the "configuration name".
* You can assign a node configuration to a node using Azure portal or with a PowerShell cmdlet.

  * In order to assign a node configuration to a node using Azure portal, open the **DSC Nodes** page, then select a node and click on **Assign node configuration** button.  
  * In order to assign a node configuration to a node using PowerShell cmdlet, use **Set-AzureRmAutomationDscNode** cmdlet

### Scenario: No node configurations (MOF files) were produced when a configuration is compiled
**Error:**
Your DSC compilation job suspends with the error: "Compilation completed successfully, but no node configuration.mofs were generated".

**Reason for the error:**
When the expression following the **Node** keyword in the DSC configuration evaluates to `$null`, then no node configurations are produced.

**Troubleshooting tips:**
Any of the following solutions fix the problem:

* Make sure that the expression next to the **Node** keyword in the configuration definition is not evaluating to $null.
* If you are passing ConfigurationData when compiling the configuration, make sure that you are passing the expected values that the configuration requires from [ConfigurationData](automation-dsc-compile.md#configurationdata).

### Scenario: The DSC node report becomes stuck "in progress" state
**Error:**
DSC Agent outputs "No instance found with given property values."

**Reason for the error:**
You have upgraded your WMF version and have corrupted WMI.

**Troubleshooting tips:**
To fix the issue follow the instructions in the [DSC known issues and limitations](https://msdn.microsoft.com/powershell/wmf/5.0/limitation_dsc) article.

### Scenario: Unable to use a credential in a DSC configuration
**Error:**
Your DSC compilation job was suspended with the error: "System.InvalidOperationException error processing property 'Credential' of type ``<some resource name>``: Converting and storing an encrypted password as plaintext is allowed only if PSDscAllowPlainTextPassword is set to true".

**Reason for the error:**
You have used a credential in a configuration but didn’t provide proper **ConfigurationData** to set **PSDscAllowPlainTextPassword** to true for each node configuration.

**Troubleshooting tips:**

* Make sure to pass in the proper **ConfigurationData** to set **PSDscAllowPlainTextPassword** to true for each node configuration mentioned in the configuration. For more information, see [assets in Azure Automation DSC](automation-dsc-compile.md#assets).

## Common errors when onboarding solutions

When onboarding solutions you may encounter errors. The following is a listing of the common errors you may run across.

### ComputerGroupQueryFormatError

**Reason for the error:**

This error code means that the saved search computer group query used to target the solution was not formatted correctly. You may have altered the query, or it may have been altered by the system.

**Troubleshooting tips:**

You can delete the query for this solution, and reonboard the solution, which recreates the query. The query can be found within your workspace, under **Saved searches**. The name of the query is **MicrosoftDefaultComputerGroup**, and the category of the query is the name of the solution associated with this query. If multiple solutions are enabled, the **MicrosoftDefaultComputerGroup** shows multiple times under **Saved Searches**.

### PolicyViolation

**Reason for the error:**

This error code means that the deployment failed due to violation of one or more policies.

**Troubleshooting tips:**

In order to successfully deploy the solution, you need to consider altering the indicated policy. As there are many different types of policies that can be defined, the specific changes required depend on the policy that is violated. For example, if a policy was defined on a resource group that denied permission to change the contents of certain types of resources within that resource group, you could, for example, do any of the following:

*	Remove the policy altogether.
* Try to onboard to a different resource group.
* Revise the policy, by, for example:
   * Re-targeting the policy to a a specific resource (such as to a specific Automation account).
   * Revising the set of resources that policy was configured to deny.

Check the notifications in the top right corner of the Azure portal or navigate to the resource group that contains your automation account and select **Deployments** under **Settings** to view the failed deployment. To learn more about Azure Policy visit: [Overview of Azure Policy](../azure-policy/azure-policy-introduction.md?toc=%2fazure%2fautomation%2ftoc.json).

## Next steps

If you have followed the preceding troubleshooting steps and can't find the answer, you can review the additional support following options:

* Get help from Azure experts. Submit your issue to the [MSDN Azure or Stack Overflow forums](https://azure.microsoft.com/support/forums/).
* File an Azure support incident. Go to the [Azure Support site](https://azure.microsoft.com/support/options/) and click **Get support** under **Technical and billing support**.
* Post a Script Request on [Script Center](https://azure.microsoft.com/documentation/scripts/) if you are looking for an Azure Automation runbook solution or an integration module.
* Post feedback or feature requests for Azure Automation on [User Voice](https://feedback.azure.com/forums/34192--general-feedback).
