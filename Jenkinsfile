pipeline {
    agent {
        label 'AGENT-1'
    }

    environment {
        appVersion = ''
        REGION     = "us-east-1"
        ACC_ID     = "203981192510"
        PROJECT    = "roboshop"
        COMPONENT  = "catalogue"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    parameters {
        string(name: 'appVersion', description: 'image version of the application')
        choice(name: 'deploy_to', choices: ['dev', 'qa', 'prod'], description: 'Pick the Environment')
    }

    stages {
        stage('deploy') {
            steps {
                script {
                    withAWS(credentials: 'aws-cli', region: "${REGION}") {
                        sh """
                           # Update kubeconfig
                           aws eks update-kubeconfig --region $REGION --name "$PROJECT-${params.deploy_to}"
                           kubectl get nodes

                           # Ensure namespace exists
                           kubectl get namespace $PROJECT >/dev/null 2>&1 || kubectl create namespace $PROJECT

                           # Patch Helm ownership labels/annotations
                           kubectl label namespace $PROJECT app.kubernetes.io/managed-by=Helm --overwrite
                           kubectl annotate namespace $PROJECT meta.helm.sh/release-name=$COMPONENT --overwrite
                           kubectl annotate namespace $PROJECT meta.helm.sh/release-namespace=$PROJECT --overwrite

                           # Update values file with image version
                           sed -i "s/IMAGE_VERSION/${params.appVersion}/g" values-${params.deploy_to}.yaml

                           # Deploy using Helm
                           helm upgrade --install $COMPONENT -f values-${params.deploy_to}.yaml -n $PROJECT .
                        """
                    }
                }
            }
        }

        stage('Check Status') {
            steps {
                script {
                    withAWS(credentials: 'aws-cli', region: "${REGION}") {
                        def deploymentStatus = sh(
                            returnStdout: true, 
                            script: "kubectl rollout status deployment/${COMPONENT} --timeout=30s -n $PROJECT || echo FAILED"
                        ).trim()

                        if (deploymentStatus.contains("successfully rolled out")) {
                            echo "Deployment is success"
                        } else {
                            sh """
                                helm rollback $COMPONENT -n $PROJECT
                                sleep 20
                            """
                            def rollbackStatus = sh(
                                returnStdout: true,
                                script: "kubectl rollout status deployment/${COMPONENT} --timeout=30s -n $PROJECT || echo FAILED"
                            ).trim()
                            if (rollbackStatus.contains("successfully rolled out")) {
                                error "Deployment is Failure, Rollback Success"
                            } else {
                                error "Deployment is Failure, Rollback Failure. Application is not running"
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
            deleteDir()
        }
        success {
            echo 'hello success'
        }
        failure {
            echo 'hello failure'
        }
    }
}




// pipeline {
//      // agent any 
//     agent {
//         label 'AGENT-1'
//     }
//     environment {
//         appVersion = ''
//         REGION = "us-east-1"
//         ACC_ID = "203981192510"
//         PROJECT = "roboshop"
//         COMPONENT = "catalogue"
//         //COURSE = 'jenkins'
//     }

//     // }
//     options { // pipeline expries 30 mint
//         timeout(time: 30, unit: 'MINUTES')
//         disableConcurrentBuilds() // not parallel to pipelines at a time so, one complted after another complted
//     }
//     parameters {
//         string(name: 'appVersion',  description: 'image version of the application')
//         choice(name: 'deploy_to', choices: ['dev', 'qa', 'prod'], description: 'Pick the Environment')
        
//     } 
//    // build
//    stages {
//         stage('deploy') {
//             steps {
//                 script {
//                     withAWS(credentials: 'aws-cli', region: 'us-east-1') {
//                         sh """
//                            aws eks update-kubeconfig --region $REGION --name "$PROJECT-${params.deploy_to}"
//                            kubectl get nodes
//                            kubectl apply -f 01-namespace.yaml
//                            sed -i "s/IMAGE_VERSION/${params.appVersion}/g" values-${params.deploy_to}.yaml
//                            helm upgrade --install $COMPONENT -f values-${params.deploy_to}.yaml -n $PROJECT .
//                         """
//                     }
//                 }
//             }
//         }
//         stage('Check Status'){
//             steps{
//                 script{
//                     withAWS(credentials: 'aws-cli', region: 'us-east-1') {
//                         def deploymentStatus = sh(returnStdout: true, script: "kubectl rollout status deployment/catalogue --timeout=30s -n $PROJECT || echo FAILED").trim()
//                         if (deploymentStatus.contains("successfully rolled out")) {
//                             echo "Deployment is success"
//                         } else {
//                            sh """
//                                helm rollback  $COMPONENT -n $PROJECT
//                                sleep 20
//                             """
//                             def rollbackStatus = sh(returnStdout: true, script: "kubectl rollout status deployment/catalogue --timeout=30s -n $PROJECT || echo FAILED").trim()
//                             if (rollbackStatus.contains("successfully rolled out")) {
//                                error "Deployment is Failure, Rollback Success"
//                             }
//                            else{
//                              error "Deployment is Failure, Rollback Failure. Application is not running"
//                             }
//                         }                
//                     }
//                 }
//             }             
//         }
//     }

//     post { 
//         always { 
//             echo 'I will always say Hello again!'
//             deleteDir() // delete post build pipeline in workspace  
//         }
//         success { 
//             echo 'hello success'
//         }
//         failure { 
//             echo 'hello failure'
//         }
//     }

// }


