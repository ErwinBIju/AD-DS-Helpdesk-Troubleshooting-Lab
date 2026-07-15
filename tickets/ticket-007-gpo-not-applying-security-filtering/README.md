# Ticket 007: Group Policy Not Applying Due to Security Filtering

## Issue Summary

A domain user reported that an expected user-based Group Policy setting was not applying on the Windows client.

Investigation showed that the `Student Lockdown GPO` was configured and linked correctly, but the affected user `LAB\erwin` was excluded by incorrect Security Filtering. The GPO was filtered to `Domain Computers`, so the user did not have permission to apply the user-side policy.

The Security Filtering was corrected by restoring `Authenticated Users`, then Group Policy was refreshed and verified successfully.

## Environment

| Item | Details |
|---|---|
| Domain | lab.local |
| NetBIOS | LAB |
| Domain Controller | DC01 |
| DC IP Address | 192.168.40.10 |
| Server OS | Windows Server 2022 |
| Client | DESKTOP-J57NE1D |
| Client OS | Windows 11 Enterprise |
| Affected User | LAB\erwin |
| GPO | Student Lockdown GPO |
| Policy Type | User Configuration |
| Policy Setting | Prohibit access to Control Panel and PC settings |
| Troubleshooting Layer | Group Policy Security Filtering |

## Symptoms

- User could sign in successfully.
- Client could communicate with the domain controller.
- The user was located in the correct OU.
- The GPO was linked to the correct OU.
- The GPO setting was enabled.
- The expected Control Panel restriction was not applying.
- `gpresult` showed that the GPO was denied because of Security Filtering.

## Troubleshooting Steps

### 1. Confirmed the GPO setting was enabled

I verified that the required setting was configured inside the `Student Lockdown GPO`.

Path checked:

`User Configuration → Policies → Administrative Templates → Control Panel → Prohibit access to Control Panel and PC settings`

![GPO setting enabled](../../Screenshots/ticket-007/01-gpo-setting-enabled.PNG)

The setting was enabled, which confirmed that the GPO contained the correct user-side restriction.

---

### 2. Confirmed the GPO was linked to the correct OU

I checked Group Policy Management and confirmed that the `Student Lockdown GPO` was linked to the OU containing `LAB\erwin`.

![GPO linked to user OU](../../Screenshots/ticket-007/02-gpo-linked-to-user-ou.PNG)

The GPO link was present and enabled.

---

### 3. Confirmed the user was in the correct OU

I verified the affected user’s Active Directory location with PowerShell.

![User Distinguished Name](../../Screenshots/ticket-007/03-user-distinguished-name.PNG)

The command confirmed that `LAB\erwin` was located inside the OU where the GPO was linked.

Command used:

`Get-ADUser -Identity erwin | Select-Object Name, DistinguishedName`

---

### 4. Confirmed the GPO worked before the fault

Before reproducing the issue, I refreshed Group Policy on the client and confirmed that the Control Panel restriction worked.

![GPO working baseline](../../Screenshots/ticket-007/04-gpo-working-baseline.PNG)

This proved that the GPO, OU link, and policy setting were valid before the Security Filtering problem was introduced.

---

### 5. Confirmed the original Security Filtering configuration

I checked the `Scope` tab of the `Student Lockdown GPO`.

![Security Filtering before fault](../../Screenshots/ticket-007/05-security-filtering-before-fault.PNG)

Before the issue, Security Filtering included:

`Authenticated Users`

This allowed the affected user to read and apply the GPO.

---

### 6. Verified incorrect Security Filtering

I inspected the GPO Security Filtering after the issue occurred.

![Incorrect Security Filtering](../../Screenshots/ticket-007/06-incorrect-security-filtering.PNG)

The GPO was filtered to:

`Domain Computers`

This was incorrect because the configured setting was under User Configuration. The affected user `LAB\erwin` was not included in the GPO’s Security Filtering, so the user did not have permission to apply the GPO.

---

### 7. Confirmed the user-facing policy failure

I refreshed Group Policy on the client and signed out and back in as `LAB\erwin`.

![Policy not applied symptom](../../Screenshots/ticket-007/07-policy-not-applied-symptom.PNG)

Control Panel was accessible, which confirmed that the expected restriction was not applying.

---

### 8. Verified the GPO denial with gpresult

I ran `gpresult` under the affected user account.

![gpresult Security Filtering denied](../../Screenshots/ticket-007/08-gpresult-security-filtering-denied.PNG)

Command used:

`gpresult /r /scope:user`

The output showed that the `Student Lockdown GPO` was not applied because it was filtered out.

This confirmed that the issue was not the GPO setting itself, but the GPO’s eligibility through Security Filtering.

---

### 9. Generated a detailed Group Policy HTML report

I generated a detailed HTML Group Policy report for the broken state.

