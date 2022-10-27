

Get Compliance status for rules
```
oc get compliancecheckresults
```

Get compliance status for rules with remediations available:
```
oc get compliancecheckresults -l 'compliance.openshift.io/automated-remediation'
```

Filter compliance status for failed rules with automated remediaitons available
```
oc get compliancecheckresults -l 'compliance.openshift.io/check-status=FAIL,compliance.openshift.io/automated-remediation'
```
Example output:
```
NAME                                                          STATUS   SEVERITY
rhcos4-moderate-master-kernel-module-bluetooth-disabled       FAIL     medium
rhcos4-moderate-master-kernel-module-can-disabled             FAIL     medium
rhcos4-moderate-master-kernel-module-cramfs-disabled          FAIL     low
```

Confirm Status of remediation:
```
oc get complianceremediation rhcos4-moderate-master-kernel-module-bluetooth-disabled
```
Output:
```
NAME                                                      STATE
rhcos4-moderate-master-kernel-module-bluetooth-disabled   NotApplied
```

Inspect remediation:
```
oc describe complianceremediation rhcos4-moderate-master-kernel-module-bluetooth-disabled
```
Output:
```
Name:         rhcos4-moderate-master-kernel-module-bluetooth-disabled
Namespace:    openshift-compliance
Labels:       compliance.openshift.io/scan-name=rhcos4-moderate-master
              compliance.openshift.io/suite=rhcos4-nist-moderate
Annotations:  <none>
API Version:  compliance.openshift.io/v1alpha1
Kind:         ComplianceRemediation
Metadata:
  Creation Timestamp:  2022-10-19T15:43:52Z
  Generation:          1
  Managed Fields:
    API Version:  compliance.openshift.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        f:applicationState:
    Manager:      compliance-operator
    Operation:    Update
    Subresource:  status
    Time:         2022-10-24T18:27:39Z
  Owner References:
    API Version:           compliance.openshift.io/v1alpha1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  ComplianceCheckResult
    Name:                  rhcos4-moderate-master-kernel-module-bluetooth-disabled
    UID:                   f91ec193-8910-4420-a88d-29ec2a7d1e15
  Resource Version:        7949313
  UID:                     d87f745d-3eff-43fc-acf0-8f878d9af36b
Spec:
  Apply:  false
  Current:
    Object:
      API Version:  machineconfiguration.openshift.io/v1
      Kind:         MachineConfig
      Spec:
        Config:
          Ignition:
            Version:  3.1.0
          Storage:
            Files:
              Contents:
                Source:   data:,install%20bluetooth%20/bin/true%0A
              Mode:       420
              Overwrite:  true
              Path:       /etc/modprobe.d/75-kernel_module_bluetooth_disabled.conf
  Outdated:
  Type:  Configuration
Status:
  Application State:  NotApplied
Events:               <none>
```

Pause affected MCP's
```
oc patch mcp/{master|worker|other} --patch '{"spec":{"paused":true}}' --type=merge
```

Apply remediatons

``
oc patch complianceremediations/{rule} --patch '{"spec":{"apply":true}}' --type=merge
``



