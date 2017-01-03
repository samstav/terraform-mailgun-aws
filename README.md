# tf_mailgun_aws  
[![Circle CI](https://circleci.com/gh/samstav/tf_mailgun_aws/tree/master.svg?style=shield)](https://circleci.com/gh/samstav/tf_mailgun_aws)

A Terraform module for creating a Mailgun domain, Route53 Zone, and corresponding DNS records

This project automates the following setup, on AWS Route 53:

https://documentation.mailgun.com/quickstart-sending.html#verify-your-domain

Sending & Tracking DNS Records created by this module:  

| Type | Value | Purpose |
| --- | --- | ---|
| TXT | “v=spf1 include:mailgun.org ~all” | SPF (Required) |
| TXT | [_This value is dynamic_](https://documentation.mailgun.com/quickstart-sending.html#add-sending-tracking-dns-records)| DKIM (Required) |
| CNAME | “mailgun.org” | Tracking (Optional) |

Receiving MX Records Records created by this module:  

| Type | Value | Purpose |
| --- | --- | ---|
| MX | mxa.mailgun.org | Receiving (Optional) |
| MX | mxb.mailgun.org	| Receiving (Optional) |

There is an [open issue](https://github.com/samstav/tf_mailgun_aws/issues/1) to make these receiving records optional for this module. 

## Prerequisites

### mailgun

You'll need your Mailgun API Key, found in your control panel homepage. 

Sign up: https://mailgun.com/signup  
Control Panel: https://mailgun.com/cp

### terraform

https://www.terraform.io/downloads.html

or mac users can `brew install terraform`

The included script can help you configure your [terraform remote state](https://www.terraform.io/docs/state/remote/).

```
$ ./main.py tf-remote-config big-foo.com --dry-run
Would run command:

terraform remote config -state="terraform.tfstate" -backend="S3" -backend-config="bucket=terraform-state-big-foo-dot-com" -backend-config="key=terraform.tfstate" -backend-config="region=us-east-1" -backend-config="encrypt=1"
```

Run the same, but without `--dry-run`, to configure terraform to use remote state. This will also create [your s3 bucket](https://www.terraform.io/docs/state/remote/s3.html) if it doesn't already exist.

Mailgun domains do not support `terraform import`, so you need to let this module
create the mailgun domain for you, otherwise you end up manually editing your
state file which probably won't end well.

## Usage

Utilize this module in one or more of your tf files:

### variables file, `terraform.tfvars`  

```bash
domain = "big-foo.com"
aws_region = "us-east-1"
mailgun_api_key = "key-***********"
mailgun_smtp_password = "*********"
```

### terraform file, e.g. `main.tf`  

```hcl
provider "mailgun" {
  api_key = "${var.mailgun_api_key}"
}

provider "aws" {
  region = "${var.aws_region}"
}

module "mailer" {
  source                = "github.com/samstav/tf_mailgun_aws"
  domain                = "${var.domain}"
  mailgun_smtp_password = "${var.mailgun_smtp_password}"
}
```

Then

```
$ terraform get -update=true
$ terraform plan -out=mailer.plan
$ terraform apply mailer.plan
```

__*Before running your plan, [fetch the module with `terraform get`](https://www.terraform.io/docs/commands/get.html)*__


### Using an existing route53 zone for your domain

To use an existing zone, instead of letting this tf module create the zone,
you need to import your zone (by id) *into the mailgun-aws tf module*:

```bash
$ terraform import module.INSTANCE.aws_route53_zone.this <your_route53_zone_id>
```

where INSTANCE is the name you choose as in

```hcl
module "INSTANCE" {
  source = "github.com/samstav/tf_mailgun_aws"
}
```

To find the zone id for your existing Route53 Hosted Zone:

```bash
$ aws route53 list-hosted-zones-by-name --dns-name big-foo.com
```

### Adding a route in mailgun to forward all mail

Route resources are not available in the [mailgun terraform provider](https://www.terraform.io/docs/providers/mailgun/), so we do it with the script. 

```
$ ./main.py create-route big-foo.com --forward bigfoo@gmail.com
{
  "message": "Route has been created",
  "route": {
    "actions": [
      "forward(\"bigfoo@gmail.com\")"
    ],
    "created_at": "Sun, 01 Jan 2017 18:21:16 GMT",
    "description": "Forward all mailgun domain email to my specified forwarding location(s).",
    "expression": "match_recipient(\".*@big-foo.com\")",
    "id": "84jfnnb3oepoj85jhbaae4f6",
    "priority": 1
  }
}
```

See `./main.py create-route --help` for more options on creating routes. 

### Nameservers

Make sure the nameservers (the values on your NS record in your zone) match the nameservers configured at your registrar.
