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

### Optionally apply tailoring to check & remediation content

### Manually apply remediation

### Rescan environment to confirm sucessful remediation

### Lather, rinse, repeat until available remediations applied

### Automating applying remediation - pros, cons, and recommendations

### Manual Remeditions



## Next Steps: Using ACS to view and export compliance



