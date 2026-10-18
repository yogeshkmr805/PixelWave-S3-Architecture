# PixelWave Studios S3 Architecture

## Project Description

A mid-sized digital media company named PixelWave Studios manages tens of thousands of graphics, marketing assets, campaign files, and HTML-based microsites for clients. Their existing storage structure is messy, lacks proper access control, and makes it difficult to track which version of an asset is the latest. Some files get accidentally deleted, old files pile up causing high storage bills, and external partners sometimes need access to specific assets.

This project delivers a scalable, secure, and cost-efficient S3 architecture for PixelWave, implementing:

<!-- Round button options -->
<p>
	<button style="border-radius:9999px; padding:8px 14px; margin:4px; border:1px solid #ccc; background:#f6f8fa;">🔁 Versioning</button>
	<button style="border-radius:9999px; padding:8px 14px; margin:4px; border:1px solid #ccc; background:#f6f8fa;">🗂️ Lifecycle management</button>
	<button style="border-radius:9999px; padding:8px 14px; margin:4px; border:1px solid #ccc; background:#f6f8fa;">👥 IAM roles & users</button>
	<button style="border-radius:9999px; padding:8px 14px; margin:4px; border:1px solid #ccc; background:#f6f8fa;">🔒 Bucket policies</button>
	<button style="border-radius:9999px; padding:8px 14px; margin:4px; border:1px solid #ccc; background:#f6f8fa;">🛡️ ACLs (limited use case demonstration)</button>
	<button style="border-radius:9999px; padding:8px 14px; margin:4px; border:1px solid #ccc; background:#f6f8fa;">🔐 Server-side encryption (SSE-S3 & SSE-KMS)</button>
	<button style="border-radius:9999px; padding:8px 14px; margin:4px; border:1px solid #ccc; background:#f6f8fa;">🌐 Static website hosting</button>
</p>

## Pre-requisites
- AWS Account with S3, IAM, and KMS permissions.
- Basic knowledge of AWS Console and S3 concepts.
- A web browser to access the AWS Management Console.

## Project File Structure
```
.
├── README.md
├── Assets/
│   ├── pixelwave-assets/
│   │   ├── logo.png
│   │   ├── logo_v2.png
│   │   ├── pixewave-banner.png
│   │   └── pixewave-banner_v1.png
│   └── pixelwave-web/
│       └── index.html
└── Policies/
    ├── bucket_policy.json
    ├── designer_policy.json
    └── automation_policy.json

```

## Challenges
The company has these challenges:

**Challenge 1:** No versioning system. Designers often overwrite files, losing previous iterations.

**Challenge 2:** Storage cost is increasing because assets are stored indefinitely without automated cleanup.

**Challenge 3:** Need secure and controlled access for internal teams, contractors, and automated scripts.

**Challenge 4:** Need all data encrypted at rest due to client confidentiality requirements.

**Challenge 5:** Need to host static project previews for clients.

## Problem Solving Approach

We will follow a structured approach to build the solution:

**Step 1:** Create an S3 bucket structure for PixelWave. One bucket for assets, one for static sites.

**Step 2:** Enable versioning on the assets bucket to track all file iterations.

**Step 3:** Create lifecycle policies to automatically transition older versions to cheaper storage or delete them after a defined period.

**Step 4:** Create IAM groups, users, and roles that follow the principle of least privilege.

**Step 5:** Apply bucket policies restricting access only to approved teams and roles.

**Step 6:** Use ACLs sparingly to showcase object-level access control for specific partner use cases.

**Step 7:** Enable server-side encryption using S3-managed keys (SSE-S3) and AWS KMS (SSE-KMS).

**Step 8:** Configure static hosting on the site bucket and upload sample HTML files.

**Step 9:** Test various access scenarios to ensure everything works as expected.

## Step-by-Step Implementation
### Step 1: Create S3 Buckets
- Create two S3 buckets:
  - `pixelwave-assets` for storing images, campaign files, and design assets.
  - `pixelwave-web` for hosting static microsites.
![alt text](Buckets-1.png)

### Step 2: Enable Versioning
- Enable versioning on the `pixelwave-assets` bucket to keep track of all file versions. This allows designers to recover previous iterations of their work if needed.
![alt text](Bucket_versioning_disabled-1.png)
![alt text](Bucket_versioning_enabled-1.png)

Validating that versioning is enabled by uploading 2 different asset png files, modifying them by replacing them with new versions, and checking that multiple versions exist in the bucket.
![alt text](object_versioning_v2.png)


