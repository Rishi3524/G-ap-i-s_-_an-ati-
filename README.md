<div style="display: flex; flex-direction: column; justify-content: center; align-items: center; height: 100vh;">

  <h2>Labs 6-9</h2>
  
  <p>Student ID: 23463452</p>
  <p>Student Name: Rishwanth Katherapalle</p>

</div>

# Lab 6

## Set up an EC2 instance

### [1] Create an EC2 micro instance with Ubuntu and SSH into it. 

For this task I used the following Python script from LAB2 named **create_ec2.py** by using **nano create_ec2.py**: 

**Step 1** Copy the following script and paste it and then CTRL+X, Y, ENTER to save it.
   
```
from __future__ import annotations

import os
from pathlib import Path

import boto3
from botocore.exceptions import ClientError, NoCredentialsError

STUDENT_NUMBER = "23463452"
REGION = "ap-northeast-1"
AMI_ID = "ami-054400ced365b82a0"
INSTANCE_TYPE = "t3.micro"


def _default_vpc_id(ec2) -> str: # Basically to get the vpc id for creating the security group using the functions.
    vpcs = ec2.describe_vpcs(Filters=[{"Name": "isDefault", "Values": ["true"]}]).get("Vpcs", [])
    if not vpcs:
        raise RuntimeError("No default VPC in region.")
    return vpcs[0]["VpcId"]


def _default_subnet_id(ec2, vpc_id: str) -> str: # Needed to run the instance of EC2.
    subnets = ec2.describe_subnets(
        Filters=[{"Name": "vpc-id", "Values": [vpc_id]}, {"Name": "default-for-az", "Values": ["true"]}]
    ).get("Subnets", [])
    if not subnets:
        subnets = ec2.describe_subnets(Filters=[{"Name": "vpc-id", "Values": [vpc_id]}]).get("Subnets", [])
        if not subnets:
            raise RuntimeError("No subnet in default VPC.")
    return sorted(subnets, key=lambda s: s["AvailabilityZone"])[0]["SubnetId"]


def main() -> int:
    sg_name = f"{STUDENT_NUMBER}-sg"
    key_name = f"{STUDENT_NUMBER}-key"
    inst_name = f"{STUDENT_NUMBER}-vm"
    pem_path = Path.home() / ".ssh" / f"{key_name}.pem"

    try:
        ec2 = boto3.client("ec2", region_name=REGION)

        # 1) Creates a Security Group
        vpc_id = _default_vpc_id(ec2)
        sg_resp = ec2.create_security_group(
            Description="security group for development environment",
            GroupName=sg_name,
            VpcId=vpc_id,
        )
        sg_id = sg_resp["GroupId"]
        # 2) Allows SSH access and permit instance to receive traffic from port 22
        ec2.authorize_security_group_ingress(
            GroupId=sg_id,
            IpPermissions=[
                {
                    "IpProtocol": "tcp",
                    "FromPort": 22,
                    "ToPort": 22,
                    "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "SSH"}],
                }
            ],
        )

        # 3) Create Key Pair and write the permissions
        pem_path.parent.mkdir(parents=True, exist_ok=True)
        key_resp = ec2.create_key_pair(KeyName=key_name)
        pem_path.write_text(key_resp["KeyMaterial"], encoding="utf-8")
        os.chmod(pem_path, 0o400)

        # 4) Run the instance
        subnet_id = _default_subnet_id(ec2, vpc_id)
        run_resp = ec2.run_instances(
            ImageId=AMI_ID,
            InstanceType=INSTANCE_TYPE,
            MinCount=1,
            MaxCount=1,
            KeyName=key_name,
            NetworkInterfaces=[
                {
                    "DeviceIndex": 0,
                    "SubnetId": subnet_id,
                    "AssociatePublicIpAddress": True,
                    "Groups": [sg_id],
                }
            ],
        )
        instance_id = run_resp["Instances"][0]["InstanceId"]

        # 5) Tag the instance using a Name 
        ec2.create_tags(Resources=[instance_id], Tags=[{"Key": "Name", "Value": inst_name}])

        # 6) get Public IP
        ec2.get_waiter("instance_running").wait(InstanceIds=[instance_id])
        described = ec2.describe_instances(InstanceIds=[instance_id])
        public_ip = described["Reservations"][0]["Instances"][0].get("PublicIpAddress", "")
        if public_ip:
            print(f"PublicIp: {public_ip}")
            print(f"SSH: ssh -i '{pem_path}' ubuntu@{public_ip}")
        return 0

    except NoCredentialsError:
        print("ERROR: Configure AWS credentials (e.g., `aws configure`).")
    except ClientError as e:
        print(f"AWS ERROR: {e.response.get('Error', {}).get('Code', 'ClientError')}: {e}")
    except Exception as e:
        print(f"ERROR: {e}")

    return 1


if __name__ == "__main__":
    raise SystemExit(main())

```
**The breakdown of the functions used in the script**:

**_default_vpc_id(ec2) -> str**:

Input: takes a Boto3 EC2 client.
Action: It filters describe_vpcs by isDefault=true; chooses the first match.
Output: The default VPC ID (e.g., vpc-abc123) is given.

**_default_subnet_id(ec2, vpc_id: str) -> str**:

Input: It takes a EC2 client and a VPC ID.
Action: It tries describe_subnets with the filters vpc-id=<vpc_id> and default-for-az=true (auto public IP). If none of them then, fall back to any subnet in the VPC. It is then sorted by AvailabilityZone and picks the first one to determine.
Output: a Subnet ID is in which the EC2 instance lives.

**main() -> int**:

This implements all the steps of AWS CLI preparing the names/paths: builds security group name, key name, instance name, and PEM path under ~/.ssh/.

