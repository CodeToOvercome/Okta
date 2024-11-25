## Decoupling AD from Entra and Okta: A Practical Template

When it comes to decoupling AD from Entra and Okta, it's crucial to follow a well-structured phase-by-phase approach. This guide walks you through the major phases, sharing insights and practical steps based on my recent experience in a similar project.

### Phase 1: Decouple Okta Apps from AD Groups

The first phase involves decoupling Okta applications from AD groups by creating equivalent Okta groups and mirroring their membership. The easiest way to achieve this is to export the AD-sourced group and its members directly within Okta. Since you already have the Okta IDs of each member and their respective groups, this makes the transition smooth. If AD groups aren't sourced in Okta, you can export them from AD using PowerShell and import them into Okta using the Okta API.

This phase ensures that applications using Okta groups can operate independently from AD without any service disruption.

### Phase 2: Upgrade MS-365 Integration within Okta

Next, focus on upgrading your MS-365 integration within Okta, if it's not already enabled for provisioning. This involves creating Okta groups, assigning these groups to the Microsoft application, and [linking all relevant license SKUs](https://learn.microsoft.com/en-us/entra/identity/users/licensing-service-plan-reference) to each group. It's advisable to create a dedicated group for each license type for scalability rather than combining multiple licenses as a bundle. It is recommended to create a dedicated group for each active role intended for user assignments. 

An important setting during this phase is to modify the roles and license attributes under the Profile Editor. Change the setting from 'priority (default)' to 'combine values across groups' to handle licensing better.

**A critical note**: if a user is not part of an Okta group but is assigned to the Microsoft application, enabling provisioning can override existing licenses, potentially leaving them with no licenses and no roles  resulting in denied access to services like mailboxes and admin portals.

### Phase 3: Decouple Okta Accounts from AD

This phase is to decouple Okta accounts that are sourced from AD. This step involves unassigning each user from the AD app within Okta. If users are linked to AD for password management through delegated authentication, resetting each user's password becomes necessary after unassigning.

To make this process user-driven, we encouraged users to fill out a Microsoft Form, which triggered a webhook POST to Okta workflows. Once submitted, Okta automatically unassigns the AD link and sets a temporary password. We also sent a Slack message to each user containing their temporary password, allowing them to log in and reset it securely. We utilized Microsoft Power Automate to trigger the webhook when the form was submitted, automating the entire decoupling workflow.

To encourage and accelerate user adoption, we used N8N workflows to query Okta or a custom CSV file for user status and send regular emails and Slack messages prompting users to submit the form.

Once all AD groups and accounts are decoupled within Okta, you can deactivate the AD app and remove the Okta AD Sync & password agents from your AD server.

### Phase 4: Convert AD Objects into Entra Objects

&#x20;AD objects, such as service accounts, shared mailboxes, distribution, security, and mail-enabled security groups, also need to be converted. Objects, like service accounts and shared mailboxes, are easier to convert by simply moving them to a No Sync OU. The No Sync OU is created by default when AD Connect Sync is enabled, or you can create an OU that is excluded from synchronization. Once an object is moved, during the next sync, it is moved to the deleted users folder within Entra, where you can select and restore it as an Entra object.

For objects like groups, they need to be recreated. I recommend creating all groups with a prefix (e.g., 'stg') and staging them in advance. Next, move the old group to the No Sync OU, rename the newly created group to the original name, and reassign the old alias to prevent any downtime during the transition. If you used 'stg' as a prefix, consider removing any automatically added aliases during group creation.

### Phase 5: Convert AD User Accounts to Entra

This phase follows a similar approach to Phase 3, utilizing Microsoft Forms for a user-driven process. We employed N8N and Azure Runbook to run automated PowerShell scripts that moved user objects into the No Sync OU and triggered delta syncs on command. Delta syncs were scheduled every 10 minutes to avoid locking out the sync agent.

Next, we used Azure Graph API to fetch recently deleted users and restore them. Slack notifications were sent to users both before and after the conversion.  We recommend running the N8N instance in the cloud to avoid potential issues with on-prem server setups.

### Phase 6: Turn Off DirSync

Review the AD OU and ensure there are no active objects in sync with Entra. Go to portal.azure.com and filter objects sourced from AD to identify any objects that may have been missed during the conversion. Once you confirm that no objects are left behind, you can turn off directory synchronization by following the instructions in this Microsoft article: [Turn Off Directory Synchronization](https://learn.microsoft.com/en-us/microsoft-365/enterprise/turn-off-directory-synchronization?view=o365-worldwide\&source=recommendations).

If you are using Okta to create users in MS-365, Okta will automatically set the immutableId upon user creation. For existing users, you can create a Microsoft 365 or Okta attribute to save the immutableId if needed. In our case, we saved it, though in hindsight it may not have been necessary, as Entra retains the immutableId even after conversion.