### Step 3: Implement Lifecycle Policies
Open the `pixelwave-assets` bucket.
Navigate to the Management tab.
Under Lifecycle rules, click Create lifecycle rule.
Use the following configuration template:

Rule name: `versioned-object-cost-optimization`

Scope: Apply to the entire bucket.

Lifecycle actions:
- Transition previous versions to the Standard-IA storage class after 30 days.
- Transition previous versions to Glacier Flexible Retrieval after 180 days.
- Expire previous versions after 365 days.

![alt text](Lifecycle_rule_setup_v2.png)

Why above Choices?
- 30 days: PixelWave designers typically revise files heavily only during the first month.
- 180 days: Older versions rarely get accessed but must be kept for audit.
- 365 days: Compliance requirements allow removing outdated versions after 1 year.

Testing Delete Marker Behavior
- Select object `logo.png` → Delete.
- S3 will NOT delete the versions; it will add a delete marker.
- Enable `Show versions` view.
![alt text](delete_marker-1.png)

### Step 4: Create IAM Groups and Users
- Create IAM groups for different teams:
  - `pixelwave-designers`: Designers can upload new assets, download existing assets, and list `pixelwave-assets` bucket contents. They cannot delete or modify lifecycle rules, versioning, or bucket settings.
  - `pixelwave-automation` group has a policy that allows uploading to the bucket but denies read and delete actions.
- Create IAM users and assign them to the appropriate groups. For example, `designer-alex` is added to the `pixelwave-designers` group, while `automation-uploader` is added to the `pixelwave-automation` group.

  ![alt text](IAM_user_groups.png)

  ![alt text](IAM_users.png)

- Create IAM policies for each group to enforce the principle of least privilege. 
![alt text](IAM_policies.png)

    - `pixelwave-designers` group policy allows `s3:PutObject`, `s3:GetObject`, and `s3:ListBucket` actions on the `pixelwave-assets` bucket, while denying 
    `s3:DeleteObject`.
    ![alt text](IAM_policies_designer.png)

    - `pixelwave-automation` group policy allows `s3:PutObject` but denies `s3:GetObject` and `s3:DeleteObject`.
    ![alt text](IAM_policies_automation.png)


### Step 5: Apply Bucket Policies
- Apply a bucket policy to the `pixelwave-assets` bucket that allows access only to the IAM groups created in Step 4. This ensures that only authorized users can interact with the bucket. Even if an object becomes public accidentally, this bucket policy prevents exposure.

![alt text](Bucket_policy.png)

### Step 6: Use ACLs Sparingly
- Demonstrate the use of ACLs for a specific partner use case. For example, PixelWave wants to share a single asset file with an external partner (for example a marketing agency) without giving access to the entire bucket.

- This does not make the bucket public—it only grants access to one specific AWS user external to your account.

- Go to permissions tab of the object, click on Access Control List (ACL), and add the external partner's AWS account ID with read permissions.

![alt text](S3_object_ACL.png)

### Testing Phase: 
#### Test Case 1: Designer Permission
- Log in as `designer-alex` (using IAM user console URL):

    - Try to upload a new asset to the `pixelwave-assets` bucket. ✅ Success
    ![alt text](designer_user_write_access.png)

    - Try to delete an existing asset. ❌ Access Denied
    ![alt text](designer_user_delete_access.png)

    - Try to list the contents of the bucket. ✅ Success
    ![alt text](designer_user_list_objects.png)

#### Test Case 2: Automation User Permission
- Log in as `automation-uploader` (using IAM user console URL):

    - Try to upload a new asset to the `pixelwave-assets` bucket. ✅ Success
    ![alt text](automation-uploader_user_write_access.png)

    - Try to download an existing asset. ❌ Access Denied
    ![alt text](automation-uploader_user_download_access.png)

#### Test Case 3: Bucket Policy Enforcement
- Copy the S3 Bucket URL of the `pixelwave-assets` bucket and try to access it from a browser without logging in. ❌ Access Denied
![alt text](Bucket_public_access.png)

### Step 7: Enable Server-Side Encryption
- Created SSE-S3 encryption for the `pixelwave-web` bucket to ensure all data is encrypted at rest. This is crucial for maintaining client confidentiality and meeting compliance requirements.
![alt text](Encryption_key_SSE_S3.png)

- Created SSE-KMS encryption for the `pixelwave-assets` bucket to provide an additional layer of security and control over encryption keys. This allows PixelWave to manage access to the keys and audit their usage.

    
    **AWS Console → search KMS → Customer managed keys →  click create key → select symmetric → give alias name and description → click next → add tags if needed → click next → add key administrators and users → click next → review and create key.**
    