EC2 client: boto3.client("ec2", region_name=REGION).

1) sg_resp = ec2.create_security_group(): creates SG in the default VPC with description; saves sg_id.

2) ec2.authorize_security_group_ingress(): opens TCP/22 from 0.0.0.0/0 (world-open for convenience; risky on the open internet).

3) key_resp = ec2.create_key_pair(): creates key pair, writes PEM to disk, sets 0400 permissions.

4) run_resp = ec2.run_instances(): chooses a subnet from the above function, runs one t3.micro from the given AMI; attaches SG; ensures public IP via AssociatePublicIpAddress=True.

5) ec2.create_tags(): sets Name=23463452-vm.

6) desc = ec2.describe_instances():describes the instance.

Returns: 0 on success, 1 on handled errors.

**Note**: Refer to [page](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html) for details of all thefunctions used to initiate the EC2instance. 

**Step 2** Now run this script using:

```
python3 create_ec2.py
```

Instance is created:

<img width="1919" height="491" alt="Screenshot 2025-09-17 104418" src="https://github.com/user-attachments/assets/bc157c7c-7b31-4bb8-92ac-c8984f0ad8c4" />


### [2] Install the Python 3 virtual environment package

Install them using the following commands

```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install python3-venv
```
It is easier now if you change the bash to operate as sudo using:

```
sudo bash
```

### [3] Access a directory  

Create a directory with a path `/opt/wwc/mysites` and `cd` into the directory using:

```
mkdir /opt/wwc/mysites
cd /opt/wwc/mysites
```

### [4] Set up a virtual environment

```
python3 -m venv myvenv
```

### [5] Activate the virtual environment

```
source myvenv/bin/activate

pip install django

django-admin startproject lab

cd lab

python3 manage.py startapp polls
```

### [6] Install nginx

```
apt install nginx
```

### [7] Configure nginx

edit `/etc/nginx/sites-enabled/default` by **nano /etc/nginx/sites-enabled/default** and replace the contents of the file with the below
and then CTRL+X, Y, ENTER to save it:

```
server {
  listen 80 default_server;
  listen [::]:80 default_server;

  location / {
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Real-IP $remote_addr;

    proxy_pass http://127.0.0.1:8000;
  }
}
```

### [8] Restart nginx

```
service nginx restart
```


### [9] Access your EC2 instance

Go to your app directory: `cd /opt/wwc/mysites/lab`, run:

```
python3 manage.py runserver 8000
```

Open a browser and enter the IP address of your EC2 instance.

<img width="1469" height="554" alt="Screenshot 2025-09-23 205457" src="https://github.com/user-attachments/assets/099c329b-577e-4ecc-8046-35c567fa6548" />


<img width="1919" height="1153" alt="Screenshot 2025-09-23 205029" src="https://github.com/user-attachments/assets/2331513b-7cdb-4e53-ba62-767ce5f87467" />


## Set up Django inside the created EC2 instance

### [1] Edit the following files (create them if not exist)

edit polls/views.py by **nano polls/views.py** and replace the contents of the file with the below
and then CTRL+X, Y, ENTER to save it:

```
from django.http import HttpResponse

def index(request):
    return HttpResponse("Hello, world.")
```

edit polls/urls.py by **nano polls/urls.py** and replace the contents of the file with the below
and then CTRL+X, Y, ENTER to save it:

```
from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
```

edit lab/urls.py by **nano lab/urls.py** and replace the contents of the file with the below
and then CTRL+X, Y, ENTER to save it:

```
from django.urls import include, path
from django.contrib import admin

urlpatterns = [
    path('polls/', include('polls.urls')),
    path('admin/', admin.site.urls),
]
```

### [2] Run the web server again

```
python3 manage.py runserver 8000
```

### [3] Access the EC2 instance

Access the URL: http://\<ip address of your EC2 instance>/polls/, and output what you've got from above. 

**NOTE**: remember to put the /polls/ on the end and you may need to restart nginx if it does not work.

<img width="1476" height="583" alt="Screenshot 2025-09-23 210638" src="https://github.com/user-attachments/assets/c72176aa-5c60-4fce-afe1-13e5bd9aae75" />

<img width="1919" height="119" alt="Screenshot 2025-09-23 210609" src="https://github.com/user-attachments/assets/43a9b76c-ec80-42af-8763-e5bd71a76eb6" />


## Set up an ALB

### [1] Create an application load balancer


### [2] Health check


### [3] Access


<div style="page-break-after: always;"></div>

# Lab 7

## [1]Set up an EC2 instance

For this task I used the following Python script from LAB2 named **create_ec2.py** by using **nano create_ec2.py**: 

**Step 1** Copy the following script and paste it and then CTRL+X, Y, ENTER to save it.
   
