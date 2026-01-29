# Adding FortiCNAPP Users

To grant other users access to FortiCNAPP, configure permissions in FortiCloud IAM.

Docs: <a href="https://docs.fortinet.com/document/forticnapp/latest/administration-guide/720640/iam-users" target="_blank">IAM Users</a>

## Steps

1. In <a href="https://forticloud.com" target="_blank">FortiCloud</a>, navigate to Services > IAM.

2. Create a Permission Profile (skip if one already exists):
   - Navigate to Permission Profiles > Add New
   - Enter a Permission Profile Name (eg FortiCNAPP)
   - Click Add Portal, select Lacework FortiCNAPP
   - Toggle on Access
   - Save the profile

3. Add the user:
   - Navigate to Users > Add New > IAM User
   - Enter user details (name, email, etc.)
   - Click Next
   - Choose user type (eg Organisation or Local)
   - Select the Permission Profile created in step 2
   - Click Next
   - Review and Confirm

4. Log in to FortiCNAPP:
   - In <a href="https://forticloud.com" target="_blank">FortiCloud</a>, navigate to Services > Show More > Lacework FortiCNAPP

5. Assign FortiCNAPP user group:
   - Navigate to Settings > Users
   - Find the user, click ... > Edit
   - Select user group: Admin
   - Save
