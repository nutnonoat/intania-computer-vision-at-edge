# Intania Computer Vision at The Edge Workshop Instructions



## 1. Provision EC2 with VS Code Server using CloudFormation Template (in Default VPC only)

* Make sure to use AWS Region `eu-central-1`
* Create a Key Pair
    * name → `ws-default-keypair`
    * type → `RSA`
    * format → `.pem`
* [Launch CloudFormation stack in eu-central-1](https://console.aws.amazon.com/cloudformation/home?region=eu-central-1#/stacks/create/review?templateURL=https://ws-assets-prod-iad-r-fra-b129423e91500967.s3.eu-central-1.amazonaws.com/5ecc2416-f956-4273-b729-d0d30556013f/vscs-with-cloudfront.yaml)
    * Parameters
        * Stack Name → `VSCodeServerStack`
        * VPC → Select default (The only available)
        * Subnet → `172.31.0.0/20`
        * Leave the rest as default
        * Check `Acknowledgement`
    * Click `Create Stack`
    * Wait until Stack creation completed
    * Get the `VS Code URL` and `Password` from the Stack Output
    * Verify if you can log on the VS Code

* * *

## 2. Install Greengrass Core on the EC2

### Installation

* In VS Code, open Terminal
* Set Environment Variables

```
export AWS_ACCESS_KEY_ID=<Your access key ID>
export AWS_SECRET_ACCESS_KEY=<Your secret access key>
export AWS_SESSION_TOKEN=<Your session token>
export AWS_DEFAULT_REGION=eu-central-1
export THING_NAME=cv-at-edge-ec2-gg-core
export GROUP_NAME=cv-at-edge-group
export ROLE_NAME=CVLabGreengrassV2Role
export ROLE_ALIAS=CVLabGreengrassCoreTokenExchangeRoleAlias
```

* Ensure Java is installed

```
java --version
```

* Change current directory to home

```
cd ~
```

* Download and unzip Greengrass Core

```
curl -s https://d2s8p88vqu9w66.cloudfront.net/releases/greengrass-nucleus-latest.zip \
> greengrass-nucleus-latest.zip

unzip greengrass-nucleus-latest.zip -d GreengrassCore && \
rm greengrass-nucleus-latest.zip
```

* Run Greengrass Core installer

```
sudo -E java -Dlog.store=FILE \
  -jar ./GreengrassCore/lib/Greengrass.jar \
  --aws-region $AWS_DEFAULT_REGION \
  --root /greengrass/v2 \
  --thing-name $THING_NAME \
  --thing-group-name $GROUP_NAME \
  --tes-role-name $ROLE_NAME \
  --tes-role-alias-name $ROLE_ALIAS \
  --component-default-user ggc_user\:ggc_group \
  --provision true \
  --deploy-dev-tools true \
  --setup-system-service true

sudo chmod 755 /greengrass/v2 && sudo chmod 755 /greengrass
```



### **Validate Greengrass Core**

* Install Greengrass Quick Development Tool

```
cd ~
curl https://raw.githubusercontent.com/aws-greengrass/aws-greengrass-quick-templates/main/install.sh  | sudo bash
```

* Deploy Hello World Component locally

```
cd ~
echo 'print("Hello from Python!")'>hello.py
ggq hello.py
```

* Validate Output

```
sudo cat /greengrass/v2/logs/hello.log
```

Output should look like

```
2021-06-22T14:44:57.324Z [INFO] (pool-2-thread-18) hello: shell-runner-start. {scriptName=services.hello.lifecycle.install, serviceName=hello, currentState=NEW, command=["ln -f -s -t . /greengrass/v2/packages/artifacts-unarchived/hello/0.0.0/A41E885..."]}
2021-06-22T14:44:57.333Z [INFO] (pool-2-thread-18) hello: shell-runner-start. {scriptName=services.hello.lifecycle.run, serviceName=hello, currentState=STARTING, command=["python3 hello.py"]}
2021-06-22T14:44:57.368Z [INFO] (Copier) hello: stdout. Hello from Python!. {scriptName=services.hello.lifecycle.run, serviceName=hello, currentState=RUNNING}
2021-06-22T14:44:57.374Z [INFO] (Copier) hello: Run script exited. {exitCode=0, serviceName=hello, currentState=RUNNING}
```


* * *

## 3. Deploy Sample Data Component

### Preparation

* Create S3 bucket

```
export region=$AWS_DEFAULT_REGION
export acct_num=$(aws sts get-caller-identity --query "Account" --output text)
export bucket_name=greengrass-component-artifacts-$acct_num-$region
aws s3 mb s3://$bucket_name
```

* Install Jq and Zip

```
sudo apt install zip
sudo apt install jq
```

* Update IAM Policy used by Greengrass Core to allow read access on the S3 bucket

```
mkdir -p ~/scripts
cat << EOF > ~/scripts/s3_policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    $(aws iam get-policy-version --policy-arn arn:aws:iam::$acct_num:policy/CVLabGreengrassV2RoleAccess --version-id v1 | jq '.PolicyVersion.Document.Statement[0]'),
    {
      "Sid": "AccessArtifactsInS3",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::$bucket_name/*"
      ]
    }
  ]
}
EOF

aws iam create-policy-version \
    --policy-arn arn:aws:iam::$acct_num:policy/CVLabGreengrassV2RoleAccess \
    --policy-document file:///home/ubuntu/scripts/s3_policy.json \
    --set-as-default
```



### **Create and Deploy Greengrass Component**

* Download sample data

```
export component_name=com.example.sample_data
export component_version=1.0.0
mkdir -p ~/GreengrassCore/artifacts/$component_name/$component_version && cd $_
wget http://personal.ie.cuhk.edu.hk/~ccloy/files/datasets/mall_dataset.zip
unzip mall_dataset.zip && rm mall_dataset.zip
mv mall_dataset/frames ./ && rm -rf mall_dataset
rm frames/Thumbs.db
```

* Prepare artifacts

```
cd ~/GreengrassCore/artifacts/$component_name/$component_version/
zip -m $component_name.zip frames/* && rm -rf frames
```

* Upload to S3

```
aws s3 sync ~/GreengrassCore/ s3://$bucket_name/ --delete
```

* Create Recipe for the component

```
mkdir -p ~/GreengrassCore/recipes/
cat << EOF > ~/GreengrassCore/recipes/$component_name-$component_version.json
{
  "RecipeFormatVersion": "2020-01-25",
  "ComponentName": "$component_name",
  "ComponentVersion": "$component_version",
  "ComponentDescription": "A component that installs sample frames.",
  "ComponentPublisher": "Amazon",
  "Manifests": [
    {
      "Platform": {
        "os": "linux"
      },
      "Lifecycle": {},
      "Artifacts": [
        {
          "URI": "s3://$bucket_name/artifacts/$component_name/$component_version/$component_name.zip",
          "Unarchive": "ZIP",
          "Permission": {
            "Read": "ALL",
            "Execute": "NONE"
          }
        }
      ]
    }
  ]
}
EOF
```

* Create the Greengrass component

```
aws greengrassv2 create-component-version --inline-recipe fileb://~/GreengrassCore/recipes/$component_name-$component_version.json
```

* Deploy the Sample Image Component to the existing Deployment Group
    * Go to [Greengrass Console](https://eu-central-1.console.aws.amazon.com/iot/home?region=eu-central-1#/greengrass/v2/components)
    * Click the component `com.example.sample_data`  then `Deploy`
    * Select `Add to existing deployment`
    * Select `Deployment for cv-at-edge-group` Deployment and `Next`
    * Click `Skip to Review `and ensure the component is in the list
    * Click `Deploy`
    * Wait until the deployment status is `Succeeded`



* Verify if the deployment on the Target Device (EC2), we should see `2000` after running below.

```
export gg_home=/greengrass/v2
find $gg_home/packages/artifacts-unarchived/$component_name/$component_version/$component_name/frames/*.jpg | xargs ls -lrt | wc -l
```


* * *

## 4. Deploy Acquisition Component (GStreamer)

### Preparation

* Verify if Docker is installed on Target Device

```
docker --version
```

* Add Greengrass user to the Docker group

```
export gg_user_name=ggc_user
export gg_group=ggc_group
sudo usermod -aG docker $gg_user_name
```

* Create output directory and assign Greengrass user as the owner

```
mkdir -p /tmp/data
export data_dir=/tmp/data
sudo mkdir -p $data_dir
sudo chown -R $gg_user_name:$gg_group $data_dir
```

### Create and Deploy Greengrass component

* Clone the component repository

```
cd ~
git clone https://github.com/aws-samples/aws-iot-greengrass-component-gstreamer-frame-grabber.git
cd ~/aws-iot-greengrass-component-gstreamer-frame-grabber
```

* Modify timezone in the Dockerfile

```
sed -i 's/<your timezone, e.g. America\/Los_Angeles>/Asia\/Bangkok/g' Dockerfile
```

* Build docker image with fakesrc and fakesink

```
export container_name=gst
docker build --rm -t $container_name .
```

* Test the docker image

```
docker run -v $data_dir:/data --name=$container_name $container_name
```

You will see output as below

```
Factory Details:
  Rank                     none (0)
  Long-name                Fake Sink
  Klass                    Sink
  Description              Black hole for data
  ...
```

* Archive Docker image

```
export component_name=com.example.gst-grabber
export component_version=1.0.0

mkdir -p ~/GreengrassCore && cd $_
mkdir -p ~/GreengrassCore/artifacts/$component_name/$component_version

docker save $container_name > ~/GreengrassCore/artifacts/$component_name/$component_version/$container_name.tar
```

* Compress Docker image and upload to S3 bucket

```
export region=$AWS_DEFAULT_REGION
export acct_num=$(aws sts get-caller-identity --query "Account" --output text)
export bucket_name=greengrass-component-artifacts-$acct_num-$region

gzip ~/GreengrassCore/artifacts/$component_name/$component_version/$container_name.tar
aws s3 sync ~/GreengrassCore/ s3://$bucket_name/ --delete
```

* Create recipe for the component

```
export arch=$(uname -m)
export sample_data_component_name=com.example.sample_data
export sample_data_component_version=1.0.0

mkdir -p ~/GreengrassCore/recipes/
touch ~/GreengrassCore/recipes/$component_name-$component_version.json
cat << EOF > ~/GreengrassCore/recipes/$component_name-$component_version.json
{
  "RecipeFormatVersion": "2020-01-25",
  "ComponentName": "$component_name",
  "ComponentVersion": "$component_version",
  "ComponentDescription": "A component that runs a Docker container from an image in an S3 bucket.",
  "ComponentPublisher": "Amazon",
  "ComponentConfiguration": {
      "DefaultConfiguration": {
          "mounts": "-v /tmp/data:/data -v /greengrass/v2/packages/artifacts-unarchived/$sample_data_component_name/$sample_data_component_version/$sample_data_component_name/frames:/frames",
          "entrypoint": "gst-launch-1.0",
          "command": "multifilesrc location=\"/frames/seq_%06d.jpg\" index=1 loop=true caps=\"image/png,framerate=\\\(fraction\\\)12/1\\" ! multifilesink location=\"/data/frame.jpg\""
      }
  },
  "Manifests": [
    {
      "Platform": {
        "os": "linux",
        "architecture": "$arch"
      },
      "Lifecycle": {
        "Install": {
          "Script": "mkdir -p /tmp/data; docker load -i {artifacts:path}/$container_name.tar.gz"
        },
        "Startup": {
          "Script": "docker run --user=\$(id -u):\$(id -g) --rm -d {configuration:/mounts} --name=$container_name --entrypoint {configuration:/entrypoint} $container_name {configuration:/command}"
        },
        "Shutdown": {
          "Script": "docker stop $container_name"
        }
      },
      "Artifacts": [
        {
          "URI": "s3://$bucket_name/artifacts/$component_name/$component_version/$container_name.tar.gz"
        }
      ]
    }
  ]
}
EOF
```

* Create Greengrass Component

```
aws greengrassv2 create-component-version \
  --inline-recipe fileb://~/GreengrassCore/recipes/$component_name-$component_version.json
```

* Clean up docker image and container

```
docker system prune
docker image rm $container_name:latest 
```

* Deploy the GStreamer Component to the existing Deployment Group
    * Go to [Greengrass Console](https://eu-central-1.console.aws.amazon.com/iot/home?region=eu-central-1#/greengrass/v2/components)
    * Click the component `com.example.gst-grabber`  then `Deploy`
    * Select `Add to existing deployment`
    * Select `Deployment for cv-at-edge-group` Deployment and `Next`
    * Click `Skip to Review `and ensure the component is in the list
    * Click `Deploy`
    * Wait until the deployment status is `Succeeded`



* Verify result
    * Run `docker container ls` → will see container `gst` started and running
    * Run `ls /tmp/data -la` → will see frame.jpg kept overwritten by `gst`



### [Optional - For Troubleshooting] Test Run GStreamer Locally

* Start Container manually on the Core Device

```
docker run -v /tmp/data:/data -v $frame_home:/frames -it --entrypoint /bin/bash $container_name
gst-launch-1.0 multifilesrc location="/frames/seq_%06d.jpg" index=1 loop=true caps="image/png,framerate=\(fraction\)12/1" ! multifilesink location="/data/frame.jpg"
```

* Remove the container after finish testing

```
docker image rm $container_name:latest 
docker system prune
```


* * *

## 5. Deploy Inference Components (People Counter)

### Preparation

* Create input and output directory and grant Greengrass user the owner

```
# Input directory
sudo mkdir -p /tmp/data
sudo chown -R ggc_user:ggc_group /tmp/data

# Output directory
sudo mkdir -p /tmp/out
sudo chown -R ggc_user:ggc_group /tmp/out
```



### Create Inference component

* **Create environment variables**

```
export component_name=com.example.count_people
export component_version=1.0.0
```

* **Download and Package People Counter script**

```
# Clone People Counter component repo
cd ~
git clone https://github.com/aws-samples/aws-iot-greengrass-component-gluoncv-people-counter.git

mkdir -p ~/GreengrassCore/artifacts/$component_name/$component_version
cp ~/aws-iot-greengrass-component-gluoncv-people-counter/artifacts/count_people/* ~/GreengrassCore/artifacts/$component_name/$component_version/
cd ~/GreengrassCore/artifacts/$component_name/$component_version/

zip -m $component_name.zip *
```

* **Upload file to S3**

```
export region=$AWS_DEFAULT_REGION
export acct_num=$(aws sts get-caller-identity --query "Account" --output text)
export bucket_name=greengrass-component-artifacts-$acct_num-$region

aws s3 sync ~/GreengrassCore/ s3://$bucket_name/ --delete
```

* **Create Recipe file**

```
mkdir -p ~/GreengrassCore/recipes/
cat << EOF > ~/GreengrassCore/recipes/$component_name-$component_version.json
{
  "RecipeFormatVersion": "2020-01-25",
  "ComponentName": "$component_name",
  "ComponentVersion": "$component_version",
  "ComponentDescription": "A component that runs an inference model to count people.",
  "ComponentPublisher": "Amazon",
  "ComponentConfiguration": {
    "DefaultConfiguration": {
      "accessControl": {
        "aws.greengrass.ipc.mqttproxy": {
          "com.example.Pub:publisher:1" :{
            "policyDescription": "publish results to topic",
            "operations": [
              "aws.greengrass#PublishToIoTCore"
            ],
            "resources": [
              "demo/topic"
            ]
          }
        }
      },
      "ModelName": "ssd_512_resnet50_v1_voc",
      "ClassName": "person",
      "Threshold": "0.75",
      "SourceFile": "/tmp/data/frame.jpg",
      "FrameRate": "0.1",
      "Topic": "demo/topic",
      "Output": "/tmp/out/frame%04d.jpg",
      "Num_Outputs": 1000
    }
  },
  "Manifests": [
    {
      "Platform": {
        "os": "linux",
        "architecture": "aarch64"
      },
      "Lifecycle": {
        "Install": {
          "Script": "/bin/bash {artifacts:decompressedPath}/$component_name/install.sh {configuration:/Output}",
          "timeout": "900"
        },          
        "Run": {
          "Script": "/bin/bash {artifacts:decompressedPath}/$component_name/infer.sh -m {configuration:/ModelName} -c {configuration:/ClassName} -z {configuration:/Threshold} -s {configuration:/SourceFile} -r {configuration:/FrameRate} -t {configuration:/Topic} -o {configuration:/Output} -n {configuration:/Num_Outputs}"
        }
      },
      "Artifacts": [
        {
          "URI": "s3://$bucket_name/artifacts/$component_name/$component_version/$component_name.zip",
          "Unarchive": "ZIP"
        }
      ]
    }
  ]
}
EOF

```

* Install Arm Performance Library (reference link: https://learn.arm.com/install-guides/armpl/)

```
cd ~
wget https://developer.arm.com/-/cdn-downloads/permalink/Arm-Performance-Libraries/Version_24.10/arm-performance-libraries_24.10_deb_gcc.tar
tar -xf arm-performance-libraries_24.10_deb_gcc.tar
cd arm-performance-libraries_24.10_deb/
sudo ./arm-performance-libraries_24.10_deb.sh --accept
```

* Update the linker cache

```
sudo ldconfig /opt/arm/armpl_24.10_gcc/lib
```

* Verify if `libarmpl_lp64_mp.so` has been added

```
ldconfig -p | grep -i libarmpl_lp64_mp.so
```

* Clean up unused files

```
cd ~
rm -rf arm-performance-libraries_24.10_deb arm-performance-libraries_24.10_deb_gcc.tar
```

* Create Greengrass component

```
aws greengrassv2 create-component-version --inline-recipe fileb://~/GreengrassCore/recipes/$component_name-$component_version.json
```

* Deploy the People Counter Component to the existing Deployment Group
    * Go to [Greengrass Console](https://eu-central-1.console.aws.amazon.com/iot/home?region=eu-central-1#/greengrass/v2/components)
    * Click the component `com.example.count_people`  then `Deploy`
    * Select `Add to existing deployment`
    * Select `Deployment for cv-at-edge-group` Deployment and `Next`
    * Click `Skip to Review `and ensure the component is in the list
    * Click `Deploy`
    * Wait until the deployment status is `Succeeded`



* Verify result
    * Run `ls /tmp/out -la` → will see frame images continuously created with people overlay. You may also open the directory to view the images.
    * In [AWS IoT Console](https://eu-central-1.console.aws.amazon.com/iot/home?region=eu-central-1#/test), put # in the Topic filter to subscribe to any topic. You will also see People Overlay data published successfully to demo/topic

* * *

