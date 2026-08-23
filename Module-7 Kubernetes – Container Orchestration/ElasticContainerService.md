Docker Hub had always been my go-to registry whenever I worked with Docker.

But this time, while deploying a Weather Application on AWS, I thought — **why not try Amazon ECR?**

Since I was already deploying the application on ECS with Fargate, keeping the container image within AWS felt like a good opportunity to explore ECR, especially with its integration with ECS and AWS IAM.

So, I decided to give it a try.

**Source Code → EC2 → Docker → ECR → ECS Fargate**

Here’s how the journey went.

### 1. Launch and Configure EC2

After launching and connecting to the Ubuntu EC2 instance, install and verify the tools required to build and deploy the application.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/phnxmu0l7vjjvwko9b0i.png)

#### Install Docker

```bash
sudo apt install docker.io -y
```

Installs Docker on the Ubuntu EC2 instance.

#### Verify Docker Installation

```bash
docker --version
```

Checks whether Docker is installed and displays the installed version.

#### Check Running Containers

```bash
docker ps
```

Lists the currently running Docker containers.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/0e8s8dn6vof6cu5ciy4z.png)

#### Give the Current User Docker Permissions

```bash
sudo usermod -aG docker $USER && newgrp docker
```

Adds the current user to the Docker group and applies the new group membership immediately, allowing Docker commands to run without `sudo`.

#### Verify Docker Permissions

```bash
docker ps
```
![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/nimyd46jen6vpnep4s4b.png)

Confirms that the current user can run Docker commands without requiring `sudo`.

#### Install AWS CLI

```bash
sudo apt install awscli
```

Installs the AWS Command Line Interface for interacting with AWS services from the EC2 instance.

#### Verify AWS CLI Installation

```bash
aws --version
```

Checks whether the AWS CLI is installed correctly and displays its installed version.



### 2. Pull and Dockerize the Weather Application

After configuring the EC2 instance, pull the application source code from GitHub and prepare it for containerization.

#### Clone the Repository

Clone the application repository onto the EC2 instance:

```bash
git clone https://github.com/omkarsharma2821/Weather-App-for-AWS.git
```

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/1r1x41ck1wkwpdp5f9dj.png)

Downloads the application source code from GitHub to the EC2 instance.

#### Navigate to the Application Directory

```bash
cd Weather-App-for-AWS/
```

Moves into the application directory where the source code and Dockerfile are located.

#### Inspect the Application

```bash
ls
```

Lists the application files and directories to verify that the repository was cloned successfully.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/34j6wkoiuw1vi4s8rt6b.png)

#### Build the Docker Image

```bash
docker build --build-arg VITE_API_KEY="YOUR_API_KEY" -t weather-app .
```

The command `docker build --build-arg VITE_API_KEY="YOUR_API_KEY" -t weather-app .` builds a Docker image using the `Dockerfile` in the current directory (`.`), while `--build-arg VITE_API_KEY="YOUR_API_KEY"` passes the `VITE_API_KEY` value to the Docker build process so that the Vite application can use it during `npm run build`; `-t weather-app` gives the resulting Docker image the name `weather-app`, which can then be used to run, tag, and push the image to Amazon ECR.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/640lfhz3itnzwf9zoglg.png)

#### Verify the Docker Image

```bash
docker images
```

Lists the available Docker images and confirms that the `weather-app` image was created successfully.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/ree2rmem4jbh2id4km96.png)


#### Run the Container

```bash
docker run -d -p 3000:80--name weather-app weather-app
```

Starts the weather application as a background container and maps the application's port to the EC2 instance.

> **Note:** Replace `3000:80` with the port used by your application.

#### Verify the Running Container

```bash
docker ps
```

Confirms that the weather application container is running successfully.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/u7mubkd0yweux6b84yvf.png)


#### Test the Application

The application can now be tested using the EC2 instance's public IP and the exposed port:

```text
http://<EC2-PUBLIC-IP>:3000
```

