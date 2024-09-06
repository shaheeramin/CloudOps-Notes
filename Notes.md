# CloudOps Notes and Commands

## Alpha

Login to Alpha DB using 
```
mysql -h prod-omega-mysql.nyc3.internal.digitalocean.com -u svc_cloudops_p_ro -p
Password -> 1password (Alpha Prod)
```

## Paperspace

[Playbook](https://github.com/digitalocean/paperspace-infrastructure-docs/blob/912e805e7df778[…]f9fa5e7358991b8cfb486/playbooks/procedures/vm-find-residency.md) to find a VM in Xen 

Use the following to login to PS Prod DB 
```
psql -h ps-db-prod.cluster-c1xm2pfayrwq.us-east-1.rds.amazonaws.com -U adminps -p 16309 -d paperspace
Password -> prob-db (1Password)
```
For HV info `cat /etc/xensource/pool.conf` is used

## Cancelling Stalled Events

We have different types of events so consulting this [playbook](https://github.com/digitalocean/documentation/blob/master/oncall/playbooks/procedures/cancelling-events.md) is best before proceeding

Run the following to cancel a queued live migrate. Change the queued event type and Job ID for other events
```
event cancel --status=QUEUED --vm-id=354853776 --type=/vm/live_migrate 1935006380
```
Run the following to see event on a signle Droplet
```
hvdclient hv eventinfo -A -d $droplet_id
```
### Event Fail

Make an OPs ticket for event fail -> https://do-internal.atlassian.net/browse/OPS-37200

Run this command for event fail `event fail --email="samin@digitalocean.com" --jira OPS-37200 $event_ID`

### Stalled Events

Playboook for stalled events. Another day, another [playbook](https://github.com/digitalocean/documentation/blob/master/oncall/playbooks/alerts/stalled-backups.md)

## DDOS

Run these commands to check for UDP/TCP packaets

tcpdump -n -i bond0 -c 1000000 > tdump
cat tdump | grep -v ARP | awk '{print $3}' | sort | uniq -c | sort -nr | head -n5
cat tdump | grep -v ARP | awk '{print $5}' | sort | uniq -c | sort -nr | head -n5

tcpdump udp -qn -i bond0 -c 100000 > udump
cat udump | grep -v ARP | awk '{print $3}' | sort | uniq -c | sort -nr | head -n5
cat udump | grep -v ARP | awk '{print $5}' | sort | uniq -c | sort -nr | head -n5

## Resharding a Bucket

Consult the [playbook](https://github.com/digitalocean/documentation/blob/master/oncall/playbooks/procedures/reshard-spaces-bucket.md) and [glass](https://glass.internal.digitalocean.com/object/bucketresharding/candidates) page to see which buckets need resharding, check the right region as well.

Sometimes we need to stop a reashard and can look at this [playbook](https://github.com/digitalocean/documentation/blob/master/oncall/playbooks/procedures/reshard-spaces-bucket.md#stopping-a-reshard) for that. SSHing listed in here is important to understand.
