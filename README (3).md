# Misconfiguration Scan

I scanned deliberately vulnerable Terraform code with Checkov, fixed three security problems on one S3 bucket, and scanned again to prove the fixes worked.

No cloud account was used and nothing was deployed. Checkov only reads the code.

## Result

Scanned file: `terraform/aws/s3.tf` from TerraGoat.

| | Passed | Failed |
|---|---|---|
| Before my fixes | 25 | 31 |
| After my fixes | 32 | 28 |

Passed went up by 7 and failed went down by 3. Fixing one setting can also satisfy related checks, so passes can rise by more than the number of fixes.

## What I used

- [Checkov](https://www.checkov.io/) 3.3.29, an open-source scanner for infrastructure code
- [TerraGoat](https://github.com/bridgecrewio/terragoat) by Bridgecrew, a Terraform project that is vulnerable on purpose, for practice only. Never deploy it.
- Windows, VS Code and Python 3.13

## How I ran it

Install Checkov:

```
py -3.13 -m pip install checkov
```

Scan a file and save the result:

```
py -3.13 -m checkov.main -f terraform/aws/s3.tf --compact > before.txt 2>&1
```

Show only the summary line:

```
Select-String "Passed checks" before.txt
```

I made a copy of `s3.tf` in a folder called `my-fixes`, edited the copy, and scanned it the same way to get the "after" number.

## The three fixes (bucket `data`)

**1. Block public access.** The bucket was public. I added a public access block so it cannot be made public by accident.

```hcl
resource "aws_s3_bucket_public_access_block" "data" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

**2. Turn on encryption.** The bucket was not encrypted. I added default encryption with a KMS key, inside the bucket block.

```hcl
server_side_encryption_configuration {
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.logs_key.arn
    }
  }
}
```

**3. Turn on versioning.** Without versioning, a deleted or overwritten file is gone for good. With it, old versions are kept.

```hcl
versioning {
  enabled = true
}
```

The syntax above matches the AWS provider style TerraGoat uses. Newer provider versions split encryption and versioning into their own resources.

## What I left unfixed on purpose

Four findings remain on the `data` bucket: access logging, a lifecycle rule, event notifications and cross-region replication. This lab fixes one bucket, and the other buckets in the file still fail by design.

## What I learned

- I edited a copy so I always had the original to compare against.
- An empty file made the scan print nothing. Checking the file size showed the problem.
- One typo, `true1` instead of `true`, quietly turned off the versioning fix. Scanning again showed the check was still failing, and I found the typo from that.
- A scan result only means something if I compare it with a before result.

## Credit

TerraGoat belongs to its authors. This repo contains only my notes and the lines I added, not their code.
