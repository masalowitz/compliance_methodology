# Compliance Operator

TL;DR: tips and tricks to manage/assess compliance.

Long version:

This repo contains useful Compliance Operator tools and techniques to help rapidly assess and secure OpenShift clusters using the Compliance Operator available in the OpenShift Operator Marketplace. 

This is a work in progress... 

# Compliance Methodology

## General Principles

A well-implemented enterprise compliance policy begins with determining the baseline standard which most closely aligns with the security requirements of your organization, whether it be a US DOD STIG, a US Civilian NIST control set, other Non-US standards, or even commercial guidance available from various published sources. In addition to the generally published policies, your organization may have additional rules or more restrictive rules than the published document. A healthy enterprise compliance policy is the sum of the constituent parts created by comparing and contrasting all available sources of policy, then [usually] selecting the more restrictive setting of any conflicting controls.

Changes to the baseline control set in order to adapt to enterprise policies are done with what is called tailoring. By tailoring compliance content you can adapt and measure organizational compliance to the aggregate organizational standard and in many cases apply changes via automated means. By having a published list of tailored items unique to your organization you provide a documented and auditable asset of approved enterprise policy.

Examples might be items such as changing a password history from a DOD mandated 5 to a higher setting such as 24. In this example, if you merely set it to 24, compliance scans will show as failed becuase the number is not 5; further automated redmiation would set it to 5 in violation of an organizational policy that says 24. By tailoring that value to match 24, if it is not 24 it will be remediated to 24, and the tailoring itself is evidence that can be tied to an overriding (and published) organizational policy.

In general, most organizations have at least a few changes to "off-the-shelf" policies. The more of them you catch and tailor into your compliance, the better off you'll be in the long run.

## Specific Workflow for Openshift

### Install Compliance Operator

You can install the Compliance Operator from both web and CLI instances, choose whatever method works best for you and your environment. 

If you have a small number of clusters, the Operator Catalog method may suffice. If you are managing a large number of clusters, applying the namespace manifests via ACM, GitOps, or other tooling is the best solution. 

Regardless of the install method, it is important to keep the operator up to date to receive content updates and bug fixes. The content update will include new policies, rules, checks, and remediations as released in the upstream Compliance projects. If automated updates are not available due to a disconnected environment, a regular practice should include checking for updated releases and importing them to your internal registory for use. 

