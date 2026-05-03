# devops-web

Create a aws instance on AWS and flow the steps.

sudo apt-get update

mkdir hello
cd hello

copy the file and paste in the terminal 
------------------------------------------------------------------------------------------------------------------------------------------------
cat > app.py << EOF
from pathlib import Path
from flask import Flask, render_template, request, redirect, url_for, flash
import json

BASE_DIR = Path(__file__).resolve().parent
DATA_DIR = BASE_DIR / "data"

app = Flask(__name__)
app.config["SECRET_KEY"] = "replace-with-a-secure-secret"


def load_json(filename):
    with open(DATA_DIR / filename, encoding="utf-8") as file:
        return json.load(file)


courses = load_json("courses.json")
roadmap = load_json("roadmap.json")
testimonials = load_json("testimonials.json")

contact = {
    "phone": "+91 97982-53860",
    "email": "info@devopsacademy.co",
    "address": "BTM Layout, Bengaluru, Karnataka 560076",
    "website": "www.devopsacademy.co",
}


@app.route("/")
def home():
    return render_template(
        "index.html",
        courses=courses,
        roadmap=roadmap,
        testimonials=testimonials,
        contact=contact,
    )


@app.route("/apply", methods=["POST"])
def apply():
    name = request.form.get("name", "").strip()
    email = request.form.get("email", "").strip()
    phone = request.form.get("phone", "").strip()
    message = request.form.get("message", "").strip()

    if not name or not email:
        flash("Name and email are required. Please complete the form.", "error")
        return redirect(url_for("home") + "#enroll")

    flash("Thank you! Your enrollment request has been received.", "success")
    app.logger.info("Enrollment request: %s, %s, %s, %s", name, email, phone, message)
    return redirect(url_for("home") + "#enroll")


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)

EOF

-----------------------------------------------------------------------------------------------------------------------------------------------
sudo apt install python3-pip

python app.py

--------------------------------------------------------------------------------------------------
git init
git add .
git commit -m "v1"
git push

--------------------------------------------------------------------------------------
create a docker file and requirements.txt file 

cat > requirements.txt << EOF
flask
EOF

---------------------------

cat > dockerfile << EOF
# Use official Python slim image
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    FLASK_ENV=production

# Set working directory
WORKDIR /app

# Install dependencies first (layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application source
COPY app.py .
COPY data/ data/
COPY static/ static/
COPY templates/ templates/

# Expose Flask port
EXPOSE 5000

# Run with gunicorn in production
CMD ["python", "app.py"]

EOF

------------------------------------------------------------
create a image = docker build -t devops:v1
run the container = docker run -it -p 5000-5000

------------------------------------------------------
DockerHub

docker login

docker  logout

docker login 
#verify the account

docker tag devops:v1 <docker-username>/devops:v1

docker push <docker-username>/devops:v1

---------------------------------------------------------------
Jenkinsfile

cat > Jenkinsfile << EOF
pipeline {
  agent any

  triggers {
    githubPush()
  }

  environment {
    IMAGE = "devops-site"
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main',
            url: 'https://github.com/devopsplan2026/devOps-Site.git'
      }
    }

    stage('Build') {
      steps {
        script {
          def VERSION = sh(script: "date +%Y%m%d%H%M", returnStdout: true).trim()
          env.VERSION = VERSION
        }
        sh "docker build -t ${IMAGE}:${env.VERSION} ."
      }
    }

    stage('Login & Push') {
      steps {
        withCredentials([usernamePassword(
              credentialsId: 'DOCKERHUB_CRAD',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh """
            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
            docker tag ${IMAGE}:${env.VERSION} \$DOCKER_USER/${IMAGE}:${env.VERSION}
            docker push \$DOCKER_USER/${IMAGE}:${env.VERSION}
          """
        }
      }
    }

    stage('Deploy') {
      steps {
        sh """
          docker stop ${IMAGE} || true
          docker rm   ${IMAGE} || true
          docker run -d \
            --name ${IMAGE} \
            --restart unless-stopped \
            -p 5000:5000 \
            \$DOCKER_USER/${IMAGE}:${env.VERSION}
        """
      }
    }

  }

  post {
    success {
      echo "Build ${env.VERSION} deployed successfully."
    }
    failure {
      echo "Pipeline failed. Check the logs above."
    }
    always {
      sh "docker logout || true"
    }
  }
}

EOF 

Now go to jenkins >> go to settings >> cradentials >> username and password >> enter the docker hub username and tocken id name is according to jenkins pipeline

Now create a job with pipeline >> trigger webhook >> pipeline (SCM) 

go to repo setting >> webhook >> <jenkins_url>/github-webhook

----------------------------------------------------------------------------------------------------
eks with terraform

cat > main.tf << EOF
terraform {
  required_version = ">= 1.3.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

###############################################################
# VPC
###############################################################
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.8"

  name = "devops-site-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = {
    Terraform   = "true"
    Environment = "dev"
    Project     = "devops-site"
  }

  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
}

###############################################################
# EKS Cluster
###############################################################
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "devops-site-cluster"
  cluster_version = "1.30"

  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
    }
    eks-pod-identity-agent = {
      most_recent = true
    }
  }

  cluster_endpoint_public_access           = true
  enable_cluster_creator_admin_permissions = true

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  ###############################################################
  # Node Groups
  ###############################################################
  eks_managed_node_groups = {
    default = {
      instance_types = ["t3.medium"]
      min_size       = 1
      max_size       = 3
      desired_size   = 2

      labels = {
        Environment = "dev"
        Project     = "devops-site"
      }

      tags = {
        Terraform   = "true"
        Environment = "dev"
        Project     = "devops-site"
      }
    }
  }

  tags = {
    Environment = "dev"
    Terraform   = "true"
    Project     = "devops-site"
  }
}

###############################################################
# Outputs
###############################################################
output "cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "cluster_endpoint" {
  description = "EKS cluster API endpoint"
  value       = module.eks.cluster_endpoint
}

output "cluster_certificate_authority_data" {
  description = "Base64 encoded certificate data for the cluster"
  value       = module.eks.cluster_certificate_authority_data
  sensitive   = true
}

output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "private_subnets" {
  description = "Private subnet IDs"
  value       = module.vpc.private_subnets
}

output "configure_kubectl" {
  description = "Command to configure kubectl for this cluster"
  value       = "aws eks update-kubeconfig --region ap-south-1 --name ${module.eks.cluster_name}"
}
EOF

create a IAM role with VPCFULLACCESS, ADMINSTARTOR, EKS_CLUSTER

Create access key of role.

download aws_cli, kubeclt, eksctl, terraform

now connect terminal with aws 
aws config
enter the access key and enter just

now 
terrform init
terraform plan
terraform apply

--------------------------------------------------------
aws eks update-kubeconfig --region ap-south-1 --name devops-site-cluster

# Verify nodes are ready
kubectl get nodes

--------------------------------------------------------
jenkinsfile update 

stage('Update Manifest') {
  steps {
    withCredentials([usernamePassword(
      credentialsId: 'GITHUB_CRED',
      usernameVariable: 'GIT_USER',
      passwordVariable: 'GIT_PASS'
    )]) {
      sh """
        sed -i 's|image:.*|image: \$DOCKER_USER/${IMAGE}:${env.VERSION}|' k8s-manifest.yaml
        git config user.email "jenkins@devopsacademy.co"
        git config user.name  "Jenkins"
        git add k8s-manifest.yaml
        git commit -m "ci: update image to ${env.VERSION} [skip ci]"
        git push https://\$GIT_USER:\$GIT_PASS@github.com/devopsplan2026/devOps-Site.git main
      """
    }
  }
}