```
from __future__ import annotations

import os
from pathlib import Path

import boto3
from botocore.exceptions import ClientError, NoCredentialsError

STUDENT_NUMBER = "23463452"
REGION = "ap-northeast-1"
AMI_ID = "ami-054400ced365b82a0"
INSTANCE_TYPE = "t3.micro"


def _default_vpc_id(ec2) -> str: # Basically to get the vpc id for creating the security group using the functions.
    vpcs = ec2.describe_vpcs(Filters=[{"Name": "isDefault", "Values": ["true"]}]).get("Vpcs", [])
    if not vpcs:
        raise RuntimeError("No default VPC in region.")
    return vpcs[0]["VpcId"]


def _default_subnet_id(ec2, vpc_id: str) -> str: # Needed to run the instance of EC2.
    subnets = ec2.describe_subnets(
        Filters=[{"Name": "vpc-id", "Values": [vpc_id]}, {"Name": "default-for-az", "Values": ["true"]}]
    ).get("Subnets", [])
    if not subnets:
        subnets = ec2.describe_subnets(Filters=[{"Name": "vpc-id", "Values": [vpc_id]}]).get("Subnets", [])
        if not subnets:
            raise RuntimeError("No subnet in default VPC.")
    return sorted(subnets, key=lambda s: s["AvailabilityZone"])[0]["SubnetId"]


def main() -> int:
    sg_name = f"{STUDENT_NUMBER}-sg"
    key_name = f"{STUDENT_NUMBER}-key"
    inst_name = f"{STUDENT_NUMBER}-vm"
    pem_path = Path.home() / ".ssh" / f"{key_name}.pem"

    try:
        ec2 = boto3.client("ec2", region_name=REGION)

        # 1) Creates a Security Group
        vpc_id = _default_vpc_id(ec2)
        sg_resp = ec2.create_security_group(
            Description="security group for development environment",
            GroupName=sg_name,
            VpcId=vpc_id,
        )
        sg_id = sg_resp["GroupId"]
        # 2) Allows SSH access and permit instance to receive traffic from port 22
        ec2.authorize_security_group_ingress(
            GroupId=sg_id,
            IpPermissions=[
                {
                    "IpProtocol": "tcp",
                    "FromPort": 22,
                    "ToPort": 22,
                    "IpRanges": [{"CidrIp": "0.0.0.0/0", "Description": "SSH"}],
                }
            ],
        )

        # 3) Create Key Pair and write the permissions
        pem_path.parent.mkdir(parents=True, exist_ok=True)
        key_resp = ec2.create_key_pair(KeyName=key_name)
        pem_path.write_text(key_resp["KeyMaterial"], encoding="utf-8")
        os.chmod(pem_path, 0o400)

        # 4) Run the instance
        subnet_id = _default_subnet_id(ec2, vpc_id)
        run_resp = ec2.run_instances(
            ImageId=AMI_ID,
            InstanceType=INSTANCE_TYPE,
            MinCount=1,
            MaxCount=1,
            KeyName=key_name,
            NetworkInterfaces=[
                {
                    "DeviceIndex": 0,
                    "SubnetId": subnet_id,
                    "AssociatePublicIpAddress": True,
                    "Groups": [sg_id],
                }
            ],
        )
        instance_id = run_resp["Instances"][0]["InstanceId"]

        # 5) Tag the instance using a Name 
        ec2.create_tags(Resources=[instance_id], Tags=[{"Key": "Name", "Value": inst_name}])

        # 6) get Public IP
        ec2.get_waiter("instance_running").wait(InstanceIds=[instance_id])
        described = ec2.describe_instances(InstanceIds=[instance_id])
        public_ip = described["Reservations"][0]["Instances"][0].get("PublicIpAddress", "")
        if public_ip:
            print(f"PublicIp: {public_ip}")
            print(f"SSH: ssh -i '{pem_path}' ubuntu@{public_ip}")
        return 0

    except NoCredentialsError:
        print("ERROR: Configure AWS credentials (e.g., `aws configure`).")
    except ClientError as e:
        print(f"AWS ERROR: {e.response.get('Error', {}).get('Code', 'ClientError')}: {e}")
    except Exception as e:
        print(f"ERROR: {e}")

    return 1


if __name__ == "__main__":
    raise SystemExit(main())

```
**The breakdown of the functions used in the script**:

**_default_vpc_id(ec2) -> str**:

Input: takes a Boto3 EC2 client.
Action: It filters describe_vpcs by isDefault=true; chooses the first match.
Output: The default VPC ID (e.g., vpc-abc123) is given.

**_default_subnet_id(ec2, vpc_id: str) -> str**:

Input: It takes a EC2 client and a VPC ID.
Action: It tries describe_subnets with the filters vpc-id=<vpc_id> and default-for-az=true (auto public IP). If none of them then, fall back to any subnet in the VPC. It is then sorted by AvailabilityZone and picks the first one to determine.
Output: a Subnet ID is in which the EC2 instance lives.

**main() -> int**:

This implements all the steps of AWS CLI preparing the names/paths: builds security group name, key name, instance name, and PEM path under ~/.ssh/.

EC2 client: boto3.client("ec2", region_name=REGION).

1) sg_resp = ec2.create_security_group(): creates SG in the default VPC with description; saves sg_id.

2) ec2.authorize_security_group_ingress(): opens TCP/22 from 0.0.0.0/0 (world-open for convenience; risky on the open internet).

3) key_resp = ec2.create_key_pair(): creates key pair, writes PEM to disk, sets 0400 permissions.

4) run_resp = ec2.run_instances(): chooses a subnet from the above function, runs one t3.micro from the given AMI; attaches SG; ensures public IP via AssociatePublicIpAddress=True.

5) ec2.create_tags(): sets Name=23463452-vm.

6) desc = ec2.describe_instances():describes the instance.

Returns: 0 on success, 1 on handled errors.

**Note**: Refer to [page](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ec2.html) for details of all thefunctions used to initiate the EC2instance. 

**Step 2** Now run this script using:

```
python3 create_ec2.py
```

Instance is created:

<img width="1919" height="491" alt="Screenshot 2025-09-17 104418" src="https://github.com/user-attachments/assets/bc157c7c-7b31-4bb8-92ac-c8984f0ad8c4" />

## [2]Install and configure Fabric

Now install fabric using:

```
pip install fabric
```

You will need to create a config file in ~/.ssh with the contents:

```
nano  ~/.ssh/config
```

Copy the following script and paste it and then CTRL+X, Y, ENTER to save it.

