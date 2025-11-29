Title:
Hosting a Secure Static Website using GoDaddy Domain + AWS S3 + CloudFront + Route 53

Goal:
Deploy a secure, fast, and production-ready static website using an external domain (purchased from GoDaddy), AWS S3 for storage, CloudFront as CDN (with Origin Access Identity / Origin Access Control), and Route 53 for DNS management.
Keep the S3 bucket private and ensure users access content only through CloudFront (HTTPS).

Description:
This project documents how to take a domain bought from GoDaddy and map it to an AWS-hosted static website. The site files are stored in a private S3 bucket; CloudFront is the distribution layer that serves files securely over HTTPS using an Origin Access mechanism; Route 53 hosts DNS records and becomes authoritative for the domain after updating GoDaddy name servers.
This document covers the end-to-end steps, how the system works at runtime, common challenges and mitigations, and the expected result.

Entire process — Step by step
Preparation:
Purchase example.com (or your chosen domain) on GoDaddy and confirm you have access to the domain management panel.
Prepare your static website files locally (index.html, CSS, JS, images, assets).

AWS: S3 bucket:
In the AWS Console, open S3 → Create bucket.
Give the bucket the exact domain name (best practice): example.com.
Choose a region and keep Block all public access ON (we will serve via CloudFront).
Upload website files (index.html, assets). Optionally set index.html as default root object in CloudFront later.

AWS: CloudFront distribution:
Open CloudFront → Create distribution (Web).
Origin Domain Name: select your S3 bucket endpoint.
Set Restrict Bucket Access = Yes and Create Origin Access Identity (OAI) OR use Origin Access Control (OAC) (modern approach).
In CloudFront, enable HTTPS (Viewer Protocol Policy): Redirect HTTP to HTTPS.
Configure caching policy, allowed methods (GET, HEAD), and default root object (index.html).
Request or import an ACM certificate (in us-east-1 for CloudFront) for example.com and www.example.com.
Set the certificate on the CloudFront distribution and create it — note the CloudFront domain (xxxxxx.cloudfront.net). Wait until Deployed.

S3 bucket policy: allow only CloudFront:
Update the S3 bucket policy to allow GetObject for CloudFront OAI (or OAC principal) only. Example policy uses the CloudFront OAI ARN and arn:aws:s3:::example.com/* resource.
Confirm public access is still blocked and that only the CloudFront identity has s3:GetObject permission.

AWS: Route 53 Hosted Zone:
In Route 53 → Hosted zones → Create hosted zone for example.com (Public Hosted Zone).
Route 53 will create 4 NS records (AWS name servers). Copy these values.
Create an Alias A record for the root domain (example.com) pointing to the CloudFront distribution (CloudFront appears as an alias target). Create a CNAME or Alias record for www if desired.

GoDaddy: Update name servers:
In your GoDaddy domain panel → DNS → Nameservers → set custom nameservers.
Replace GoDaddy default nameservers with the 4 Route 53 name servers copied earlier.
Save — DNS delegation will propagate (usually within minutes to a few hours; sometimes up to 48 hours globally).

Final test & cache management:
After propagation, open https://example.com — the browser should resolve to CloudFront and load your site via HTTPS.
If you update files, either invalidate CloudFront cache (Create invalidation) or version assets (recommended: filename hashing).
Working (How the system runs at runtime)
User visits https://example.com.
Browser does a DNS lookup; the recursive resolver resolves the domain to the CloudFront distribution IP via Route 53 (authoritative name servers).
Browser connects to CloudFront (HTTPS). If CloudFront cache has the requested file, it returns it immediately.
If CloudFront cache miss, CloudFront requests the object from the S3 origin using the configured OAI/OAC credentials.
S3 validates the request originates from the authorized CloudFront identity and returns the object.
CloudFront caches the object and returns it to the user over HTTPS.

Security points:
S3 bucket is private and not directly reachable by public clients.
CloudFront handles TLS termination using ACM certificate.
Route 53 is authoritative for the domain after GoDaddy NS are pointed to AWS.

Conclusion
Using GoDaddy for domain registration and AWS for DNS + hosting gives a secure, scalable, and performant static website setup. The recommended configuration is to keep S3 private and serve content only through CloudFront using OAI/OAC. Route 53 provides native DNS integration to point the domain to CloudFront. This architecture supports HTTPS, global caching, and fine-grained security control.

Challenges typically faced and solutions:
DNS Propagation Delays
Problem: After switching GoDaddy nameservers to Route 53, propagation may take time.
Solution: Wait for propagation (monitor with dig or nslookup), reduce TTL only if you expect frequent changes, and use temporary host file entries for immediate testing.

SSL Certificate Location
Problem: CloudFront requires an ACM certificate in the us-east-1 region.
Solution: Request the certificate specifically in us-east-1 and validate via DNS (easiest) using Route 53.

S3 403/Access Denied Errors
Problem: Direct S3 URL returns 403 after making bucket private.
Solution: Ensure CloudFront has OAI/OAC and S3 bucket policy allows that identity s3:GetObject for arn:aws:s3:::example.com/*.

Stale Content due to CDN caching
Problem: Updates don't appear immediately.
Solution: Use cache-busting (versioned filenames) or issue CloudFront invalidations (cost/time trade-offs).

Mixing providers (Cloudflare vs CloudFront)
Problem: Using Cloudflare in front of CloudFront or mixing CDNs complicates TLS and origin configs.
Solution: Prefer a single CDN; if using Cloudflare, use proper origin configuration and consider bypassing CloudFront.

Wrong Name Server configuration at GoDaddy
Problem: Forgot to replace GoDaddy NS → domain still resolves to GoDaddy DNS.
Solution: Update nameservers correctly and confirm with dig NS example.com.

ACM validation failures
Problem: DNS validation failed when certificate requested.
Solution: Ensure the correct CNAME records are added to Route 53 hosted zone and that NS delegation is done.

How we solve the challenges (process + tools)
Use dig/nslookup/host to verify DNS and NS delegation. Example: dig NS example.com +short and dig A example.com.
Use Route 53’s DNS validation for ACM and automation (CloudFormation / Terraform) for repeatable setups.
Use CloudFront invalidation API or console for urgent cache clears, and prefer hashed filenames for releases.
Keep S3 public access blocked; use explicit bucket policy bound to CloudFront identity.



Example S3 Bucket Policy (with OAI)
{
"Version": "2012-10-17",
"Statement": [
{
"Sid": "AllowCloudFrontServicePrincipal",
"Effect": "Allow",
"Principal": {
"AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity EXXXXXXXXXXX"
},
"Action": "s3:GetObject",
"Resource": "arn:aws:s3:::your-bucket-name/*"
}
]
}

CloudFront Behavior Example (JSON snippet)
{
"Origins": [
{
"Id": "S3-your-bucket-name",
"DomainName": "your-bucket-name.s3.amazonaws.com",
"S3OriginConfig": {
"OriginAccessIdentity": "origin-access-identity/cloudfront/EXXXXXXXXXXX"
}
}
],
"DefaultCacheBehavior": {
"TargetOriginId": "S3-your-bucket-name",
"ViewerProtocolPolicy": "redirect-to-https",
"AllowedMethods": ["GET", "HEAD"],
"CachedMethods": ["GET", "HEAD"]
}
}

Route 53 Record Example

A Record → CloudFront Domain
Record Name: example.com
Type: A (Alias)
Value: d12345abcdef.cloudfront.net
Routing: Simple Routing
TTL: Auto
Monitor using browser DevTools (network panel), AWS CloudWatch for CloudFront metrics, and AWS S3 access logs if required.
