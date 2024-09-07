# CloudOps Notes and Commands

## Alpha

Login to Alpha DB using 
```
mysql -h prod-omega-mysql.nyc3.internal.digitalocean.com -u svc_cloudops_p_ro -p
Password -> 1password (Alpha Prod)
```

## Available Disk 

Usulally we need to reduce the log file size in root. US the following command reduce the file size to 1G
```
truncate -s 1g /var/log/$file_name
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

**Note: Cancelling evets are very risky and can lead to serious consequences, please consult a peer if you are unsure!**

We have different types of events so consulting this [playbook](https://github.com/digitalocean/documentation/blob/master/oncall/playbooks/procedures/cancelling-events.md) is best before proceeding

Run the following to cancel a queued live migrate. Change the queued event type and Job ID for other events
```
event cancel --status=QUEUED --vm-id=354853776 --type=/vm/live_migrate 1935006380
```
Run the following to see event on a signle Droplet
```
hvdclient hv eventinfo -A -d $droplet_id
```
### Using Alpha t get a list of stalled events

Login to Alphas and run the following command to see which events are stuck for a long time 
```
SELECT e.id, e.droplet_id, s.name, e.region_id, e.description
FROM events e
JOIN servers s ON s.id = e.start_server_id
WHERE e.event_type_id IN (7,8)
	AND e.created_at >= NOW() - INTERVAL 2 day
	AND e.updated_at <= NOW() - INTERVAL 6 hour
	AND e.action_status IS NULL;
```
After getting a list, login to the HV from the tables in Alpha and run the following commands to cancel these events
```
ps aux | grep backup
cd /var/lib/libvirt/images
ls | grep raw
cd backups/
ls
hvdclient hv eventinfo --vm-id $dropletID
event fail $eventID --jira inci-number
ls
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


## GPU Repave

Command for GPU provisioning
```
st2 run digitalocean.provision hosts=DHLY044 rack_name="B2R25 (TR6:01:613165:0703)" rack_position=2 role=infra-hypervisor-gpu fleet="do:compute-fleet:gpu-nvidia-h100" os=jammy train=test
```
## IPMI kdump and Reboot

Run the `diag` command for a kdump and the `power off/on` for a IPMI reboot

```
ipmitool -U root -P $DO_IPMI -I lanplus -H $BMC_IP sel elist
ipmitool -U root -I lanplus -H $BMC_IP-P $IPMI_password chassis power diag
ipmitool -U root -I lanplus -H $BMC_IP-P $IPMI_password chassis power status
ipmitool -U root -I lanplus -H $BMC_IP-P $IPMI_password chassis power off
ipmitool -U root -I lanplus -H $BMC_IP-P $IPMI_password chassis power on
```
## OSD 

Verfify the destroy of disk devices using `ceph osd tree | grep osd.###`

Sometimes, DCOPs may ask us to power on the blinking of  storage device, use the following command for that after logging into the storage node
```
$ export device=$device
$ sudo /opt/apps/storman/bin/storman locate start --disk=$device
```
The drive's identification light should now be blinking!

## Reboot Node

Run the following command to grafully shut down a node `sudo shutdown -r +1 `

## Resharding a Bucket

Consult the [playbook](https://github.com/digitalocean/documentation/blob/master/oncall/playbooks/procedures/reshard-spaces-bucket.md) and [glass](https://glass.internal.digitalocean.com/object/bucketresharding/candidates) page to see which buckets need resharding, check the right region as well.

Sometimes we need to stop a reashard and can look at this [playbook](https://github.com/digitalocean/documentation/blob/master/oncall/playbooks/procedures/reshard-spaces-bucket.md#stopping-a-reshard) for that. SSHing listed in here is important to understand.

Command from jump to remove shards
```
samin@prod-jump04:~$ ./caissonctl --region $region --cluster object01 remove-stale-instances start
```
Command to list shards (login to the cluster first)
```
samin@prod-rgw01-object01:~$ sudo radosgw-admin reshard stale-instances list
```

## Repaving a node

Source into the correct region in st2 e.g. for nyc2 use `source env/production nyc2` to repave a node in nyc2.

The command to repave can vary but this is the general syntax
```
st2 run digitalocean.provision hosts=BJBKCP2 role=infra-hypervisor hpw_workflow_wait=false release=true
```
## Sunset of a node

Use the following command to Sunset/retire a node
```
st2 run digitalocean.sunset_workflow hosts=nyc3node4148,nyc3node4135 tower_username="${LDAP_USERNAME}" tower_password="${LDAP_PASSWORD}" jira_ticket=OPS-34167 --async
```
## Vault Passwords

HV Console [Password](https://vault-ui.internal.digitalocean.com/ui/vault/secrets/secret/show/platform/infra-cred-mgr/production/password/root)

Storage Node Console [Password](https://vault-ui.internal.digitalocean.com/ui/vault/secrets/secret/show/platform/infra-cred-mgr/production/password/root_storage)

IPMI Vault [Password](https://vault-ui.internal.digitalocean.com/ui/vault/secrets/secret/show/platform/ipmi/password)
