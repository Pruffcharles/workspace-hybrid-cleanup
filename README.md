# workspace-hybrid-cleanup
Many customers use SSM to manage their Amazon Workspaces by registering them as *Hybrid managed nodes* [1]. SSM offers a *standard-instances tier* and an *advanced-instances tier* for non-EC2 machines in a hybrid and multicloud environment. You can register up to 1,000 standard hybrid-activated nodes per account per AWS Region at no additional cost. However, registering more than 1,000 hybrid nodes requires that you activate the advanced-instances tier [2]. There is a charge to use the advanced-instances tier. For customers with a large fleet of Amazon Workspaces registered as *Hybrid managed nodes*, they will have to manually clean up terminated Workspaces from Fleet Manager to remain in the *standard-instances tier*. The solution below provides an automated solution to clean up the  *Hybrid managed nodes* from Fleet Manager to remove the administrative overhead from SRE/DevOops/Ops Teams required to manually deregister the terminated managed nodes (Amazon Workspaces) from Fleet Manager and ensure they remain within the *standard-instances tier*.  

**Developed by Charles Adebayo v1 March 2025**

_________________

**Step 1: Create State Manager Association to scan for Hybrid managed nodes and create Custom Inventory**

Deploy Cloudformation Template [CFT-WorkspaceMapping.yaml](https://github.com/Pruffcharles/workspace-hybrid-cleanup/blob/main/CFT-WorkspaceMapping.yaml)in the target region(s) where the Amazon Workspaces were created. This template creates the following resources:

    SSM automation role
    SSM automation custom runbook WorkspaceMappingAssociation
    SSM State Manager Association to invoke the runbook every 30 minutes (you can update this in the CFT template to desired cron)

The runbook does the following:

1. scans for hybrid nodes `mi-123456abcdef` in list of managed nodes on a cron schedule
2. maps the hybrid nodes `mi-123456abcdef` to `workspace IDs`
3. creates Custom Inventory `Custom:WorkspaceMapping` in SSM Inventory with the managed node ID `instanceid` mapped to `workspaceID` and `computerName`. You can verify the custom inventory was created successfully for a Worskspace registered as hybrid instance using the cli command below:

```
aws ssm list-inventory-entries --instance-id mi-0aef39082e7f3e774 --type-name "Custom:WorkspaceMapping"
{
    "TypeName": "Custom:WorkspaceMapping",
    "InstanceId": "mi-0aef39082e7f3e774",
    "SchemaVersion": "1.0",
    "CaptureTime": "2025-02-10T21:56:09Z",
    "Entries": [
        {
            "ComputerName": "wsamzn-tnlmsoie",
            "WorkspaceId": "ws-khtxjq13q"
        }
    ]
}
```

**Step 2: Create EventBridge rule to track TerminateWorkspaces API events and invoke AWS Automation to De-register Managed Instance**

Deploy Cloudformation Template [deregister_managed_nodes_after_workspace_termination.yaml](https://github.com/Pruffcharles/workspace-hybrid-cleanup/blob/main/deregister_managed_nodes_after_workspace_termination.yaml) in the target region. This template creates the following resources:

    SSM automation role
    SSM automation custom runbook workspace-termination-automation-DeregisterManagedInstanceAutomation
    SSM EventBridge Rule to track TerminateWorkspaces API events and target automation runbook above

When invoked, the runbook takes the `workspaceID` that was terminated as input parameter and does the following:

1. fetches the mapping of `workspacesId` to it's managed node ID i.e. `mi-123456abcdef` from SSM Inventory using `ListInventoryEntries` API using `workspaceId` and `ComputerName` filter. 
2. de-registers the managed instance `mi-123456abcdef` from Fleet Manager using `DeregisterManagedInstance` API.

**References**

[1] https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-hybrid-multicloud.html 

[2] https://docs.aws.amazon.com/systems-manager/latest/userguide/fleet-manager-configure-instance-tiers.html
