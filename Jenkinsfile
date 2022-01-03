
import java.time.LocalDateTime
import java.time.*
import java.time.format.DateTimeFormatter
//Variables that are specefic for each user - to be changed
USERNAME = "wsoualhi"
//variables that are same for everyone 
IMAGE_REPOSITORY = "simple-nginx"
KUBERNETES_INGRESS = "ingress"
//Prod variables 
TARGET_CLUSTER_REGISTRY_URI = 'https://registry.prod.equinix.presales.demo.mirantis.com'
TARGET_CLUSTER_REGISTRY_HOSTNAME = 'registry.prod.equinix.presales.demo.mirantis.com'
TARGET_CLUSTER_KUBE_DOMAIN_NAME = "prod.presales.demo.mirantis.com"
TARGET_CLUSTER_KUBERNETES_CONTEXT = "ucp_kube.prod.equinix.presales.demo.mirantis.com:5443_jenkins"
//variables that change for every user
IMAGE_NAMESPACE_DEV = "${USERNAME}-dev"
IMAGE_NAMESPACE_PROD = "${USERNAME}-prod"
KUBERNETES_NAMESPACE_DEV = "${IMAGE_NAMESPACE_DEV}"
KUBERNETES_NAMESPACE_PROD = "${IMAGE_NAMESPACE_PROD}"
APPLICATION_DOMAIN = "${USERNAME}.${TARGET_CLUSTER_KUBE_DOMAIN_NAME}"
//variables that change on every execution
def IMAGE_TAG = LocalDateTime.now()
IMAGE_TAG = IMAGE_TAG.format(DateTimeFormatter.ofPattern("yyyyMMddHHmm"))

