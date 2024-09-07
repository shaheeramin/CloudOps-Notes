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
### Using Alpha to get a list of stalled events

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

Since this was a backup that was cancelled, head over to Atlantis for this droplet and start the backup manually.

### Event Fail

Make an OPs ticket for event fail -> https://do-internal.atlassian.net/browse/OPS-37200

Run this command for event fail `event fail --email="samin@digitalocean.com" --jira OPS-37200 $event_ID`

### Stalled Events

Playboook for stalled events. Another day, another [playbook](https://github.com/digitalocean/documentation/blob/master/oncall/playbooks/alerts/stalled-backups.md)


## COCTL

COCTL needs a container to run, either run a new one or use these coammnds to enter an existing one 
```
docker ps
docker attach [NAMES]
```
COCTL command to escalate an escalation
```
coctl escalate ESC-#####
```
COCTL command to set state on many servers
```
coctl server set-state syd1node449 syd1node465 "OPS-39210"
```
To remove state, use `""` instead of `"OPS-39210"`

COCTL command to find low droplet HV's
```
coctl alpha servers-with-few-droplets -r $region -f $fleet --model $model_name -d ####
```
COCTL command to find abusive droplets on a node
```
coctl leecher $node_name
```

## DDOS

Run these commands to check for UDP/TCP packaets
```
tcpdump -n -i bond0 -c 1000000 > tdump
cat tdump | grep -v ARP | awk '{print $3}' | sort | uniq -c | sort -nr | head -n5
cat tdump | grep -v ARP | awk '{print $5}' | sort | uniq -c | sort -nr | head -n5
```
```
tcpdump udp -qn -i bond0 -c 100000 > udump
cat udump | grep -v ARP | awk '{print $3}' | sort | uniq -c | sort -nr | head -n5
cat udump | grep -v ARP | awk '{print $5}' | sort | uniq -c | sort -nr | head -n5
```

## Evacuations/Migrations

Full node migration
```
migrate evac $(hostname) --no-confirm --emergency --email --fallback --jira=OPS-#####
```
To search for a Job ID, first login to a node of the same region region and use either of the twom commands
```
migrate jobs
migrate job $jod_ID_####
```
To migrate highest CPU Usage Droplets, use this command (change head -n ## or tail -n ## for the number of droplets to migrate)
```
/usr/local/bin/migrate droplet --safe $(ps -aux  --sort=-%cpu | grep qemu-system-x86_64 | grep Droplet- | cut -d = -f 2  | cut -d , -f 1 | cut -d - -f 2 | head -n  15 | tail -n 9 | xargs)
```
A few Flags we use with migrations
```
--exclude-server
--end-server
--ignore-state
--concurrent=30
--hvlimit=8
--no-live
--force-power-on
--safe
```

To evacuate many nodes, use this command:
```
Multiple Nodes Evac:
/usr/local/bin/migrate evac sfo3node686 sfo3node697 --fallback --hvlimit=6 --concurrent=100
```
`--hvlimit` is the number of concurrent Hyper Visors being evacuated and `--concurrent` is the number of droplets migrating concurrently
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

Use the following command to check an OSD from within the node
```
storman diag --osd ###
```
Verfify the destroy of disk devices using `ceph osd tree | grep osd.###`


Sometimes, DCOPs may ask us to power on the blinking of  storage device, use the following command for that after logging into the storage node
```
$ export device=$device
$ sudo /opt/apps/storman/bin/storman locate start --disk=$device
```
The drive's identification light should now be blinking!

## Paperspace

[Playbook](https://github.com/digitalocean/paperspace-infrastructure-docs/blob/912e805e7df778[…]f9fa5e7358991b8cfb486/playbooks/procedures/vm-find-residency.md) to find a VM in Xen 

Use the following to login to PS Prod DB 
```
psql -h ps-db-prod.cluster-c1xm2pfayrwq.us-east-1.rds.amazonaws.com -U adminps -p 16309 -d paperspace
Password -> prob-db (1Password)
```
For HV info `cat /etc/xensource/pool.conf` is used

### How to find the HV for Xen-Center/XCP-center

Use [retool](https://paperspace.retool.com/) to get HV name and IP -> search machine ID in retool -> apps -> paperspace 360 -> compute -> search bar - pgpu historical
Connect VPN and login using -> ssh root@IP  
Password in 1pass -> paperspace-prod -> the Password OR Use this [playbook](https://github.com/digitalocean/documentation/blob/master/cloudops/docs/PaperSpace/windows-vm.md) to login to our RDP Windows Machine

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
Some nodes would need to be PXE booted to Live and start their repave using their serial number as the hostnames using `dmidecode -s` or `dmidecode -s system-serial-number`

More commands for repaves
```
st2 run digitalocean.provision hosts=abc1node1 train=test hpw_workflow_wait=false
st2 run digitalocean.provision hosts=xxxx,xxxxx,xxxxx  release=true --async
st2 run digitalocean.provision hosts=fra1node3665 release=true --async
st2 execution tail 6621b703c68c7fc4cd30240d
while true; do st2 run digitalocean.hms hosts=fra1node3665; sleep 60; done
st2 run digitalocean.provision hosts=FJLY044 rack_name="B2R28 (TR6:01:613165:0706)" rack_position=2 role=infra-hypervisor-gpu fleet="do:compute-fleet:gpu-nvidia-h100" os=jammy train=test
```
## Sunset of a node

Use the following command to Sunset/retire a node (needs to be run from st2)
```
st2 run digitalocean.sunset_workflow hosts=nyc3node4148,nyc3node4135 tower_username="${LDAP_USERNAME}" tower_password="${LDAP_PASSWORD}" jira_ticket=OPS-34167 --async
```
## VAST

### VAST credential change

[Playbook](https://github.com/digitalocean/documentation/blob/1d00ced569909627566bd03f183624b535d4e36e/oncall/playbooks/procedures/vast-support-credentials.md#L4) to change the credentials for VAST when they need to use our clusters.

[Playbook](https://github.com/digitalocean/documentation/blob/master/oncall/playbooks/procedures/vast.md#access-to-management-interface-vms) to access the UI for VAST Clusters.

## Vault Passwords

HV Console [Password](https://vault-ui.internal.digitalocean.com/ui/vault/secrets/secret/show/platform/infra-cred-mgr/production/password/root)

Storage Node Console [Password](https://vault-ui.internal.digitalocean.com/ui/vault/secrets/secret/show/platform/infra-cred-mgr/production/password/root_storage)

IPMI Vault [Password](https://vault-ui.internal.digitalocean.com/ui/vault/secrets/secret/show/platform/ipmi/password)