![HTML report Security Filtering denied](../../Screenshots/ticket-007/09-html-report-security-filtering-denied.PNG)

Command used:

`gpresult /h C:\Temp\ticket007-gpresult-broken.html /f`

The report showed that the `Student Lockdown GPO` was denied because of Security Filtering.

---

### 10. Confirmed DNS and domain-controller communication

I verified that the client could communicate with the domain controller and resolve the domain controller name.

![Domain connectivity confirmed](../../Screenshots/ticket-007/10-domain-connectivity-confirmed.PNG)

Commands used:

`ipconfig /all`

`nslookup dc01.lab.local`

`nltest /dsgetdc:lab.local`

The client was using the domain controller as DNS, could resolve `dc01.lab.local`, and could locate a domain controller for `lab.local`.

This ruled out DNS and domain-controller communication as the cause.

## Root Cause

The `Student Lockdown GPO` was not applying because of incorrect Security Filtering.

The GPO was filtered to:

`Domain Computers`

However, the affected setting was located under User Configuration. User Configuration policies are processed using the signed-in user’s security context. Since `LAB\erwin` was not included in the GPO’s Security Filtering, the user did not have permission to apply the GPO.

The GPO was correctly configured and linked, but Windows denied the GPO during policy processing because the user was outside the allowed security scope.

## Fix

I corrected the GPO Security Filtering in Group Policy Management.

The incorrect entry was removed:

`Domain Computers`

The correct entry was restored:

`Authenticated Users`

![Security Filtering corrected](../../Screenshots/ticket-007/11-security-filtering-corrected.PNG)

This allowed authenticated users in the linked OU, including `LAB\erwin`, to read and apply the user-side GPO.

---

I also verified the GPO permissions from the Delegation tab.

![GPO Read and Apply permissions](../../Screenshots/ticket-007/12-gpo-read-and-apply-permissions.PNG)

The required permissions were confirmed:

`Read : Allow`

`Apply group policy : Allow`

## Verification

### 1. Verified the GPO was applied after the fix

After correcting Security Filtering, I refreshed Group Policy on the client.

Command used:

`gpupdate /force`

Then I signed out and signed back in as `LAB\erwin`.

I ran `gpresult` again.

![gpresult GPO applied](../../Screenshots/ticket-007/13-gpresult-gpo-applied.PNG)

Command used:

`gpresult /r /scope:user`

The `Student Lockdown GPO` appeared under Applied Group Policy Objects.

---

### 2. Verified the post-fix HTML Group Policy report

I generated a new HTML report after correcting Security Filtering.

![HTML report GPO applied](../../Screenshots/ticket-007/14-html-report-gpo-applied.PNG)

Command used:

`gpresult /h C:\Temp\ticket007-gpresult-fixed.html /f`

The report showed that the `Student Lockdown GPO` was now applied instead of denied.

---

### 3. Verified the user-facing policy restriction

I tested Control Panel access again while signed in as `LAB\erwin`.

![Control Panel restricted after fix](../../Screenshots/ticket-007/15-control-panel-restricted-after-fix.PNG)

Control Panel access was blocked by policy, confirming that the GPO was applying correctly and the user-facing issue was resolved.

## Explanation

Group Policy application depends on more than the GPO being linked to the correct OU.

For a GPO to apply, the target user or computer must be in scope through both:

- Active Directory location
- Security Filtering permissions

In this ticket, the user was in the correct OU and the GPO was linked correctly. The policy setting was also enabled. The failure happened because Security Filtering allowed only `Domain Computers`.

Since the setting was under User Configuration, Windows evaluated the signed-in user account, `LAB\erwin`. Because that user did not have permission to apply the GPO, Windows filtered the GPO out.

The fix was to restore `Authenticated Users` to Security Filtering and confirm that it had both `Read` and `Apply Group Policy` permissions.

After running `gpupdate /force`, signing out and back in, and checking `gpresult`, the GPO applied successfully and the Control Panel restriction worked again.

## Help Desk Notes

- Confirmed the GPO setting was enabled.
- Confirmed the GPO was linked to the correct OU.
- Confirmed the affected user was in the correct OU.
- Confirmed the GPO worked before the Security Filtering fault.
- Verified the original Security Filtering configuration.
- Identified incorrect Security Filtering set to `Domain Computers`.
- Confirmed the policy was not applying on the client.
- Verified the denial with `gpresult /r /scope:user`.
- Generated a detailed HTML report with `gpresult /h`.
- Ruled out DNS and domain-controller communication problems.
- Restored `Authenticated Users` to Security Filtering.
- Verified `Read` and `Apply Group Policy` permissions.
- Refreshed Group Policy with `gpupdate /force`.
- Confirmed the GPO appeared under Applied Group Policy Objects.
- Confirmed the Control Panel restriction worked after the fix.
- Root cause: GPO denied because the affected user was excluded by Security Filtering.
