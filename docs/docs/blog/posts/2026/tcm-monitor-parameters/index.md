---
title: Using Parameters When Creating Tenant Configuration Management Monitors
---

<h1 class="blog-title">Using Parameters When Creating Tenant Configuration Management Monitors</h1>
<div class="article-date">2026-07-10</div>

<p>When you create a new monitor in Tenant Configuration Management, or TCM, you provide a baseline that describes the desired configuration state of your tenant. The simplest examples usually hard-code every value directly in the baseline. That works for a quick test, but it does not scale well when the same baseline needs to be reused across multiple tenants, environments, or naming conventions.</p>

<p>This is where <strong>parameters</strong> become useful. Parameters let you keep tenant-specific values outside of the reusable baseline definition. Instead of creating one baseline per tenant just because the tenant domain name, tenant identifier, or naming prefix changes, you define the variable value once and reference it from the resources in the baseline.</p>

<h2>The two places where parameters appear</h2>

<p>A parameterized monitor payload has two related sections:</p>

<ul>
<li><strong>baseline.parameters:</strong> declares the parameters the baseline expects.</li>
<li><strong>parameters:</strong> provides the concrete values to use when the monitor is created.</li>
</ul>

<p>The declaration belongs inside the baseline because it describes the shape of the reusable template. The values belong at the monitor level because they describe the tenant-specific input for this monitor instance.</p>

```json
{
  "displayName": "Monitor for Contoso",
  "description": "Monitor created from a parameterized baseline",
  "baseline": {
    "displayName": "Reusable Exchange baseline",
    "description": "Baseline that uses tenant-specific parameters",
    "parameters": [
      {
        "displayName": "FQDN",
        "description": "The fully qualified domain name of the tenant",
        "parameterType": "String"
      }
    ],
    "resources": []
  },
  "parameters": {
    "FQDN": "contoso.onmicrosoft.com"
  }
}
```

<p>By itself, the example above does not yet use the parameter. It only declares and supplies it. The important part is how the resource properties reference that value.</p>

<h2>Referencing a parameter in a resource property</h2>

<p>Inside a resource property, a parameter can be referenced with the <code>parameters()</code> expression. For example, if a property should be set to the tenant's fully qualified domain name, you can write:</p>

```json
{
  "Identity": "[parameters('FQDN')]"
}
```

<p>That expression tells TCM to resolve the value of <code>FQDN</code> from the monitor-level <code>parameters</code> object when the monitor evaluates the baseline.</p>

<h2>Using concat to build tenant-specific values</h2>

<p>In many cases, the parameter is only part of the desired value. For example, an email address might always start with the same local part, while the domain changes per tenant. For that scenario, use the <code>concat</code> function to combine literal strings and parameter values.</p>

```json
{
  "PrimarySmtpAddress": "[concat('testSharedMailbox@', parameters('FQDN'))]"
}
```

<p>If the monitor-level parameter value is <code>contoso.onmicrosoft.com</code>, the resolved value becomes:</p>

```text
testSharedMailbox@contoso.onmicrosoft.com
```

<p>You can use the same pattern anywhere a baseline property needs a value that is partially static and partially tenant-specific.</p>

<h2>Complete monitor creation example</h2>

<p>The following example creates a monitor with a parameterized Exchange shared mailbox resource. The baseline declares the <code>FQDN</code> parameter, uses it to build the shared mailbox primary SMTP address and an additional email address, and then supplies the tenant-specific value when creating the monitor.</p>

```http
POST https://graph.microsoft.com/v1.0/admin/configurationManagement/configurationMonitors/
```

