node{
    stage('git checkout') {
        git branch: 'main', url: 'https://github.com/Shivendra73/kubernetes_jenkins_project.git'
        
    }
    stage('sending docker file to ansible server using sshagent'){
        sshagent(['ansible_server']) {
         sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89'
         sh 'scp /var/lib/jenkins/workspace/pipeline_demo/Dockerfile ubuntu@172.31.46.89:/home/ubuntu'
        }
    }
    stage('building docker image'){
        sshagent(['ansible_server']) {
        sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89'
        sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89 sudo docker image build -t $JOB_NAME:v1.$BUILD_ID -f /home/ubuntu/Dockerfile .'

        }
    
    }
    stage("Docker Image Tagging"){
          sshagent(['ansible_server']) {
            sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89 cd /home/ubuntu'
            sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89 sudo docker image tag $JOB_NAME:v1.$BUILD_ID shivendrapatel/$JOB_NAME:v1.$BUILD_ID'
            sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89 sudo docker image tag $JOB_NAME:v1.$BUILD_ID shivendrapatel/$JOB_NAME:latest'

        }
            
    }
    stage('Pushing Docker images from asible serer to docker hub'){
        sshagent(['ansible_server']) {
        withCredentials([string(credentialsId: 'dockerhub_access', variable: 'dockerhub_access')])  {
            sh "ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89 sudo docker login -u shivendrapatel -p $dockerhub_access "
            sh "ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89 sudo docker image push shivendrapatel/$JOB_NAME:v1.$BUILD_ID "
            sh "ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89 sudo docker image push shivendrapatel/$JOB_NAME:latest "

          }
       }
    }  
    stage ('sending Deployment and service file to kubernetes server'){
        sshagent(['kubernetes_login']) {
           sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.80.138'
           sh 'scp /var/lib/jenkins/workspace/pipeline_demo/Deployment.yaml ubuntu@172.31.80.138:/home/ubuntu'
           sh 'scp /var/lib/jenkins/workspace/pipeline_demo/Service.yaml ubuntu@172.31.80.138:/home/ubuntu'
           
            
        } 
        
    }
    stage ('sending the ansible playbook to ansible server'){
         sshagent(['ansible_server']) {
             sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89 cd /home/ubuntu'
             sh 'scp /var/lib/jenkins/workspace/pipeline_demo/ansible.yaml ubuntu@172.31.46.89:/home/ubuntu'
             
         }
    }
    stage('kubernetes deployment using ansible playbook '){
        sshagent(['ansible_server']){
            sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89 cd /home/ubuntu'
            sh 'ssh -o StrictHostKeyChecking=no ubuntu@172.31.46.89  ansible-playbook ansible.yaml'
        }
            
        }
    

}    