# Ticket 001: Mapped Drive Missing

## Issue Summary

A domain user reports that the expected mapped network drive does not appear after logging into the Windows client.

## Environment

| Item | Details |
|---|---|
| Domain | lab.local |
| Domain Controller | DC01 |
| Client | DESKTOP-J57NE1D |
| User | LAB\erwin |
| GPO involved | Finance Mapped Drive Policy |
| Shared folder | \\\DC01\FinanceShare |
| Security group | Finance-Users |

## Symptoms

- The mapped drive does not appear in File Explorer after login.
- The user expects access to the Finance shared folder.
- The issue may involve Group Policy, group membership, or permissions.
- Direct UNC access to `\\DC01\FinanceShare` worked successfully, which confirmed that the share was reachable and the user had permission.
- The mapped drive was still missing, suggesting the issue was related to Group Policy drive mapping rather than file share access.

## Initial Hypothesis

The mapped drive may be missing because one or more requirements are not met:

- The mapped drive GPO is not linked to the correct OU.
- The user is not in the OU where the GPO applies.
- Security filtering prevents the GPO from applying.
- The user is not in the correct security group.
- The shared folder permissions or NTFS permissions are incorrect.
- The user has not logged off and back on after a group membership change.

## Troubleshooting Steps

1. Confirmed that the user was able to log in to the domain client.
2. Checked File Explorer and verified that the expected Finance mapped drive was missing.
3. Tested direct access to the shared folder using `dir \\DC01\FinanceShare`. The command listed the folder contents successfully, confirming that the share and permissions were working.
4. Ran `gpresult /r` to check which Group Policy Objects applied to the user.\
5. Confirmed that the mapped drive GPO was not listed under applied user policies.
6. Opened Group Policy Management on DC01.
7. Found that the mapped drive GPO was linked to the wrong OU.
8. Linked the GPO to the OU containing the affected user.
9. Ran `gpupdate /force` on the client.
10. Logged the user off and back on.
11. Verified that the mapped drive appeared successfully.

## Commands Used

gpupdate /force
gpresult /r
net use
dir \\DC01\Finance-Share

## Root Cause
The mapped drive GPO was linked to the wrong OU. Because the affected user was not in the OU where the GPO was linked, the user-side drive mapping policy did not apply.

## Fix
Linked the mapped drive GPO to the correct OU containing the affected user, then refreshed Group Policy and logged the user back in.

## Verification
After the GPO was linked correctly, gpresult /r showed the mapped drive GPO under applied user policies. The Finance mapped drive appeared in File Explorer, and net use confirmed the network drive connection.

## Explanation
If a user reports that a mapped drive is missing, I would first check whether the drive mapping GPO applied using gpresult /r. Then I would verify that the GPO is linked to the correct OU, confirm the user is in that OU, check security filtering, and test direct access to the UNC path. If the GPO does not appear in gpresult, the issue is likely Group Policy scope, link, filtering, or OU placement. If the GPO applies but the drive still fails, I would check permissions and network access to the share.