```
Host <your EC2 instance name>
	Hostname <your EC2 instance public IPv4 DNS>
	User ubuntu
	UserKnownHostsFile /dev/null
	StrictHostKeyChecking no
	PasswordAuthentication no
	IdentityFile <path to your private key>
```

Replace `<your EC2 instance name>` and `<your EC2 instance public IPv4 DNS>` above and below with your real ones from step 1 where you created your ec2  instance.
**Note:** You can get these after running the script as well as these details are printed on the terminal while initiating the ec2 instance.

Rely on the fabric code below to connect to your instance.

```
python3
>>> from fabric import Connection
>>> c = Connection('<your EC2 instance name>')
>>> result = c.run('uname -s')
Linux
>>>
```

## [3]Use Fabric for automation

<div style="page-break-after: always;"></div>

# Lab 8

## [1]Create a Dockerfile and build a Docker image

**Step 1**
Make a directory for your lab: " mkdir lab8 " and then go into that directory: " cd lab8 ".

**Step 2**
Create your Dockerfile: 

```
nano Dockerfile
```

Then paste the below script into it and then CTRL+X, Y, ENTER to save it.

```
FROM python:3.10

RUN pip install jupyter boto3 sagemaker awscli
RUN mkdir /notebook

# Use a sample access token
ENV JUPYTER_ENABLE_LAB=yes
ENV JUPYTER_TOKEN="CITS5503"

# Allow access from ALL IPs
RUN jupyter notebook --generate-config
RUN echo "c.NotebookApp.ip = '0.0.0.0'" >> /root/.jupyter/jupyter_notebook_config.py

# Copy the ipynb file
RUN wget -P /notebook https://raw.githubusercontent.com/zhangzhics/CITS5503_Sem2/master/Labs/src/LabAI.ipynb

WORKDIR /notebook
EXPOSE 8888

CMD ["jupyter", "notebook", "--ip=0.0.0.0", "--port=8888", "--no-browser", "--allow-root"]

```
**Step 3**
Now build the dockerfile using:

```
docker build -t YOUR_STUDENT_NUMBER-lab8 .
```
**Note:** Replace student_number with your own.

**Step 4**
After the above script finishes without errors, you can test your image locally by running:

```
docker run -p 8888:8888 YOUR_STUDENT_NUMBER-lab8
```

You can go to 127.0.0.1:8888 to check if a notebook file has been downloaded successfully.

<img width="1919" height="1059" alt="image" src="https://github.com/user-attachments/assets/998e007f-ba8d-49cd-9209-94467a7c2be1" />

The token is **"CITS5503"** as coded in the above script.

<img width="1919" height="518" alt="Screenshot 2025-10-13 134927" src="https://github.com/user-attachments/assets/f7d89e5d-d518-484f-924a-98f81c0b232e" />


## [2]Prepare ECR via Boto3 scripts on your local machine

### ECR
**Step 1**
Now we use this Boto3 script to create a ECR repository:

```
nano create_ecr_repo.py
```

Then paste the below script into it and then CTRL+X, Y, ENTER to save it.

```
import boto3

def create_or_check_repository(repository_name):
    ecr_client = boto3.client('ecr')
    try:
        response = ecr_client.describe_repositories(repositoryNames=[repository_name])
        repository_uri = response['repositories'][0]['repositoryUri']
    except ecr_client.exceptions.RepositoryNotFoundException:
        response = ecr_client.create_repository(repositoryName=repository_name)
        repository_uri = response['repository']['repositoryUri']
    return repository_uri


repository_name = 'YOUR_STUDENT_NUMBER' + '_ecr_repo'
repository_uri = create_or_check_repository(repository_name)
print("ECR URI:", repository_uri)
```
**Note:** Replace student_number with your own.

Then run the script:  **python3 create_ecr_repo.py**

This gives you a **ECR URI**, and you need use this uri to push your Dockerfile into the ECR repository.

**Step 2**
The following code uses the AWS Boto3 to obtain an authorisation token from AWS ECR, decodes it to retrieve the username and password, 
and then generates a Docker login command. This allows the user to log into ECR using the produced command, enabling them to push and pull Docker images.

To get the Docker token:

```
nano create_docker_login_cmd_ecr.py
```
Then paste the below script into it and then CTRL+X, Y, ENTER to save it.

```
import boto3
import base64
def get_docker_login_cmd():
    ecr_client = boto3.client('ecr')
    token = ecr_client.get_authorization_token()
    username, password = base64.b64decode(token['authorizationData'][0]['authorizationToken']).decode().split(':')
    registry = token['authorizationData'][0]['proxyEndpoint']
    return f"docker login -u {username} -p {password} {registry}"

print(get_docker_login_cmd())
```

Run the script: **python3 create_docker_login_cmd_ecr.py**

You will get the command to grant the Docker access to the ECR repo. You have to run the output command from the 
script above in your terminal, and you will get "Login Succeeded" from the terminal if it goes well.

<img width="1898" height="854" alt="Screenshot 2025-10-13 140902" src="https://github.com/user-attachments/assets/33bbb5f6-4700-43d7-b713-3dc3a94e699e" />


**NOTE**: If you're using WSL 2, DNS often breaks and returns an error message such as "no such host". If so, try this:
- Edit your WSL resolv.conf:
  
```bash
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
```
- Restart WSL2

## [3]Push a local Docker image onto ECR

Once you see the "Login Succeeded" message, you tag and push your local Docker image to your ECR repository via terminal. To tag your image as the latest version, do:

```
docker tag YOUR_STUDENT_NUMBER-lab8:latest YOUR_ECR_URI:latest
```
Replace your student number.

Then push to your ECR:

```
docker push YOUR_ECR_URI:latest
```
**YOUR_ECR_URI** is the one we get from the above step.