```json
{
  "displayName": "Monitor for Exchange shared mailbox",
  "description": "Monitor created from a reusable parameterized baseline",
  "baseline": {
    "displayName": "Exchange shared mailbox baseline",
    "description": "Baseline with a tenant-specific domain parameter",
    "parameters": [
      {
        "displayName": "FQDN",
        "description": "The fully qualified domain name of the tenant",
        "parameterType": "String"
      }
    ],
    "resources": [
      {
        "displayName": "Shared mailbox resource",
        "resourceType": "microsoft.exchange.sharedmailbox",
        "properties": {
          "DisplayName": "TestSharedMailbox",
          "Alias": "testSharedMailbox",
          "Identity": "TestSharedMailbox",
          "Ensure": "Present",
          "PrimarySmtpAddress": "[concat('testSharedMailbox@', parameters('FQDN'))]",
          "EmailAddresses": [
            "[concat('smtp:testSharedMailbox@', parameters('FQDN'))]"
          ]
        }
      }
    ]
  },
  "parameters": {
    "FQDN": "contoso.onmicrosoft.com"
  }
}
```

<p>The reusable part of this payload is the baseline. To create a similar monitor for another tenant, keep the same baseline and change the value in the monitor-level <code>parameters</code> object.</p>

```json
{
  "parameters": {
    "FQDN": "fabrikam.onmicrosoft.com"
  }
}
```

<p>With that value, the <code>PrimarySmtpAddress</code> expression resolves to <code>testSharedMailbox@fabrikam.onmicrosoft.com</code>.</p>

<h2>Using more than one parameter</h2>

<p>A baseline can declare multiple parameters. This is useful when a value needs to combine more than one tenant-specific input, such as a naming prefix and a tenant domain.</p>

```json
{
  "baseline": {
    "parameters": [
      {
        "displayName": "MailboxPrefix",
        "description": "The prefix to use for generated mailbox names",
        "parameterType": "String"
      },
      {
        "displayName": "FQDN",
        "description": "The fully qualified domain name of the tenant",
        "parameterType": "String"
      }
    ],
    "resources": [
      {
        "displayName": "Shared mailbox resource",
        "resourceType": "microsoft.exchange.sharedmailbox",
        "properties": {
          "Alias": "[concat(parameters('MailboxPrefix'), 'SharedMailbox')]",
          "PrimarySmtpAddress": "[concat(parameters('MailboxPrefix'), 'SharedMailbox@', parameters('FQDN'))]"
        }
      }
    ]
  },
  "parameters": {
    "MailboxPrefix": "finance",
    "FQDN": "contoso.onmicrosoft.com"
  }
}
```

<p>In this example, the primary SMTP address resolves to <code>financeSharedMailbox@contoso.onmicrosoft.com</code>.</p>

<h2>Common mistakes to avoid</h2>

<ul>
<li><strong>Declaring parameters that are never used:</strong> keep the <code>baseline.parameters</code> list limited to values that are actually referenced by the baseline.</li>
<li><strong>Using a parameter without providing a value:</strong> every referenced parameter needs a corresponding entry in the monitor-level <code>parameters</code> object.</li>
<li><strong>Hard-coding tenant-specific values in reusable baselines:</strong> if the value changes between tenants, make it a parameter.</li>
<li><strong>Forgetting that parameters are strings:</strong> when using <code>concat</code>, make sure the resulting value matches the format expected by the target resource property.</li>
</ul>

<h2>When to parameterize a TCM monitor</h2>

<p>Parameterization is most valuable when you expect the same baseline to be used more than once. Good candidates include tenant domains, tenant identifiers, environment names, prefixes, notification addresses, and any resource property that follows a predictable naming pattern. If a value is unique to one tenant but the structure of the baseline is reusable, it is usually a good parameter candidate.</p>

<p>The main design goal is to separate <strong>what</strong> you want to monitor from <strong>which tenant-specific values</strong> should be used for that monitor instance. That separation makes baselines easier to review, easier to version, and easier to reuse across environments.</p>

<h2>Conclusion</h2>

<p>Parameters make Tenant Configuration Management monitors much more reusable. Define the expected inputs in <code>baseline.parameters</code>, provide the actual values in the monitor-level <code>parameters</code> object, and use expressions such as <code>parameters()</code> and <code>concat()</code> inside resource properties to build the final values TCM should evaluate.</p>

<p>If you are creating monitors manually, parameters may feel like an extra step. If you are building a repeatable monitoring process across tenants, they quickly become one of the most important parts of the baseline design.</p>

<script src="https://utteranc.es/client.js"
        repo="NikCharlebois/Nik-Charlebois.com"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
