def set_up_buildx(){
    sh "docker buildx create --name builder --use --platform linux/amd64 --node builder0"
    sh "docker buildx inspect builder --bootstrap"
}

def build_step(){
    withCredentials([[
    $class: 'AmazonWebServicesCredentialsBinding',
    credentialsId: "aws-credentials",
    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
    ]]){
    sh """
     export AWS_DEFAULT_REGION=us-east-1
     aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/z1l0c6l7 && \
     make publish-buildx
    """
    }
}
pipeline {
    agent any
    options {
        skipStagesAfterUnstable()
        timestamps()
    }
    environment {
        cloneDir = 'app-config'
        imageName = 'public.ecr.aws/z1l0c6l7/simpleapp'
        scmInfo = checkout scm
        gitCommit = "${scmInfo.GIT_COMMIT}"
        ENV = "head"
        deployRepoUrl = "git@github.com:Faithtosin/argocd-apps.git"
        BUILDX_ENABLED = set_up_buildx()
    }
    stages {
         stage('Clone repository') { 
            steps { 
                script{
                checkout scm
                sh "docker run --rm --privileged multiarch/qemu-user-static --reset -p yes"
                }
            }
        }
        stage('test library') { 
            steps { 
                script{HelloWorld()}
            }
        }
        stage('Build Docker file') {
            parallel {
                stage('Build-1'){
                    steps {
                        script {build_step()}
                    }
                }
                stage('Build-2'){
                    steps {
                        script {build_step()}
                    }
                }
            }
        }
        stage('Deploy'){
            steps {
                withCredentials([file(credentialsId: 'GIT_DEPLOY_SSH', variable: 'GIT_DEPLOY_KEY')]) {
                    sh """
                    mkdir /var/lib/jenkins/.ssh/ && cp \$GIT_DEPLOY_KEY /var/lib/jenkins/.ssh/id_rsa && chmod 400 /var/lib/jenkins/.ssh/id_rsa
                    rm -rf ${cloneDir}
                    git clone ${deployRepoUrl} ${cloneDir}
                    cd ${cloneDir}
                    
                    export cloneDirFullPath=`pwd`
                    cd simpleapp-public/overlays/${ENV}
                    kustomize edit set image ${imageName}:${gitCommit}
                    cd \$cloneDirFullPath
                    ls -la
                    ./update-image.sh ${ENV} ${imageName} ${gitCommit}
                    """
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
