| Key | Value |
|--|--|
| Date          | 	2020-03-05 |
| Application|	    Data Science |
| Owner of  Role | 	Osman Currim (Osman.Currim@rci.rogers.com)
| Role	| Digital-Operations
| Risk Level of Role | 	high
| Business Description	| Data Science Application is trying to read from CEX-Staging S3 buckets into DSC-WorkStation . For this the S3 buckets in CEX-Stage must allow through its bucket policies, the profiles to come in and read the contents in them.<br> - 737619476443 - Is the account ID of DSC-WS account <br> - rogers-rfm-infra-staging-internal-storage-1 is the s3 bucket name

6. Permissions in Role	S3 Bucket Policy 
	
	```
	{
	"Sid": "crossAccountAccessFromDSCWStoCEXstage",
	"Effect": "Allow",
	"Principal": {
	"AWS": [
	        "arn:aws:iam::737619476443:role/DataScienceGlueExecutionRole",
	        "arn:aws:iam::737619476443:role/DataScienceEC2Role"
	]
	},
	"Action": [
	        "s3:ListBucket",
	        "s3:GetObject"
	],
	"Resource": [
	        "arn:aws:s3:::rogers-rfm-infra-staging-internal-storage-1/*",
	        "arn:aws:s3:::rogers-rfm-infra-staging-internal-storage-1"
	]
	}
	```
7. Access to Customer Data?	Yes
8. Privileged Admin access?	No
9. Vital Application	Yes
10. Which environment? (Dev, QA, Staging, Prod)	DSC-WorkStation (Stage)
11. Which services applied to?	S3 bucket policy
	
12. Will users be Assigned to role?	No
13. Who are users assigned to?	NA