This confirms that the application works correctly inside the Docker container before pushing the image to Amazon ECR.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/9vu3ilgn4pfxfv7ug5yz.png)

### 3. Create an ECR Repository and Push the Docker Image

Once the Docker image has been built and tested on the EC2 instance, the next step is to store the image in **Amazon Elastic Container Registry (ECR)**. ECR provides a private container registry where the Docker image can be securely stored and later pulled by Amazon ECS.

#### Create the ECR Repository

Create a private ECR repository from the AWS Management Console.

Navigate to:

**AWS Console → ECR → Repositories → Create repository**

Give the repository a name, for example:

```text
weather-app
```

Keep the repository private and create it.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/rt3yhakrrh3jfartpgtv.png)


#### Authenticate Docker with Amazon ECR

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/qq9z6vgpu92iyx7g5tsy.png)

> **Note:** Ensure the IAM user/role has ECR permissions, especially `ecr:GetAuthorizationToken`, before authenticating Docker with ECR.

This creates an ECR-compatible tag for the local `weather-app` image.

#### Push the Image to ECR

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/sd5hr2prvvre7kw5z150.png)

#### Verify the Image

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/2t6k3urj9sym9zvqx7dk.png)

At this point, the application image is available in ECR and ready to be deployed through **Amazon ECS with AWS Fargate**.


### 4. Create ECS Cluster and Deploy Using Fargate

Once the Docker image is available in Amazon ECR, the next step is to deploy the containerized weather application using **Amazon ECS with AWS Fargate**.

#### Create an ECS Cluster

Navigate to:

**AWS Console → ECS → Clusters → Create Cluster**

Create a cluster and select the **AWS Fargate** infrastructure option.

Give the cluster a suitable name, for example:

```text
weather-app-cluster
```

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/v8j1jogvcsxouj2kf2q2.png)


An ECS cluster provides a logical grouping of ECS services and tasks.

#### Why Fargate?

AWS Fargate is a serverless compute engine for containers. It allows ECS to run containers without requiring you to provision or manage EC2 instances for the container workloads.

In this deployment:

```text
ECS = Container orchestration
Fargate = Compute used to run the containers
```

ECS manages the tasks and services, while Fargate provides the underlying compute capacity.

#### Create an ECS Task Definition

Navigate to:

**ECS → Task Definitions → Create new task definition**

Configure the task definition with the required application settings.

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/dyk1bgmr3auanb8w78nl.png)

#### Create an ECS Service

After creating the task definition, create an ECS service inside the previously created cluster.

Navigate to:

**ECS → Clusters → weather-app-cluster → Create Service**

Select:

```text
Launch type: Fargate
Task definition: weather-app
Desired tasks: 1
```

The desired task count determines how many copies of the container ECS should keep running.

#### Configure Networking

Select the VPC and subnets where the Fargate task will run.

Configure:

* **VPC:** Application VPC
* **Subnets:** Required subnets
* **Security Group:** Application security group
* **Public IP:** Enable if the application needs to be directly accessible from the internet

The security group should allow inbound traffic on the application's required port.

For example:

```text
Type: Custom TCP
Port: 3000
Source: Required traffic source
```

For a production architecture, the application would typically be placed in private subnets and exposed through an **Application Load Balancer (ALB)** rather than assigning a public IP directly to the task.

#### Deploy the Service

Review the configuration and create the ECS service.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/57vsdas3a2hi1n196jmg.png)

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/sg1c8w043vmpewvnjy2z.png)


ECS will then:

1. Pull the Docker image from ECR.
2. Provision the required Fargate compute.
3. Start the container.
4. Attach the configured networking and security group.
5. Keep the desired number of tasks running.

Once the task reaches the **RUNNING** state, the weather application is successfully deployed on ECS using Fargate.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/31al2fm25zp3aq4ik29k.png)

---

✍️ **Author**: *Omkar Sharma*  
📬 *Feel free to connect on [LinkedIn](https://www.linkedin.com/in/omkarsharmaa/) or explore more on [GitHub](https://github.com/omkarsharma2821)*