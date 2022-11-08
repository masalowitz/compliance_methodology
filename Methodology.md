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

Compliance Operator comes with two default ScanSetting configurations, "default" and "default-auto-apply". Both ScanSetting files contain a default execution time in standard cron format of 01:00hrs daily system time. If necessary, adjust these for acceptable time period where system workloads may be least impacted.

Default is a scan-only setting. 

Default-auto-apply is a non-interative scheduled remediation of rules.

Neither of these are "active" until they have been bound to by a ScanSettingBinding.

### Create ScanSettingBindings for enterprise-appropriate scans




### Confirm Execution of scans

### Optional but highly recommended: Install ACS 

Now is a really good time to address the timeless question: "What did it look like before?" 

There are a couple of ways to extract compliance data, but ACS is by far the easiest. If you'd like a pre-remediation snapshot of your complaince state to keep the data for comparison, skip ahead to the ACS section [Using ACS to View and Export Compliance](/Methodology.md#next-steps-using-acs-to-view-and-export-compliance) and return when you have your CSV(s) safely tucked away.

### Identify items which FAIL analysis

### Identify failing scans with remediation content

### Review and confirm remediations

### Optionally apply tailoring to check & remediation content

### Manually apply remediation

### Rescan environment to confirm sucessful remediation

### Lather, rinse, repeat until available remediations applied

### Automating applying remediation - pros, cons, and recommendations

### Manual Remeditions



## Next Steps: Using ACS to view and export compliance



