#!groovy
def gitRepo = 'https://github.com/Nikita026/sample-java-app.git'
def gitCredentialsId = "Git"
def gitBranch = "main"
pipeline {
    agent any
    stages{
        stage('Deploy EKS Cluster') {
            steps{
                script{
                    try{
                        cleanWs()
                        git ( branch: 'main', credentialsId: 'Git', url: 'https://github.com/Nikita026/sample-java-app.git' )
                        sh '''
                            cd eks-terraform
                            terraform init
                            terraform apply -auto-approve

                            region=$(terraform output -raw region)
                            cluster_name=$(terraform output -raw cluster_name)
                            vpc_id=$(terraform output -raw vpc_id)

                            aws eks --region $region update-kubeconfig \
                                --name $cluster_name

                            curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json
                            aws iam create-policy \
                                --policy-name AWSLoadBalancerControllerIAMPolicy \
                                --policy-document file://iam_policy.json

                            eksctl create iamserviceaccount \
                            --cluster=$cluster_name \
                            --namespace=kube-system \
                            --name=aws-load-balancer-controller \
                            --role-name "AmazonEKSLoadBalancerControllerRole" \
                            --region $region \
                            --attach-policy-arn=arn:aws:iam::952890408605:policy/AWSLoadBalancerControllerIAMPolicy \
                            --approve

                            helm repo add eks https://aws.github.io/eks-charts
                            helm repo update
                            helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
                            -n kube-system \
                            --set clusterName=$cluster_name \
                            --set serviceAccount.create=false \
                            --set serviceAccount.name=aws-load-balancer-controller \
                            --set region=$region \
                            --set vpcId=$vpc_id \
                            --set image.repository=602401143452.dkr.ecr.$region.amazonaws.com/amazon/aws-load-balancer-controller

                            '''
                        stash 'workspace'
                    }
                    catch (err) {
                        echo "Failed: ${err}"
                        CI_ERROR = "Failed in deploying EKS Cluster: "+err.toString()
                        error(${CI_ERROR})
                    }
                }
            }
        }
        stage('Deploy Java App') {
            steps{
                script{
                    try{
                        unstash 'workspace'
                        sh 'cd $WORKSPACE && ansible-playbook playbook.yaml'
                    }
                    catch (err) {
                        echo "Failed: ${err}"
                        CI_ERROR = "Failed in App Deploy Stage: "+err.toString()
                        error(${CI_ERROR})
                    }
                }
            }
        }
    }
}
