pipeline {
    agent { label 'roboshop' }

    environment {
        PROJECT      = "roboshop"
        COMPONENT    = "catalogue"
        REGION       = "us-east-1"
        CLUSTER_NAME = "roboshop-dev"
        APP_VERSION  = ''   // left blank, can be set dynamically later
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                          branches: [[name: "*/main"]],
                          userRemoteConfigs: [[url: "https://github.com/siri-123706/catalogue-cd.git",
                                               credentialsId: "ssh-auth"]]
                ])
            }
        }

        stage('Set Version') {
            steps {
                script {
                    // Set APP_VERSION dynamically here
                    // Example: use git commit hash
                    env.APP_VERSION = sh(returnStdout: true, script: "git rev-parse --short HEAD").trim()

                    echo ">>> Using APP_VERSION=${env.APP_VERSION}"
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    withAWS(credentials: 'aws-cli', region: "${REGION}") {
                        echo ">>> Updating kubeconfig"
                        sh "aws eks update-kubeconfig --region ${REGION} --name ${CLUSTER_NAME}"

                        echo ">>> Checking namespace ${PROJECT}"
                        def nsExists = sh(returnStatus: true, script: "kubectl get namespace ${PROJECT}") == 0
                        if (nsExists) {
                            echo "Namespace ${PROJECT} already exists, patching metadata for Helm..."
                            sh "kubectl label namespace ${PROJECT} app.kubernetes.io/managed-by=Helm --overwrite || true"
                            sh "kubectl annotate namespace ${PROJECT} meta.helm.sh/release-name=${COMPONENT} --overwrite || true"
                            sh "kubectl annotate namespace ${PROJECT} meta.helm.sh/release-namespace=${PROJECT} --overwrite || true"
                        }

                        echo ">>> Preparing Helm values"
                        sh "sed -i s/IMAGE_VERSION/${APP_VERSION}/g values-dev.yaml"

                        echo ">>> Deploying with Helm"
                        sh "helm upgrade --install ${COMPONENT} -f values-dev.yaml -n ${PROJECT} --create-namespace ."
                    }
                }
            }
        }

        stage('Check Status') {
            steps {
                script {
                    withAWS(credentials: 'aws-cli', region: "${REGION}") {
                        echo ">>> Checking rollout status"
                        def deploymentStatus = sh(returnStdout: true, script: "kubectl rollout status deployment/${COMPONENT} --timeout=60s -n ${PROJECT} || echo FAILED").trim()

                        if (deploymentStatus.contains("successfully rolled out")) {
                            echo "✅ Deployment is successful"
                        } else {
                            echo "❌ Deployment failed. Collecting debug info..."
                            sh "kubectl get pods -n ${PROJECT} -o wide || true"

                            sh """
                               for pod in \$(kubectl get pods -n ${PROJECT} -l app=${COMPONENT} -o jsonpath='{.items[*].metadata.name}'); do
                                   echo ">>> Describing pod: \$pod"
                                   kubectl describe pod \$pod -n ${PROJECT} || true
                                   echo ">>> Logs for pod: \$pod"
                                   kubectl logs \$pod -n ${PROJECT} --tail=50 || true
                               done
                            """

                            def revision = sh(returnStdout: true, script: "helm history ${COMPONENT} -n ${PROJECT} --max=2 | awk 'NR==2{print \$1}' || echo NONE").trim()

                            if (revision != "NONE" && revision != "1") {
                                echo ">>> Rolling back to previous revision"
                                sh """
                                   helm rollback ${COMPONENT} -n ${PROJECT}
                                   sleep 20
                                """
                                def rollbackStatus = sh(returnStdout: true, script: "kubectl rollout status deployment/${COMPONENT} --timeout=60s -n ${PROJECT} || echo FAILED").trim()

                                if (rollbackStatus.contains("successfully rolled out")) {
                                    error "Deployment failed, rollback successful"
                                } else {
                                    error "Deployment failed and rollback also failed"
                                }
                            } else {
                                error "Deployment failed, but no previous release to rollback to"
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning workspace..."
            deleteDir()
        }
        failure {
            echo "❌ Pipeline failed"
        }
        success {
            echo "✅ Pipeline succeeded"
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
//                         if (deploymentStatus.contains("successfuly rooled out")) {
//                             echo "Deployemnt is success"
//                         } else {
//                            sh """
//                              helm rollback  $COMPONENT -n $PROJECT
//                               sleep 20
//                             """
//                             def rollbackStatus = sh(returnStdout: true, script: "kubectl rollout status deployment/catalogue --timeout=30s -n $PROJECT || echo FAILED").trim()
//                             if (deploymentStatus.contains("successfuly rolled out")) {
//                                error "Deployment is Failure,Rollback is success"
//                             }
//                            else{
//                              error "Deployment is Failure,Rollback is Failure,Application is not running"
//                            }
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