<img width="1887" height="392" alt="Screenshot 2025-10-13 140930" src="https://github.com/user-attachments/assets/76e316cb-3ce6-4035-8143-69a9c0e67b7e" />

The step above takes some time to upload, which depends on your internet connection.

<img width="1919" height="526" alt="Screenshot 2025-10-13 140945" src="https://github.com/user-attachments/assets/18e39dc9-5262-4ea6-9840-573344aafd00" />


## [4]Deploy your Docker image onto ECS


### Create a task definition for an ECS task:

To inject environment variables into your ECS task, add an environment field in your container definition as follows:

**Step 1**

```
nano create_task_def_ecs.py
```
Then paste the below script into it and then CTRL+X, Y, ENTER to save it.
Replace student number with your own number and ECR_URI from the above steps

```
import boto3

def create_ecs_task_definition(
    client, image_uri, account_id, task_role_name, execution_role_name, student_id,
    environment_dict=None,port=8888, cpu='256', memory='512'
):
    task_role_arn = f'arn:aws:iam::{account_id}:role/{task_role_name}'
    execution_role_arn = f'arn:aws:iam::{account_id}:role/{execution_role_name}'

    env_list = [{'name': k, 'value': v} for k, v in (environment_dict or {}).items()]
    
    response = client.register_task_definition(
        family=f'{student_id}-task-family',
        networkMode='awsvpc',
        requiresCompatibilities=['FARGATE'],
        cpu=cpu,
        memory=memory,
        taskRoleArn=task_role_arn,
        executionRoleArn=execution_role_arn,
        containerDefinitions=[
            {
                'name': f'{student_id}-container',
                'image': image_uri,
                'essential': True,
                'portMappings': [
                    {
                        'containerPort': port,
                        'hostPort': port,
                        'protocol': 'tcp'
                    },
                ]
            },
        ],
    )
    return response

account_id = '489389878001'
student_id = "YOUR_STUDENT_NUMBER"
task_role_name = 'SageMakerRole'
execution_role_name = 'ecsTaskExecutionRole'
image_uri = 'YOUR_ECR_URI'


ecs_client = boto3.client('ecs')

task_definition = create_ecs_task_definition(
    ecs_client,
    image_uri,
    account_id,
    task_role_name,
    execution_role_name,
    student_id,
    port=8888                      
)
print("Task Definition ARN:", task_definition['taskDefinition']['taskDefinitionArn'])

```
**Step 2**
Run the script: **python3 create_task_def_ecs.py**

The printed task definition ARN is used for the next step.

<img width="1201" height="104" alt="Screenshot 2025-10-13 142844" src="https://github.com/user-attachments/assets/4e6eadb2-d573-4603-9513-703cfdb6777b" />


### [5]Create an ECS service:

**Step1**
First, create a cluster then create an ECS service:

```
nano create_ecs_service.py
```
Then paste the below script into it and then CTRL+X, Y, ENTER to save it.
Replace student number with your own number and ECR_URI from the above steps

```
import boto3

def create_ecs_cluster(client, cluster_name):
    response = client.create_cluster(
        clusterName=cluster_name
    )
    return response

def create_ecs_service(client, cluster_name, service_name, task_definition, subnet_ids, security_group_ids):
    response = client.create_service(
        cluster=cluster_name,
        serviceName=service_name,
        taskDefinition=task_definition,
        desiredCount=1,
        launchType='FARGATE',
        networkConfiguration={
            'awsvpcConfiguration': {
                'subnets': subnet_ids,
                'securityGroups': security_group_ids,
                'assignPublicIp': 'ENABLED'
            }
        },
        deploymentConfiguration={
            'maximumPercent': 200,
            'minimumHealthyPercent': 100
        }
    )
    return response

#This function is to check when the service becomes stable
def wait_for_service_stability(client, cluster_name, service_name):
    waiter = client.get_waiter('services_stable')
    waiter.wait(cluster=cluster_name, services=[service_name])

ecs_client = boto3.client('ecs')

student_id = "YOUR_STUDENT_NUMBER"
ECR_image_uri = 'YOUR_ECR_URI'

cluster_name = student_id + '-cluster'
create_ecs_cluster(ecs_client, cluster_name)

service_name = student_id + '-service'
task_definition = 'YOUR_TASK_DEFINITION_ARN'
subnet_id_1= 'YOUR_SUBNET_ID_1'
subnet_id_2= 'YOUR_SUBNET_ID_2'
subnet_id_3= 'YOUR_SUBNET_ID_3'

subnet_ids = [subnet_id_1, subnet_id_2, subnet_id_3]
security_group_ids = ['YOUR_SECURITY_GROUP_ID']

ecs_client = boto3.client('ecs')

service_response = create_ecs_service(ecs_client, cluster_name, service_name, task_definition, subnet_ids, security_group_ids)
print(f'ECS Service created: {service_response["service"]["serviceArn"]}')

print(f'Waiting for service {service_name} to become stable...')
wait_for_service_stability(ecs_client, cluster_name, service_name)
print(f'Service {service_name} is now stable.')

```

**Step 2**  
Security Group setup (via Console)
Go to VPC → Security Groups → Create security group

Name: ecs-lab8-sg
VPC: default

Inbound rules:
Type: Custom TCP, Port: 8888, Source: 0.0.0.0/0

Outbound rules:
Type: HTTPS, Port: 443, Destination: 0.0.0.0/0

Copy the Security Group ID (e.g. sg-0abcd1234ef567890) for the above code.

**Step 3** — Find your subnet IDs

In AWS Console:

Navigate to VPC → Subnets

Note 3 subnet IDs from your default VPC in ap-northeast-1a, 1b, 1c
For example:
subnet-0a1b2c3d4e5f6a7b
subnet-1a2b3c4d5e6f7a8b
subnet-2a3b4c5d6e7f8a9b

