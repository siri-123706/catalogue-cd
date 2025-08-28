pipeline {
     // agent any 
    agent {
        label 'AGENT-1'
    }
    environment {
        appVersion = ''
        REGION = "us-east-1"
        ACC_ID = "203981192510"
        PROJECT = "roboshop"
        COMPONENT = "catalogue"
        //COURSE = 'jenkins'
    }

    // }
    options { // pipeline expries 30 mint
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds() // not parallel to pipelines at a time so, one complted after another complted
    }
    parameters {
        string(name: 'appVersion',  description: 'image version of the application')
        choice(name: 'deploy_to', choices: ['dev', 'qa', 'prod'], description: 'Pick the Environment')
        
    } 
   // build
   stages {
        stage('deploy') {
            steps {
                script {
                    withAWS(credentials: 'aws-cli', region: 'us-east-1') {
                        sh """
                           aws eks update-kubeconfig --region $REGION --name "$PROJECT-${params.deploy_to}"
                           kubectl get nodes
                           sed -i "s/IMAGE_VERSION/${params.appVersion}/g" values-${params.deploy_to}.yaml
                           helm upgrade --install $COMPONENT -f values-${params.deploy_to}.yaml -n $PROJECT --create -namespace .
                        """
                    }
                }
            }
        }
        stage('Check Status'){
            steps{
                script{
                    withAWS(credentials: 'aws-cli', region: 'us-east-1') {
                        def deploymentStatus = sh(returnStdout: true, script: "kubectl rollout status deployment/catalogue --timeout=30s -n $PROJECT || echo FAILED").trim()
                        if (deploymentStatus.contains("successfully rolled out")) {
                            echo "Deployment is success"
                        } 
                        else {
                           sh """ 
                               helm rollback  $COMPONENT -n $PROJECT
                               sleep 20
                            """
                            def rollbackStatus = sh(returnStdout: true, script: "kubectl rollout status deployment/catalogue --timeout=30s -n $PROJECT || echo FAILED").trim()
                            if (rollbackStatus.contains("successfuly rolled out")) {
                               error "Deployment is Failure,Rollback is Success"
                            }
                            else{
                               error "Deployment is Failure,Rollback is Failure,Application is not running"
                            }
                        }                
                    }
                }
            }             
        }
    }

    post { 
        always { 
            echo 'I will always say Hello again!'
            deleteDir() // delete post build pipeline in workspace  
        }
        success { 
            echo 'hello success'
        }
        failure { 
            echo 'hello failure'
        }
    }

}


