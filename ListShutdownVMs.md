# List VMs that have been shutdown

It's easy to query the Graph API for VMs that are shutdown:
```
az graph query -q "HealthResources | where type =~ 'microsoft.resourcehealth/availabilitystatuses' | where properties.availabilityState =~ 'unknown' | summarize count() by subscriptionId, AvailabilityState = tostring(properties.availabilityState)"
```

What about if we would like to know for how long a VM has been shutdown? That data isn't availble in the HealthResources table used in the first query.
We can look at the Activity log instead, if we send it to Log analytics.

One way of looking at the data is to look for the action DEALLOCATE:

```
AzureActivity
| where ResourceProviderValue == 'MICROSOFT.COMPUTE'
| where CategoryValue == 'Administrative'
| sort by TimeGenerated desc 
| extend prop= parse_json(Properties)
| summarize arg_max(TimeGenerated, *) by _ResourceId
| where OperationNameValue == 'MICROSOFT.COMPUTE/VIRTUALMACHINES/DEALLOCATE/ACTION'
| project ShutdownTime=TimeGenerated, VMName=prop.resource, ShutdownFor=datetime_diff('minute', now(), TimeGenerated)
```

Another way could be to look at Healthstatus for VMs, to see for how long they have been unavailable:

```
AzureActivity
| where ResourceProviderValue == 'MICROSOFT.COMPUTE'
| where CategoryValue == 'ResourceHealth'
| sort by TimeGenerated desc 
| extend prop= parse_json(Properties)
| summarize arg_max(TimeGenerated, *) by _ResourceId
| where prop.currentHealthStatus != 'Available'
| project ShutdownTime=TimeGenerated, VMName=prop.resource, ShutdownFor=datetime_diff('minute', now(), TimeGenerated)
```

In my tests these two queries have given me the same result, but I have not tried this out in a larger environment with more variation of configuration.

That should however do it. The downside with this solution is that we don't tend to save the data in Log analytics for very long (since it may become expensive for large amoutns of data).

If you run these queries on a Log analytics workspace, remember to set the time range to a higher value than the default.