<img width="1913" height="746" alt="Screenshot 2025-10-13 143112" src="https://github.com/user-attachments/assets/4e8b1d34-e076-4d5c-b852-0f2f0f80d40f" />

<img width="1919" height="846" alt="Screenshot 2025-10-13 143827" src="https://github.com/user-attachments/assets/c826a93b-9a93-4ed3-b23c-749a5f9e5b6a" />

**Step 4**
Now run the above script: **python3 create_ecs_service.py**

### [6]Get a public IP address

Remember to update relevant variables below such as cluster and service names after running the above script in the previous step to create a service:
Then run this command after running the above script.

```
aws ecs describe-tasks \
    --cluster YOUR_CLUSTER_NAME \
    --tasks $(aws ecs list-tasks --cluster YOUR_CLUSTER_NAME --service-name YOUR_SERVICE_NAME --query 'taskArns[0]' --output text) \
    --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' \
    --output text | xargs -I {} aws ec2 describe-network-interfaces \
    --network-interface-ids {} \
    --query 'NetworkInterfaces[0].Association.PublicIp' \
    --output text
```
<img width="1686" height="488" alt="Screenshot 2025-10-13 150442" src="https://github.com/user-attachments/assets/bd93f624-5ad2-423b-ab3d-06f58c23c3dc" />

Note the IP address from this step.

Open a browser and navigate to the following address to run it within your ECS. Your public IP address was returned in the previous step.

```
<YOUR PUBLIC IP>:8888
```
<img width="1915" height="1110" alt="Screenshot 2025-10-13 150737" src="https://github.com/user-attachments/assets/0f63b564-b263-49e9-b0b6-d3ae21311c35" />

## Run Hyperparameter Tuning Jobs