node {
    def docker_image

    stage('Checkout') {
        checkout scm
    }

    stage('Build') {
        docker_image = docker.build("${IMAGE_NAMESPACE_DEV}/${IMAGE_REPOSITORY}")
    }

    stage('Unit Tests') {
        docker_image.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Push') {
        //MSRequinixProd are the credentials configured in Jenkins
        docker.withRegistry(TARGET_CLUSTER_REGISTRY_URI, 'MSRequinixProd') {
            docker_image.push(IMAGE_TAG)
        }
    }

    stage('Scan') {
        //httpRequest acceptType: 'APPLICATION_JSON', authentication: TARGET_CLUSTER['REGISTRY_CREDENTIALS_ID'], contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${TARGET_CLUSTER['REGISTRY_URI']}/api/v0/imagescan/scan/${IMAGE_NAMESPACE_DEV}/${IMAGE_REPOSITORY}/${IMAGE_TAG}/linux/amd64"
        httpRequest acceptType: 'APPLICATION_JSON', authentication: 'MSRequinixProd', contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${TARGET_CLUSTER_REGISTRY_URI}/api/v0/imagescan/scan/${IMAGE_NAMESPACE_DEV}/${IMAGE_REPOSITORY}/${IMAGE_TAG}/linux/amd64"

        def scan_result

        def scanning = true
        while(scanning) {
            def scan_result_response = httpRequest acceptType: 'APPLICATION_JSON', authentication: 'MSRequinixProd', httpMode: 'GET', ignoreSslErrors: true, responseHandle: 'LEAVE_OPEN', url: "${TARGET_CLUSTER_REGISTRY_URI}/api/v0/imagescan/scansummary/repositories/${IMAGE_NAMESPACE_DEV}/${IMAGE_REPOSITORY}/${IMAGE_TAG}"
            scan_result = readJSON text: scan_result_response.content

            if (scan_result.size() != 1) {
                println('Response: ' + scan_result)
                error('More than one imagescan returned, please narrow your search parameters')
            }

            scan_result = scan_result[0]

            if (!scan_result.check_completed_at.equals("0001-01-01T00:00:00Z")) {
                scanning = false
            } else {
                sleep 15
            }

        }
        println('Response JSON: ' + scan_result)
    }

    stage('Sign Development Image') {
        withEnv(["REGISTRY_HOSTNAME=${TARGET_CLUSTER_REGISTRY_HOSTNAME}",
                 "IMAGE_NAMESPACE=${IMAGE_NAMESPACE_DEV}",
                 "IMAGE_REPOSITORY=${IMAGE_REPOSITORY}",
                 "IMAGE_TAG=${IMAGE_TAG}",
                 ]) {
            withCredentials([string(credentialsId: 'TRUST_SIGNER_PASSPHRASE_CREDENTIALS_ID', variable: 'DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE')]) {
                sh 'docker trust key load /var/lib/jenkins/client-bundle/key.pem'
                sh 'docker trust sign ${REGISTRY_HOSTNAME}/${IMAGE_NAMESPACE}/${IMAGE_REPOSITORY}:${IMAGE_TAG}'
            }
        }
    }

    stage('Deploy to Development') {
        withEnv(["APPLICATION_FQDN=${IMAGE_REPOSITORY}.dev.${APPLICATION_DOMAIN}",
                 "REGISTRY_HOSTNAME=${TARGET_CLUSTER_REGISTRY_HOSTNAME}",
                 "IMAGE_NAMESPACE=${IMAGE_NAMESPACE_DEV}",
                 "IMAGE_REPOSITORY=${IMAGE_REPOSITORY}",
                 "IMAGE_TAG=${IMAGE_TAG}",
                 "USERNAME=${USERNAME}"]) {
                withEnv(["KUBERNETES_CONTEXT=${TARGET_CLUSTER_KUBERNETES_CONTEXT}", 
                         "KUBERNETES_INGRESS=${KUBERNETES_INGRESS}",
                         "KUBERNETES_NAMESPACE=${KUBERNETES_NAMESPACE_DEV}"]) {
                    sh 'envsubst < kubernetes/001_simple-nginx_deployment.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                    sh 'envsubst < kubernetes/002_simple-nginx_service.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                    sh 'envsubst < kubernetes/003_simple-nginx_${KUBERNETES_INGRESS}.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                }
            println("Application deployed to Development: http://${APPLICATION_FQDN}")
        }
    }

    stage('Integration Tests') {
        docker_image.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Promote') {
        httpRequest acceptType: 'APPLICATION_JSON', authentication: 'MSRequinixProd', contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, requestBody: "{\"targetRepository\": \"${IMAGE_NAMESPACE_PROD}/${IMAGE_REPOSITORY}\", \"targetTag\": \"${IMAGE_TAG}\"}", responseHandle: 'NONE', url: "${TARGET_CLUSTER_REGISTRY_URI}/api/v0/repositories/${IMAGE_NAMESPACE_DEV}/${IMAGE_REPOSITORY}/tags/${IMAGE_TAG}/promotion"
    }

    stage('Sign Development Image') {
        withEnv(["REGISTRY_HOSTNAME=${TARGET_CLUSTER_REGISTRY_HOSTNAME}",
                 "IMAGE_NAMESPACE=${IMAGE_NAMESPACE_PROD}",
                 "IMAGE_REPOSITORY=${IMAGE_REPOSITORY}",
                 "IMAGE_TAG=${IMAGE_TAG}",
                 ]) {
            withCredentials([string(credentialsId: 'TRUST_SIGNER_PASSPHRASE_CREDENTIALS_ID', variable: 'DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE')]) {
                sh 'docker pull ${REGISTRY_HOSTNAME}/${IMAGE_NAMESPACE}/${IMAGE_REPOSITORY}:${IMAGE_TAG}'
                sh 'docker trust key load /var/lib/jenkins/client-bundle/key.pem'
                sh 'docker trust sign ${REGISTRY_HOSTNAME}/${IMAGE_NAMESPACE}/${IMAGE_REPOSITORY}:${IMAGE_TAG}'
            }
        }
    }

    stage('Deploy to Production') {
        withEnv(["APPLICATION_FQDN=${IMAGE_REPOSITORY}.prod.${APPLICATION_DOMAIN}",
                 "REGISTRY_HOSTNAME=${TARGET_CLUSTER_REGISTRY_HOSTNAME}",
                 "IMAGE_NAMESPACE=${IMAGE_NAMESPACE_PROD}",
                 "IMAGE_REPOSITORY=${IMAGE_REPOSITORY}",
                 "IMAGE_TAG=${IMAGE_TAG}",
                 "USERNAME=${USERNAME}"]) {
                withEnv(["KUBERNETES_CONTEXT=${TARGET_CLUSTER_KUBERNETES_CONTEXT}", 
                         "KUBERNETES_INGRESS=${KUBERNETES_INGRESS}",
                         "KUBERNETES_NAMESPACE=${KUBERNETES_NAMESPACE_PROD}"]) {
                    sh 'envsubst < kubernetes/001_simple-nginx_deployment.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                    sh 'envsubst < kubernetes/002_simple-nginx_service.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                    sh 'envsubst < kubernetes/003_simple-nginx_${KUBERNETES_INGRESS}.yaml | kubectl --context=${KUBERNETES_CONTEXT} --namespace=${KUBERNETES_NAMESPACE} apply -f -'
                }
            println("Application deployed to Production: http://${APPLICATION_FQDN}")
        }

    }
}
