# businessplan
Angles Home

AWS service Used for this Deployment.

DNS - GoDaddy
AWS CloudFront - for CDN
S3 - where the index.html file is stored
ACM - for TLS certificate.
CodePipeline - Connect to Github repo and automation.


Angels Senior Care Website

Static brochure website for Angels Senior Care, a nursing home in Klang, Selangor. Live at https://angels.sebasu.space

The whole site is one file, index.html, with the CSS in a <style> block and the JavaScript in a <script> block. No frameworks, no backend, no build step.

How it fits together
Visitor -> angels.sebasu.space (CNAME at GoDaddy)
        -> CloudFront (HTTPS, ACM certificate, caching)
        -> private S3 bucket (index.html), read via origin access control (OAC)
S3 only stores the files. The bucket is private and Block Public Access stays on.
CloudFront serves the site over HTTPS and caches it at the edge.
ACM provides the free TLS certificate. It must live in us-east-1 for CloudFront.
GoDaddy only holds the domain and DNS records. It does not host anything.
Resource details

Fill in the blanks so you never have to hunt for them.

Item	Value
Domain	sebasu.space (registered at GoDaddy)
Site address	angels.sebasu.space
S3 bucket	angels-sebasu-site (region: ap-southeast-1)
CloudFront distribution ID	________________
CloudFront domain	d__________.cloudfront.net
ACM certificate ARN (us-east-1)	________________
Setup steps (what was done)

1. Create the bucket and upload the page

bash
aws s3 mb s3://angels-sebasu-site --region ap-southeast-1
aws s3 cp index.html s3://angels-sebasu-site/ --content-type text/html

2. Request the certificate (must be us-east-1)

bash
aws acm request-certificate --region us-east-1 \
  --domain-name angels.sebasu.space --validation-method DNS

3. Get the DNS validation record

bash
aws acm describe-certificate --region us-east-1 --certificate-arn <arn> \
  --query "Certificate.DomainValidationOptions[0].ResourceRecord"

In GoDaddy DNS, add a CNAME using the Name and Value returned. Drop the trailing sebasu.space from the name, since GoDaddy adds it. The certificate status changes to Issued after a few minutes.

4. Create the CloudFront distribution (console)

Origin: the S3 bucket (the ...s3.ap-southeast-1.amazonaws.com address, not the s3-website one)
Origin access: origin access control (OAC), and accept the bucket policy update
Default root object: index.html
Alternate domain name: angels.sebasu.space
Custom SSL certificate: the ACM certificate from step 2
Viewer protocol policy: redirect HTTP to HTTPS

5. Point the site address at CloudFront

In GoDaddy DNS, add a CNAME:

Name: angels
Value: the CloudFront domain (d__________.cloudfront.net)

After a few minutes the site loads at https://angels.sebasu.space.

Updating the site

Upload the changed file, then clear the CloudFront cache:

bash
aws s3 cp index.html s3://angels-sebasu-site/ --content-type text/html
aws cloudfront create-invalidation --distribution-id <distribution-id> --paths "/*"

Then hard refresh the browser (Ctrl+Shift+R, or Cmd+Shift+R on a Mac). The invalidation takes a minute or two.

To avoid invalidating after every edit, upload with a short cache time instead:

bash
aws s3 cp index.html s3://angels-sebasu-site/ --content-type text/html --cache-control "max-age=300"

If photos are added later, upload them too:

bash
aws s3 sync images/ s3://angels-sebasu-site/images/
Where to edit things in index.html
What	Where to look
Phone number	Footer text, the tel: links (Call Us Now and footer), and WHATSAPP_NUMBER in the script
Email	Footer mailto: link
Address	Footer contact list
Testimonial	The <blockquote> in the About section
Services and facilities	The cards in the services and facilities sections
Photos	The grey-green "Photo:" placeholder blocks, and the hero background in the CSS
Colors	CSS variables at the top of the <style> block (--sage, --cream, --forest)

Phone format notes: the tel: links use +60123456789 (no spaces). WHATSAPP_NUMBER uses 60123456789 (no plus sign, no spaces).

Enquiry form

The form has no server. It opens WhatsApp with the message ready to send to WHATSAPP_NUMBER. Nothing is stored. To get enquiries by email instead, add a form service such as Formspree.

Troubleshooting

"Access Denied" XML error in the browser

Default root object is not set to index.html in CloudFront.
The bucket policy for OAC is missing. Copy it from the distribution's origin settings.
index.html is not at the top level of the bucket (aws s3 ls s3://angels-sebasu-site/).
The origin is set to the s3-website endpoint instead of the normal bucket address.

Changes do not show up

Re-upload the file to S3, then run the invalidation above.
Hard refresh, or try a private browser window.

Certificate stuck on "Pending validation"

Check the validation CNAME in GoDaddy: the name should not include the domain twice.
Costs and renewals
AWS: S3, CloudFront and ACM together cost very little for a small site, usually well under a dollar a month. Check the billing console to confirm.
ACM certificates renew automatically, but only while the validation CNAME stays in GoDaddy DNS. Do not delete it.
Domain sebasu.space renews at GoDaddy. Renewal date: ____________.