Official Docs: [Compliance Operator Installation](https://docs.openshift.com/container-platform/4.11/security/compliance_operator/compliance-operator-installation.html)

### Accept or alter scan schedule in ScanSettings

ScanSettings can be though of as the "how" and the "when" to scan. They contain details that tell Compliance Operator on what node types to run, when to run them, how much storage and of what type to use to retain scans, and how long to retain the results. What it does NOT do is define thecontent or profiles to scan.. that is the province of the ScanSettingBinding.

Compliance Operator comes with two default ScanSetting configurations, "default" and "default-auto-apply". Both ScanSetting files contain a default execution time in standard cron format of 01:00hrs daily system time. If necessary, adjust these for acceptable time period where system workloads may be least impacted.

Default is a scan-only setting. 

```
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSetting
metadata:
  name: default
  namespace: openshift-compliance
rawResultStorage:
  nodeSelector:
    node-role.kubernetes.io/master: ""
  pvAccessModes:
  - ReadWriteOnce
  rotation: 3
  size: 1Gi
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
    operator: Exists
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  - effect: NoSchedule
    key: node.kubernetes.io/memory-pressure
    operator: Exists
roles:
- master
- worker
scanTolerations:
- operator: Exists
schedule: 0 1 * * *
showNotApplicable: false
strictNodeScan: true
```


Default-auto-apply is a non-interative scheduled remediation of rules. it adds the following arguments to the yaml:

```
autoApplyRemediations: true
autoUpdateRemediations: true
```

Additional options can be configured in the ScanSettings including scan retention depth, storage types and amounts, taint tolerations, etc. The default files have been included [here for reference](/manifests/ScanSetting/) 

Official Docs: [Compliance Operator Scans](https://docs.openshift.com/container-platform/4.11/security/compliance_operator/compliance-scans.html#running-compliance-scans_compliance-operator-scans)

Neither of these are "active" until they have been bound to by a ScanSettingBinding.

### Create ScanSettingBindings for enterprise-appropriate scans

ScanSettingBinding is what links a compliance profile to the execution environment specified in the ScanSettings.

```
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSettingBinding
metadata:
  name: rhcos4-nist-high
  namespace: openshift-compliance
profiles:
- apiGroup: compliance.openshift.io/v1alpha1
  kind: Profile
  name: rhcos4-high
settingsRef:
  apiGroup: compliance.openshift.io/v1alpha1
  kind: ScanSetting
  name: default
  ```

Creation of a ScanSettingBinding triggers an immediate scan of the system. Once created, there are two ways to trigger a scan/rescan of the system: change the cron entry or execute a [rescan](/Methodology.md#rescan-environment-to-confirm-sucessful-remediation)

### Confirm Execution of scans

There are a couple of ways to determine the status of a scan. If it says DONE here, no need to look farther:

```
[foo@bar]$ oc -n openshift-compliance get compliancescans
NAME                    PHASE   RESULT
ocp4-high               DONE    NON-COMPLIANT
ocp4-high-node-master   DONE    NON-COMPLIANT
ocp4-high-node-worker   DONE    NON-COMPLIANT
rhcos4-high-master      DONE    NON-COMPLIANT
rhcos4-high-worker      DONE    NON-COMPLIANT
[foo@bar]$ 
```



### Optional but highly recommended: Install ACS 

Now is a really good time to address the timeless question: "What did it look like before?" 

There are a couple of ways to extract compliance data, but ACS is by far the easiest. If you'd like a pre-remediation snapshot of your complaince state to keep the data for comparison, skip ahead to the ACS section [Using ACS to View and Export Compliance](/Methodology.md#next-steps-using-acs-to-view-and-export-compliance) and return when you have your CSV(s) safely tucked away.

### Identify items which FAIL analysis

A great place to start is just to get an idea of whats failing and what is passing on your cluster. "oc get compliancecheckresults" is the way to do this, but it gives you all profiles scanned, and all pass/fail/manual results. I like to get a quick look at some stats...overall, maybe per profile, and counts of pass/fail. 
```
[foo@bar]$ oc get compliancecheckresults | grep ocp4-high-node | grep FAIL
ocp4-high-node-master-directory-access-var-log-kube-audit                                       FAIL     medium
ocp4-high-node-master-directory-access-var-log-oauth-audit                                      FAIL     medium
ocp4-high-node-master-directory-access-var-log-ocp-audit                                        FAIL     medium
ocp4-high-node-master-file-permissions-etcd-data-dir                                            FAIL     medium
ocp4-high-node-master-kubelet-enable-protect-kernel-defaults                                    FAIL     medium
ocp4-high-node-master-kubelet-enable-protect-kernel-sysctl                                      FAIL     medium
ocp4-high-node-master-reject-unsigned-images-by-default                                         FAIL     medium
ocp4-high-node-worker-directory-access-var-log-kube-audit                                       FAIL     medium
ocp4-high-node-worker-directory-access-var-log-oauth-audit                                      FAIL     medium
ocp4-high-node-worker-directory-access-var-log-ocp-audit                                        FAIL     medium
ocp4-high-node-worker-file-permissions-etcd-data-dir                                            FAIL     medium
ocp4-high-node-worker-kubelet-enable-protect-kernel-defaults                                    FAIL     medium
ocp4-high-node-worker-kubelet-enable-protect-kernel-sysctl                                      FAIL     medium
ocp4-high-node-worker-reject-unsigned-images-by-default                                         FAIL     medium

[foo@bar]$ oc get compliancecheckresults | grep ocp4-high-node | grep -c FAIL
14

[foo@bar]$ oc get compliancecheckresults | grep ocp4-high-node | grep -c PASS
166

[foo@bar]$ oc get compliancecheckresults | grep ocp4-high-node | grep -c MANUAL
6

```

So for this particular standard, we're in pretty decent shape. We know we have some manual work to do, we can get to that later. For now, however, lets see what of these failures we can remediate with automation using filters...



### Identify failing scans with remediation content


```
oc get compliancecheckresults -l 'compliance.openshift.io/check-status=FAIL,compliance.openshift.io/automated-remediation'
```

So for my environment, 10 of the rules liste above have automated remediation:
```
[foo@bar]$ oc get compliancecheckresults -l 'compliance.openshift.io/check-status=FAIL,compliance.openshift.io/automated-remediation' | grep -c ocp4-high-node
10

[foo@bar]$ oc get compliancecheckresults -l 'compliance.openshift.io/check-status=FAIL,compliance.openshift.io/automated-remediation' | grep ocp4-high-node
ocp4-high-node-master-directory-access-var-log-kube-audit                                       FAIL     medium
ocp4-high-node-master-directory-access-var-log-oauth-audit                                      FAIL     medium
ocp4-high-node-master-directory-access-var-log-ocp-audit                                        FAIL     medium
ocp4-high-node-master-kubelet-enable-protect-kernel-defaults                                    FAIL     medium
ocp4-high-node-master-kubelet-enable-protect-kernel-sysctl                                      FAIL     medium
ocp4-high-node-worker-directory-access-var-log-kube-audit                                       FAIL     medium
ocp4-high-node-worker-directory-access-var-log-oauth-audit                                      FAIL     medium
ocp4-high-node-worker-directory-access-var-log-ocp-audit                                        FAIL     medium
ocp4-high-node-worker-kubelet-enable-protect-kernel-defaults                                    FAIL     medium
ocp4-high-node-worker-kubelet-enable-protect-kernel-sysctl                                      FAIL     medium
[foo@bar]$ 

```

### Review and confirm remediations

So before we apply a remediation for the first time, we should have a look at what it is, what criteria it uses to determine compliance, and is the embedded remediation something appropriate for our server? Let's look at one...

```
[foo@bar]$ oc describe complianceremediation ocp4-high-node-master-kubelet-enable-protect-kernel-defaults
Name:         ocp4-high-node-master-kubelet-enable-protect-kernel-defaults
Namespace:    openshift-compliance
Labels:       compliance.openshift.io/has-unmet-dependencies=
              compliance.openshift.io/scan-name=ocp4-high-node-master
              compliance.openshift.io/suite=ocp4-high-node
Annotations:  compliance.openshift.io/depends-on: xccdf_org.ssgproject.content_rule_kubelet_enable_protect_kernel_sysctl
API Version:  compliance.openshift.io/v1alpha1
Kind:         ComplianceRemediation
Metadata:
  Creation Timestamp:  2022-11-03T19:23:49Z
  Generation:          1
  Managed Fields:
    API Version:  compliance.openshift.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:applicationState:
    Manager:      compliance-operator
    Operation:    Update
    Subresource:  status
    Time:         2022-11-03T19:23:49Z
    API Version:  compliance.openshift.io/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:compliance.openshift.io/depends-on:
        f:labels:
          .:
          f:compliance.openshift.io/has-unmet-dependencies:
          f:compliance.openshift.io/scan-name:
          f:compliance.openshift.io/suite:
        f:ownerReferences:
          .:
          k:{"uid":"2309c739-28c1-4f78-937c-db824298de8c"}:
      f:spec:
        .:
        f:apply:
        f:current:
          .:
          f:object:
            .:
            f:apiVersion:
            f:kind:
            f:metadata:
            f:spec:
              .:
              f:kubeletConfig:
                .:
                f:protectKernelDefaults:
        f:outdated:
        f:type:
    Manager:    compliance-operator
    Operation:  Update
    Time:       2022-11-08T14:51:12Z
  Owner References:
    API Version:           compliance.openshift.io/v1alpha1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  ComplianceCheckResult
    Name:                  ocp4-high-node-master-kubelet-enable-protect-kernel-defaults
    UID:                   2309c739-28c1-4f78-937c-db824298de8c
  Resource Version:        623819
  UID:                     4ae59518-d2fc-41e3-b11a-53f58055381c
Spec:
  Apply:  false
  Current:
    Object:
      API Version:  machineconfiguration.openshift.io/v1
      Kind:         KubeletConfig
      Metadata:
      Spec:
        Kubelet Config:
          Protect Kernel Defaults:  true
  Outdated:
  Type:  Configuration
Status:
  Application State:  NotApplied
Events:               <none>
```

In this case, the remediation is to set 

```
  Current:
    Object:
      API Version:  machineconfiguration.openshift.io/v1
      Kind:         KubeletConfig
      Metadata:
      Spec:
        Kubelet Config:
          Protect Kernel Defaults:  true
```

in another remediation for a CoreOS ignition config (truncated):

```
oc describe complianceremediation rhcos4-high-worker-sysctl-net-ipv4-conf-default-send-redirects

<truncated>

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
                Source:   data:,net.ipv4.conf.default.send_redirects%3D0%0A
              Mode:       420
              Overwrite:  true
              Path:       /etc/sysctl.d/75-sysctl_net_ipv4_conf_default_send_redirects.conf
  Outdated:
  Type:  Configuration
```

In this case, you're creating file /etc/sysctl.d/75-sysctl_net_ipv4_conf_default_send_redirects.conf and setting the appropriate value.


### Optionally apply tailoring to check & remediation content

### Manually apply remediation

When applying CoreOS remediations, either via openshift console or command line, pause machineconfig updates prior to applying the remediations. If you do not, after every sucessfull application of a CoreOS remediation, the machiencofnig will rebuild and the nodes will boot into it. 

```
oc patch mcp/{master|worker|other} --patch '{"spec":{"paused":true}}' --type=merge
```




```
oc patch complianceremediations/{rule} --patch '{"spec":{"apply":true}}' --type=merge
```

### Rescan environment to confirm sucessful remediation
```
 oc annotate compliancescans/{scan name} compliance.openshift.io/rescan=

```

### Reduce some typing:

I like to cheat and make a quick and dirty shell scripts for regular actions:

~/bin/rescan:
```
#!/bin/sh
oc annotate compliancescans/$1 compliance.openshift.io/rescan=
```

~/bin/apply_stig:
```
#!/bin/sh
oc patch complianceremediations/$1 --patch '{"spec":{"apply":true}}' --type=merge
```

### Lather, rinse, repeat until available remediations applied

Now that we have some easy commands to use, we can start batching compliance data together into as many groups as we would like to match our comfort level with the content...

Lets start with a basic filter on the status of the rule... "FAIL" and the presence of an automated remediation "compliance.openshift.io/automated-remediation" then add a filter for the body of content we wish to address first (or don't, to do the whole shebang at once...) and filter for the rule name.

```
for rule in `oc get compliancecheckresults -l 'compliance.openshift.io/check-status=FAIL,compliance.openshift.io/automated-remediation,compliance.openshift.io/scan-name=ocp4-high-node-master' | awk '{print $1}'` ; do \
  echo ${rule} ; \
done
```
This will output a list of currently unremediated rules which contain automated remediation, filtered for OCP4 node rules:
```
ocp4-high-node-master-directory-access-var-log-kube-audit
ocp4-high-node-master-directory-access-var-log-oauth-audit
ocp4-high-node-master-directory-access-var-log-ocp-audit
ocp4-high-node-master-kubelet-enable-protect-kernel-defaults
ocp4-high-node-master-kubelet-enable-protect-kernel-sysctl
```

If that list were excessive, adding a grep to filter for directory as below:
```
for rule in `oc get compliancecheckresults -l 'compliance.openshift.io/check-status=FAIL,compliance.openshift.io/automated-remediation,compliance.openshift.io/scan-name=ocp4-high-node-master' | grep directory | awk '{print $1}'` ; \
  do echo ${rule} ; \
done

ocp4-high-node-master-directory-access-var-log-kube-audit
ocp4-high-node-master-directory-access-var-log-oauth-audit
ocp4-high-node-master-directory-access-var-log-ocp-audit
```
Then once you're satisfied with the ruleset, swap your echo for your handy apply_stig (Be sure to pause MCP updates!!!):

```
for rule in `oc get compliancecheckresults -l 'compliance.openshift.io/check-status=FAIL,compliance.openshift.io/automated-remediation,compliance.openshift.io/scan-name=ocp4-high-node-master' | grep directory | awk '{print $1}'` ; \
  do ~/bin/apply_stig ${rule} ; \
done
```

Filtering the rules may not seem very necessary in the OCP4 high NIST controlset, but when you add CoreOS controls, the number of remediations (191 at the time of this post) is rather extensive...filtering by sysctl rules, kernel modules, or audit rules can limit the scope of change quite effectively.  

Once your updates are applied, unpause your MCP updates and if reboots are necessary, the automation will schedule them. When the cluster updates are complete, trigger a rescan of your server and check the remediation status of the rules you applied. 

### Automating applying remediation - pros, cons, and recommendations

As you can see above, it is a fairly trivial exercise to apply a significant number of preconfigured remediations in fairly short order if so desired; resulting in a fixed compliance configuration that should then be monitored for changes on a regular basis: new rules in new content and/or rules whose compliance state has changed. 

Compliance Operator also contain the capability to schedule persistent remediation runs on whatever schedule you like. When a variance is detected the operator will (re)apply the the correct remediaton and trigger an MCP update if applicable.

There are some cons to using this method. First, you enable the content body itself to be the arbiter of compliance without applying administrative review. Second, should a check/remediation content mismatch arise, you could end up with weekly (or daily...) infrastructure restarts for no reason other than a typo.

Depending on your comfort level and enforcement requirements, manually applying new remediation content and monitoring daily may be your best approach. if policy or economy of scale dictates full automation, determining an appropriate execution period AND ensuring that no subsequent scans show FAIL after remediation may be the most sound approach.

Should full automation be your choice, set the schedule in the ScanSetting for default-auto-apply, confirm the other details for you infrastucture, then create a ScanSettingBinding to activate it. I recommend scheduling it to run before daily scans kick off, but different environments have different requirements... schedule as appropriate for you environment, approved maintenance windows, etc. 


### Manual Remediations


## Next Steps: Using ACS to view and export compliance


