# Compliance Methodology

## General Principles

A well-implemented enterprise compliance policy begins with determining the baseline standard which most closely aligns with the security requirements of your organization, whether it be a US DOD STIG, a US Civilian NIST control set, other Non-US standards, or even commercial guidance available from various published sources. In addition to the generally published policies, your organization may have additional rules or more restrictive rules than the published document. A healthy enterprise compliance policy is the sum of the constituent parts created by comparing and contrasting all available sources of policy, then [usually] selecting the more restrictive setting of any conflicting controls.

Changes to the baseline control set in order to adapt to enterprise policies are done with what is called tailoring. By tailoring compliance content you can adapt and measure organizational compliance to the aggregate organizational standard and in many cases apply changes via automated means. By having a published list of tailored items unique to your organization you provide a documented and auditable asset of approved enterprise policy.

Examples might be items such as changing a password history from a DOD mandated 5 to a higher setting such as 24. In this example, if you merely set it to 24, compliance scans will show as failed becuase the number is not 5; further automated redmiation would set it to 5 in violation of an organizational policy that says 24. By tailoring that value to match 24, if it is not 24 it will be remediated to 24, and the tailoring itself is evidence that can be tied to an overriding (and published) organizational policy.

In general, most organizations have at least a few changes to "off-the-shelf" policies. The more of them you catch and tailor into your compliance, the better off you'll be in the long run.

## Specific Workflow for Openshift


