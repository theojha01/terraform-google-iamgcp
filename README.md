# Terraform Module for IAM and Suricata

This module sets up IAM roles for you and packet mirroring in a Google Cloud VPC and a collector instance behind an ILB, running Suricata IDS.


<!--- BEGIN_TF_DOCS --->
## Requirements

| Name | Version |
|------|---------|
| terraform | >= 0.14 |
| google | >= 3.0 |

```hcl
module "suricata" {
  source  = "aakasho/suricata/google"

  project = var.project
  network = google_compute_network.ids.id
  subnet  = google_compute_subnetwork.ids.id
  zone    = "us-central1-a"
  target_subnets = [
    google_compute_subnetwork.test.id
  ]

  custom_rules_path = var.custom_rules_path
}
```

## Testing
To test that packet mirroring and Suricata are working properly, we'll create a simple rules file that triggers an alert on an innocuous event such as a DNS query. These alerts were taken from the [Qwiklab](https://www.qwiklabs.com/focuses/14864?parent=catalog) on this topic. 

### 1. Prerequisites

First create a bucket, it doesn't matter the name, using `gsutil mb $BUCKET`.

The upload the file [example/my.rules](./example/my.rules) into the bucket using:

```bash
gsutil cp example/my.rules gs://$BUCKET
```

### 2. Terraform
Next, create a file called `terraform.tfvars` in the [`example`](./example) directory. And copy this, replacing the placeholder values:
```
project = "MY_PROJECT_NAME"
custom_rules_path = "gs://MY_BUCKET_NAME/my.rules"
```

Now run `terraform apply`.

### 3. Testing Suricata

To verify packet mirroring and Suricata is working properly, let's open one terminal and SSH into the `test` instance we created. The command should look similar to this, assuming your project has been set by `gcloud config set project ...`:

```
gcloud compute ssh test --zone us-central1-a --tunnel-through-iap
```

Now in a new terminal, let's SSH into our Suricata collector instance.

```
gcloud compute instances list | grep suricata
# This should output the name of the instance
gcloud compute ssh SURICATA_INSTANCE_NAME --zone us-central1-a --tunnel-through-iap
```

Once in your Suricata instance, you first should tail the fast.log. We'll see an alert here in a moment.

```bash
# Suricata Instance
tail -f /var/log/suricata/fast.log
```

Now back in your test instance, let's make a DNS request:

```bash
# Test instance
sudo apt install dnsutils
dig @8.8.8.8 example.com
```

In your Suricata terminal (in your fast.log) you should see the alert show up immediately, something like this:
```
03/22/2021-21:05:17.558245  [**] [1:99996:1] BAD UDP DNS REQUEST [**] [Classification: (null)] [Priority: 3] {UDP} 172.21.1.3:55787 -> 8.8.8.8:53
```

Now that you verified that packet mirroring and Suricata are working correctly, you can verify the connection to Cloud Logging is setup by going to the Cloud Logging Logs viewer in the GCP console, and running the following query:

```
logName:"logs/suricata.fast"
```

You should have at least one entry. Now you have your Suricata alerts in Cloud Logging so you can eventually make alerts using Cloud Monitoring and get pestered by Pagerduty or whatever, when Suricata thinks you're being attacked!

## Providers

| Name | Version |
|------|---------|
| google | n/a |

## Providers

| Name | Version |
|------|---------|
| google | >= 3.0 |
