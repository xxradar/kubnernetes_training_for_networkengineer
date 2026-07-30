```
export AWS_PAGER=""

# Set your desired password here
INSTANCE_PASSWORD='ChangeMe-Str0ng!'

cat > user-data.txt <<EOF
#cloud-config
ssh_pwauth: true
chpasswd:
  expire: false
  users:
    - name: ubuntu
      password: ${INSTANCE_PASSWORD}
      type: text
runcmd:
  - sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
  - rm -f /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
  - systemctl restart ssh
EOF

aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id \
  --region eu-west-3 \
  --instance-type t2.xlarge \
  --associate-public-ip-address \
  --user-data file://user-data.txt \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=30,VolumeType=gp3,DeleteOnTermination=true}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=k8straining}]'
  ```