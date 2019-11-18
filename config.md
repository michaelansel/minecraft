# S3 bucket
* Lifecycle policy to move everything to Intelligent Tiering after 1 day
* Block public access: enabled
* Encryption: AES-256 (uses the shared S3 key)

# Launch Template
* AMI: Latest Amazon Linux 2 AMI
* Instance Type: t3.medium
* Security Group: point at minecraft security group defined below
* Volume: 8GB gp2, encrypted with `aws/ebs` KMS key
* Instance Tags: Name=Minecraft
* Network Interfaces: (none, let it create the default interface)
* Instance Profile: point at the Minecraft IAM role's profile
* T3 Unlimited: enabled
* User Data: use the below

# minecraft Security Group
* TCP 22 from everywhere (could be removed in favor of just SSM, but that's for later)
* TCP 25565 from everywhere
* UDP 25565 from everywhere

# IAM Role/Instance Profile
Allow ec2.amazonaws.com to AssumeRole
Policies:
* AmazonS3FullAccess (this should be tightened up with a custom policy)
* AmazonSSMManagedInstanceCore (grant shell access via the AWS console)

# User Data
Note: Pushover secrets removed (from the `curl` commands)
* TODO update to the latest version of Oracle java

```
#!/bin/bash
yum update -y
curl -L 'https://javadl.oracle.com/webapps/download/AutoDL?BundleId=240717_5b13a193868b4bf28bcb45c792fce896' > java.rpm && yum localinstall -y java.rpm
yum install -y tmux jq python3

cat <<"LAUNCH" >/home/ec2-user/launch.sh
#!/bin/bash

python3 -m venv /home/ec2-user/venv
/home/ec2-user/venv/bin/pip install mcstatus

cat <<"SHUTDOWN" >/home/ec2-user/auto-shutdown.sh
#!/bin/bash
uptime=$(awk '{print $1}' /proc/uptime | cut -d. -f1)
usercount=$(/home/ec2-user/venv/bin/python -c 'import mcstatus ; print(mcstatus.MinecraftServer.lookup("localhost").status().players.online)')
if (( uptime > 3600 && usercount == 0 )) ; then
  curl -s --form-string "token=" --form-string "user=" --form-string "message=Idle $(curl http://icanhazip.com)" https://api.pushover.net/1/messages.json
  /home/ec2-user/save-state.sh
  curl -s --form-string "token=" --form-string "user=" --form-string "message=Goodbye $(curl http://icanhazip.com)" https://api.pushover.net/1/messages.json
  sudo poweroff
fi
SHUTDOWN
chmod +x /home/ec2-user/auto-shutdown.sh
echo "* * * * * /home/ec2-user/auto-shutdown.sh" | crontab -


cat <<"SAVESTATE" >/home/ec2-user/save-state.sh
#!/bin/bash
tmux send-keys -t minecraft:minecraft "/backup" ENTER
sleep 5
tmux send-keys -t minecraft:minecraft "/stop" ENTER
while pgrep java ; do sleep 5 ; done

Bucket=ansel-data
WorldName=FTBAcademyServer
cd "/home/ec2-user"
output="$(date '+%Y-%m-%d') ${WorldName}.zip"
zip -r "$output"  "${WorldName}/"

if aws s3api list-objects --bucket ansel-data --prefix minecraft/ --output text | grep -q -e "$output" ; then
  echo "ERROR file already exists on S3"
  curl -s --form-string "token=" --form-string "user=" --form-string "message=Backup already exists $(curl http://icanhazip.com)" https://api.pushover.net/1/messages.json
else
  if aws s3 cp "$output" s3://ansel-data/minecraft/ ; then
    curl -s --form-string "token=" --form-string "user=" --form-string "message=Backup successful $(curl http://icanhazip.com)" https://api.pushover.net/1/messages.json
  else
    curl -s --form-string "token=" --form-string "user=" --form-string "message=Backup failed $(curl http://icanhazip.com)" https://api.pushover.net/1/messages.json
  fi
fi
SAVESTATE
chmod +x /home/ec2-user/save-state.sh


Bucket=ansel-data
WorldName=FTBAcademyServer
object="$(aws s3api list-objects --bucket "$Bucket" --prefix minecraft/ --output json --query 'Contents[].Key' | jq 'map(select(test("'"$WorldName"'")))[]' -r | sort -n | tail -n 1)"
cd /home/ec2-user
aws s3 cp "s3://${Bucket}/${object}" .
unzip "$(basename "$object")"
cd "/home/ec2-user/${WorldName}"
tmux new-session -d -s minecraft -n minecraft 'bash ServerStart.sh'
curl -s --form-string "token=" --form-string "user=" --form-string "message=Launching $(curl http://icanhazip.com)" https://api.pushover.net/1/messages.json
LAUNCH

sudo -u ec2-user bash -x /home/ec2-user/launch.sh
```
