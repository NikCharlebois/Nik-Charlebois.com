---
title: Authenticating to Fabric via Microsoft365DSC
---

<h1 class="blog-title">Authenticating to Fabric via Microsoft365DSC</h1>
<div class="article-date">2024-12-05</div>
<p>We recently added support for Fabric settings in <a href="https://Microsoft365DSC.com">Microsoft365DSC</a> via the FabricAdminTenantSettings resource. This resources uses a the Fabric API (api.fabric.microsoft.com), which was never used as part of the solution before the introducton of this resource. <strong>Important</strong>: this API is currently read-only and only allows you to read settings, for snapshot/export of monitoring pruposes. It canot be used to change settings at the moment; this is a Fabric API restriction and not a "direct" limitation of M365DSC. There are some things that need to be configured on your tenant before the API will allow a service principal to authenticate properly. We will cover these steps as part of this article. For the purpose of this post, we will assume that you create <strong>a brand new</strong> application registration and want to use it to authenticate, using a certificate thumbprint. For additional details on how to create such an app registration, please refer to the <a href="https://nik-charlebois.com/blog/posts/2024/getting-started-m365dsc/index.html#step5">Getting Started with Microsoft365DSC</a> article. In my case, I created a new app registration named <strong>FabricTest</strong> and uploaded a certficate for it.</p>

<h2 id="securityGroup">Step 1 - Create a Security Group for the Service Principal<a href="#securityGroup" class="anchor">⚓</a></h2>
<p>The first thing we need to do is create a brand new security group in Entra ID and add our service principal as a member. This group will be used to specify what principals will have access to our Fabric APIs. In my case I created a new security group named <strong>Fabric API Access</strong> and added my <strong>FabricText</strong> service principal as a member of it.</p>
<img src="/blog/posts/2024/authenticating-to-fabric/images/securitygroup.png" alt="Creating a new security group in Entra Id" />

<h2 id="grantPermissions">Step 2 - Grant Entra ID Roles to your Service Principal<a href="#grantPermissions" class="anchor">⚓</a></h2>
<p>In order to be allowed to authenticate to the Fabric API, your service principal will need to be granted an appropriate Entra ID role. We recommend granting it the <strong>Fabric Administrator</strong> role, but it will also work with the <strong>Global Reader</strong> role.</p>

<h2 id="allowSPN">Step 3 - Allow Service Principals to Access Read-Only APIs<a href="#allowSPN" class="anchor">⚓</a></h2>
<p>Navigate to the Fabric admin portal Tenant settings page(<a href="https://app.fabric.microsoft.com/admin-portal/tenantSettings">https://app.fabric.microsoft.com/admin-portal/tenantSettings</a>). Once on the page, scroll down to the <strong>Admin API settings</strong> section, and expand the <strong>Service principals can access read-only admin APIs</strong> option. Make sure the toggle is set to <strong>Enabled</strong> and in the "Specific security groups" box, enter the name of your security group, in my case "Fabric API Access". Click <strong>Apply</strong></p>

<img src="/blog/posts/2024/authenticating-to-fabric/images/spnreadonlyaccess.png" alt="Configuring security groups that have access to the Fabric read-only APIs" />

<h2 id="conclusion">Conclusion<a href="#conclusion" class="anchor">⚓</a></h2>
<p>After performing the above steps, you will be able to execute a new Snapshot/Export of your tenant's configuration or perform monitoring for any configuration drift. There are currently no known plans to update the APIs to allow for the full CRUD set of operations (create & update).</p>


<script src="https://utteranc.es/client.js"
        repo="NikCharlebois/Nik-Charlebois.com"
        issue-term="pathname"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>