![alt text](Encryption_key_SSE_KMS.png)

![alt text](Encryption_key_SSE_KMS_created.png)

- Enabled SSE-KMS Default Encryption on `pixelwave-assets` bucket to ensure that all new objects uploaded to the bucket are automatically encrypted using the specified KMS key.
![alt text](Enabled_Encryption_key_SSE_KMS.png)

- Enforcing encryption on the bucket ensures that all data at rest is protected, and only authorized users with the appropriate permissions can access the encrypted data.

Added Below statement to the bucket policy to enforce encryption on all objects uploaded to the `pixelwave-assets` bucket:

**S3 → pixelwave-assets → Permissions → Bucket policy**

Paste this policy (replace YOUR_ACCOUNT_ID):

```json
{
    "Sid": "DenyUnencryptedUploads",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::pixelwave-assets",
    "Condition": {
        "StringNotEquals": {
            "s3:x-amz-server-side-encryption": "aws:kms"
        }
    }
},
{
    "Sid": "DenyWrongKMSKey",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::pixelwave-assets",
    "Condition": {
        "StringNotEquals": {
            "s3:x-amz-server-side-encryption-aws-kms-key-id": "arn:aws:kms:YOUR_REGION:YOUR_ACCOUNT_ID:key/YOUR_KEY_ID"
        }
    }
}
```
### Testing Phase:
#### Test Case 4: Encryption Enforcement
- Try to upload an object to the `pixelwave-assets` bucket specifying server-side encryption. ✅ Success
![alt text](SSE_KMS_Upload_Test_case_4.png)

#### Test Case 5: Upload Without Encryption
- Try to upload an object to the `pixelwave-assets` bucket without specifying server-side encryption. ❌ Access Denied
![alt text](Without_Encryption_Upload_Test_case_5.png)

#### Test Case 6: Upload With different User KMS Key
- Uploaded an object to the `pixelwave-assets` bucket using `designer-alex` user by specifying KMS key ARN. ✅ Success
![alt text](designer_user_upload_object_test_6.png)

- Uploaded an object to the `pixelwave-assets` bucket using `automation-uploader` user by specifying same KMS key ARN. ✅ Success
![alt text](automation-uploader_user_uploaded_object_test_6.png)

- Uploaded an object to the `pixelwave-assets` bucket using `automation-uploader` user by not specifying Encryption Key. ❌ Access Denied
![alt text](automation-uploader_user_uploaded_object_without_specifying_encryp_test_6.png)

### Step 8: Configure Static Website Hosting
- Configure the `pixelwave-web` bucket for static website hosting. This allows PixelWave to host static previews of their projects for clients. The bucket is set up to serve an index.html file as the default page.

- Why PixelWave use S3 for static hosting:
    - Zero maintenance
    - Highly available
    - Extremely low-cost
    - Integrates with CloudFront for CDN delivery
    - Great for marketing sites, prototypes, documentation, or landing pages

#### Steps to configure static website hosting:
1. Go to the `pixelwave-web` bucket in the S3 console.
2. Click on the "Properties" tab.
3. Scroll down to the "Static website hosting" section and click "Edit."
4. Select "Enable" for static website hosting.
5. Specify the index document (e.g., `index.html`) and error document (optional) (e.g., `error.html`).
6. Click "Save changes."
7. Upload your static website files (HTML, CSS, JS, images) index.html to the `pixelwave-web` bucket.

Uploaded `index.html` file to the `pixelwave-web` bucket:

![alt text](index_static_site_uploaded.png)

8. Block public access for `pixelwave-web` bucket to be turned off. This is important to ensure that the bucket is not publicly accessible by default.
![alt text](index_static_site_uploaded-1.png)

9. Go to: pixelwave-web → Permissions → Bucket policy. 
Paste this policy (replace BUCKET_NAME):
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "PublicReadForStaticHosting",
                "Effect": "Allow",
                "Principal": "",
                "Action": [
                    "s3:GetObject"
                ],
                "Resource": "arn:aws:s3:::pixelwave-web/"
            }
        ]
    }
    ```
    This allows public read access only for this specific bucket and only for objects, not writes or listing. 

#### Testing Phase:
#### Test Case 7: Accessing the Static Website
Open a web browser and navigate to the static website endpoint provided by S3 for the `pixelwave-web` bucket. ✅ Success
![alt text](static_site_s3_browser_hosted.png)