For this step, it is detailed in the notebook [here](https://github.com/zhangzhics/CITS5503_Sem2/blob/master/Labs/src/LabAI.ipynb). 


<div style="page-break-after: always;"></div>

# Lab 9

## AWS Comprehend

AWS Comprehend offers different services to analyse text using machine learning. With Comprehend API, you will be able to perform common NLP tasks such as sentiment analysis, or simply detecting the language from the text.

For example, to detect the language used in a given text using boto3 you can use the following code:
```python
import boto3
client = boto3.client('comprehend')

# Detect Entities
response = client.detect_dominant_language(
    Text="The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
)

print(response['Languages'])
```

By executing the code above, we will get something like this:
```
[{'LanguageCode': 'en', 'Score': 0.9961233139038086}]
```
This means that the detected language is 'en' (English) and has a confidence in the prediction greater than 0.99. 

### Detect Languages from different texts

#### [1] Modify the code above

We are modifying the above code to detect different languages using the AWS Comprehend API `detect_dominant_language()`
and `boto3` for texts of 4 different langauges and we format in a way so that the output would be printing message in
the format "<predicted_language> was detected with confidence". Here we replace the language code with it's actual 
name and the confidence is represented as a percentage.

We use these texts from English, Italian, Spanish and French to test the the AWS Comprehend API `detect_dominant_language()`:

**English:**
"The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799."

**Spanish:**
"El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote de la Mancha."

**French:**
"Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout reconstruire Pour tout reconstruire Je l'aime à mourir"
[From the Song: "Je l'Aime a Mourir" - Francis Cabrel ]

**Italian:**
"L'amor che move il sole e l'altre stelle."
[Quote from "Divine Comedy" - Dante Alighieri]

### Step 1
Now we create a script using the command: 
```
nano detect_lang.py
```
Paste the below script and press CTRL+X and Y and ENTER.

Use the following `detect_lang.py` script to test the above texts and get our desired output format:

```
import boto3

client = boto3.client('comprehend')

# Texts in different languages
Texts = [
    "The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
    "El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más
     destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote
     de la Mancha.",
    "Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout
     reconstruire Pour tout reconstruire Je l'aime à mourir",
    "L'amor che move il sole e l'altre stelle."
]

# Dictionary to map language codes to their abbrevations
lang_dict = {
    'en': 'English',
    'es': 'Spanish',
    'fr': 'French',
    'it': 'Italian'
}

for text in Texts:
    response = client.detect_dominant_language(Text=text)
    lang_code = response['Languages'][0]['LanguageCode']
    confidence = response['Languages'][0]['Score'] * 100
    l_name = lang_dict.get(lang_code)
    print(f"{l_name} was detected with {confidence:.1f}% confidence")

```

We use a list name `Text` to store the texts of different languages, which we intend to identify.
Then we map the language codes to the language names to get the output in the desired format using
a dictionary called `lang_dict`.

AWS Comprehend returns two key values for each detected language:
  LanguageCode (like 'en', 'es', 'fr', 'it')
  Score (the confidence value, between 0 and 1) 

To get the desired output, we convert these short codes into full names using this dictionary.
`response = client.detect_dominant_language(Text=text)`.

This line calls the Comprehend API and sends one piece of text at a time to be analyzed in the for loop.
The response is a Python dictionary (JSON-style object) that contains details about the detected languages and confidence scores.

Example response:
`{'Languages': [{'LanguageCode': 'fr', 'Score': 0.9934}]}`.

Now we process this to get the actual name mapped to the language code and the percentage from score by
multiplying it with 100. This gives us our desired output.

### Step2
You can run the script using:
```
python3 detect_lang.py
```

This will give you the output in the format "<predicted_language> was detected with confidence" for the above texts.

### Analyse sentiment 

### Step 1
Now we create a script using the command: 
```
nano detect_sentiment.py
```
Paste the below script and press CTRL+X and Y and ENTER.

Use the following `detect_sentiment.py` script to test the above texts for sentiment to see if the data is positive, negative or neutral:

```
import boto3

client = boto3.client('comprehend')

# Texts in different languages
Texts = [
    "The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
    "El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote de la Mancha.",
    "Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout reconstruire Pour tout reconstruire Je l'aime à mourir",
    "L'amor che move il sole e l'altre stelle."
]

# Dictionary to map language codes to their abbrevations
lang_dict = {
    'en': 'English',
    'es': 'Spanish',
    'fr': 'French',
    'it': 'Italian'
}

for text in Texts:
    response = client.detect_dominant_language(Text=text)
    lang_code = response['Languages'][0]['LanguageCode']

    # Detect sentiment using the detected language code
    senti_response = client.detect_sentiment(Text=text, LanguageCode=lang_code)
    sentiment = senti_response['Sentiment']
    sentiment_scores = senti_response['SentimentScore']

    print(f"Sentiment: {sentiment}")

```
Here we just extend from the above code by using the API `client.detect_syntax(Text=text, LanguageCode=lang_code)`
where the lang_code is obtained from the response for the detect_dominant_language(). The response for the detect syntax
is of the format:
{
    'Sentiment': 'POSITIVE'|'NEGATIVE'|'NEUTRAL'|'MIXED',
    'SentimentScore': {
        'Positive': ...,
        'Negative': ...,
        'Neutral': ...,
        'Mixed': ...
    }
}
So, we use the "Sentiment" key of this response to get our sentiment analysis for our text.

### Step2
You can run the script using:
```
python3 detect_sentiment.py
```

This will give you the output in the format "Sentiment: Positive/Negative/Neutral" for the above texts.

### Detect entities

### Step 1
Now we create a script using the command: 
```
nano detect_entities.py
```
Paste the below script and press CTRL+X and Y and ENTER.

Use the following `detect_entities.py` script to test the above texts and find the entities of these texts:

```
import boto3

client = boto3.client('comprehend')

# Texts in different languages
Texts = [
    "The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
    "El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote de la Mancha.",
    "Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout reconstruire Pour tout reconstruire Je l'aime à mourir",
    "L'amor che move il sole e l'altre stelle."
]

# Dictionary to map language codes to their abbrevations
lang_dict = {
    'en': 'English',
    'es': 'Spanish',
    'fr': 'French',
    'it': 'Italian'
}

for text in Texts:
    response = client.detect_dominant_language(Text=text)
    lang_code = response['Languages'][0]['LanguageCode']

    # Detect entities using the detected language code
    entity_response = client.detect_entities(Text=text, LanguageCode=lang_code)
    for entity in entity_response['Entities']:
        print(f"  - {entity['Text']} : ({entity['Type']})")
    
```
Here we just extend from the above code by using the API `client.detect_entities(Text=text, LanguageCode=lang_code)`
where the lang_code is obtained from the response for the detect_dominant_language(). The response for the detect entity
is of the format:
{
    'Entities': [
        {
            'Score': ...,
            'Type': 'PERSON'|'LOCATION'|'ORGANIZATION'|'COMMERCIAL_ITEM'|'EVENT'|'DATE'|'QUANTITY'|'TITLE'|'OTHER',
            'Text': 'string',
            ....
        }
      ]
    }

So, we use the 'type' and 'text' of the "Entities" response to get the entities and it's type in our texts.

### Step2
You can run the script using:
```
python3 detect_entities.py
```

This will give you the output in the format " 'text' : 'type' " for the above texts.

**What is an entity?**

An **entity** is a specific object that is mentioned in the text. Specifically, an entity can be any name of a person, place, organization or a true date, time, or number.
So basically, entities refer to a true situation or object that has individual meaning when referred to in a sentence, and is marked specifically in the sentence.

### Detect keyphrases

### Step 1
Now we create a script using the command: 
```
nano detect_key_phrases.py
```
Paste the below script and press CTRL+X and Y and ENTER.

Use the following `detect_key_phrases.py` script to test the above texts and get their key phrases:

```
import boto3

client = boto3.client('comprehend')

# Texts in different languages
Texts = [
    "The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
    "El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote de la Mancha.",
    "Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout reconstruire Pour tout reconstruire Je l'aime à mourir",
    "L'amor che move il sole e l'altre stelle."
]

# Dictionary to map language codes to their abbrevations
lang_dict = {
    'en': 'English',
    'es': 'Spanish',
    'fr': 'French',
    'it': 'Italian'
}

for text in Texts:
    response = client.detect_dominant_language(Text=text)
    lang_code = response['Languages'][0]['LanguageCode']

    # Detect key phrase using the detected language code
    key_phrase_response = client.detect_key_phrases(Text=text, LanguageCode=lang_code)
    key_phrase = key_phrase_response['KeyPhrases'][1]['Text']

    print(f"key_phrases: {key_phrase}")

```

Here we just extend from the above code by using the API `client.detect_key_phrases(Text=text, LanguageCode=lang_code)`
where the lang_code is obtained from the response for the detect_dominant_language(). The response for the detect key phrase
is of the format:
{
    'KeyPhrases': [
        {
            'Score': ...,
            'Text': 'string',
            'BeginOffset': 123,
            'EndOffset': 123
        },
    ]
}
So, we use the 'Text' key in the entity of the 'KeyPhases' key to get the key_phrase of our texts.

### Step2
You can run the script using:
```
python3 detect_key_phrases.py
```

This will give you the output in the format "Key Phrase: key_phrase_of_the_text" for the above texts.

**What is a key phrase?**
A key phrase is any portion of text that adds to the meaning or emphasis of a sentence, even if it isn't unique or descriptive.

### Detect syntaxes

### Step 1
Now we create a script using the command: 
```
nano detect_syntax.py
```
Paste the below script and press CTRL+X and Y and ENTER.

Use the following `detect_syntax.py` script to test the above texts and get the syntax of these texts:

```
import boto3

client = boto3.client('comprehend')

# Texts in different languages
Texts = [
    "The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
    "El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote de la Mancha.",
    "Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout reconstruire Pour tout reconstruire Je l'aime à mourir",
    "L'amor che move il sole e l'altre stelle."
]

# Dictionary to map language codes to their abbrevations
lang_dict = {
    'en': 'English',
    'es': 'Spanish',
    'fr': 'French',
    'it': 'Italian'
}

for text in Texts:
    response = client.detect_dominant_language(Text=text)
    lang_code = response['Languages'][0]['LanguageCode']

    # Detect syntax using the detected language code
    syntax_response = client.detect_syntax(Text=text, LanguageCode=lang_code)
    text_phrase = syntax_response['SyntaxTokens'][1]['Text']
    text_part_of_speech = syntax_response['SyntaxTokens'][4]['PartOfSpeech']
    text_tag = text_part_of_speech['Tag']

    print(f"{text_phrase} is : {text_tag}")

```

Here we just extend from the above code by using the API `client.detect_syntax(Text=text, LanguageCode=lang_code)`
where the lang_code is obtained from the response for the detect_dominant_language(). The response for the detect syntax
is of the format:

{
    'SyntaxTokens': [
        {
            'TokenId': 123,
            'Text': 'string',
            'BeginOffset': 123,
            'EndOffset': 123,
            'PartOfSpeech': {
                'Tag': 'ADJ'|'ADP'|'ADV'|'AUX'|'CONJ'|'CCONJ'|'DET'|'INTJ'|'NOUN'|'NUM'|'O'|'PART'|'PRON'|'PROPN'|'PUNCT'|'SCONJ'|'SYM'|'VERB',
                'Score': ...
            }
        },
    ]
}

So, we use the text and the "Tag" for the 'PartOfSpeech' to get the syntax from our texts.

### Step2
You can run the script using:
```
python3 detect_syntax.py
```

This will give you the output in the format "Text phrase is: 'Tag'" for the above texts.

**What are syntaxes?**
Syntax displays the order and arrangement of words in a sentence and their level of grammatical roles.

<img width="917" height="465" alt="image" src="https://github.com/user-attachments/assets/f80ade29-fd78-4aa3-a17d-8145fe6c9d05" />

<img width="905" height="442" alt="image" src="https://github.com/user-attachments/assets/dde18be9-70ce-4b0b-95b6-dc5b9d9ebf37" />


## AWS Rekognition

### Add images

You can use the following script `create_lab9_bucket.py` based on thescript in lab 3 to create an s3 bucket and:

Add an image of an urban setting (named as urban.jpg).

Add an image of a person on the beach (named as beach.jpg).

Add an image with people showing their faces (named as faces.jpg).

Add an image with text (named as text.jpg).

into the bucket:

```
import os
import boto3
import botocore

# --- Configuration ---
STUDENT_ID = '23463452'
REGION = 'ap-northeast-1'   
BUCKET_NAME = f"{STUDENT_ID}-lab9"

# --- Initialize S3 client ---
s3 = boto3.client('s3', region_name=REGION)

# --- Create the S3 bucket ---
s3.create_bucket(
            Bucket=BUCKET_NAME,
            CreateBucketConfiguration={'LocationConstraint': REGION}
        )

print(f"Bucket '{BUCKET_NAME}' created successfully in {REGION} region.")

# --- Upload function ---
def upload_file(file_path, file_name):
    """Upload a local file to the S3 bucket."""
    print(f"Uploading {file_name} → s3://{BUCKET_NAME}/{file_name}")
    s3.upload_file(file_path, BUCKET_NAME, file_name)
    print(f"Uploaded: {file_name}")

# --- List of images to upload ---
images = [
    ('urban.jpg', 'Image of an urban setting'),
    ('beach.jpg', 'Image of a person on the beach'),
    ('faces.jpg', 'Image with people showing their faces'),
    ('text.jpg', 'Image with text')
]

# --- Upload each image ---
for filename, description in images:
    if os.path.exists(filename):
        upload_file(filename, filename)
    else:
        print(f"Missing file: {filename} — please place it in the same directory as this script.")

print("\n All uploads complete.")

```
**Note:** Make sure the directories are placed in the same directory as the script.

This code creates an Amazon S3 bucket called 23463452-lab9 in the ap-northeast-1 (Tokyo) region.
First, it initializes an S3 client using boto3, then creates the bucket using the specified region as the location constraint.

Then the program defines upload_file() method which uploads a local file to the S3 bucket using the s3.upload_file() command.
Then it checks to see if there are four specific image files - urban.jpg, beach.jpg, faces.jpg, and text.jpg - in this same folder as the script,
and if it finds any of these files, it uploads these files to the bucket.

If any image file is missing, the program prints out a message to remind the user to place the file in the same folder as the script.
Once all available files are uploaded the program confirms all uploads are complete.

So, first create this script using the command:
```
nano create_lab9_bucket.py
```
Copy paste the above script and the CTRL+X , Y and then ENTER to save it.

Now run it :

```
python3 create_lab9_bucket.py

```

<img width="1366" height="517" alt="image" src="https://github.com/user-attachments/assets/4a0331e3-238d-4a45-973f-16c65b4b53cc" />
<img width="1531" height="621" alt="image" src="https://github.com/user-attachments/assets/72c0421d-1322-45f9-9f6a-75b66161c3af" />


### Test AWS rekognition



