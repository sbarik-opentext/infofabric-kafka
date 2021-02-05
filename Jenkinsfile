#!groovy
import com.liaison.jenkins.common.kubernetes.*
import com.liaison.jenkins.common.servicenow.ServiceNow
import com.liaison.jenkins.common.slack.*
import com.liaison.jenkins.common.sonarqube.QualityGate
import com.liaison.jenkins.common.testreport.TestResultsUploader
@Library('visibilityLibs')
import com.liaison.jenkins.visibility.Utilities
import com.liaison.jenkins.common.test.*

def deployments = new Deployments()
def k8sDocker = new Docker()
def kubectl = new Kubectl()
def serviceNow = new ServiceNow()
def slack = new Slack()

def utils = new Utilities();

def dockerImageName = "registry-ci.at4d.liacloud.com/alloy/kafka"
def dockerImageTag = "2.0.0"
def otbpDeployment
def otbpDockerImageName = "bpdockerhub/infofabric-kafka"

timestamps {

    node('at4d-c3-agent') {
        stage('Checkout') {
            checkout scm
            env.VERSION = utils.runSh("awk '/^## \\[([0-9])/{ print (substr(\$2, 2, length(\$2) - 2));exit; }' CHANGELOG.md");
            env.GIT_COMMIT = utils.runSh('git rev-parse HEAD')
            env.GIT_URL = utils.runSh("git config remote.origin.url | sed -e 's/\\(.git\\)*\$//g' ")
            env.REPO_NAME = utils.runSh("basename -s .git ${env.GIT_URL}")
            env.RELEASE_NOTES = utils.runSh("awk '/## \\[${env.VERSION}\\]/{flag=1;next}/## \\[/{flag=0}flag' CHANGELOG.md")
            currentBuild.displayName = env.VERSION
            stash name: "k8smanifest", includes: "K8sfile.yaml"

        }

        stage('Build Docker image (OTBP)') {
    
            unstash name: 'k8smanifest'

            otbpDeployment = deployments.create(
                name: 'infofabric-elasticsearch',
                version: env.VERSION,
                description: 'Deployment of infofabric-elasticsearch',
                dockerImageName: otbpDockerImageName,
                dockerImageTag: env.VERSION,
                yamlFile: 'K8sfile.yaml',   // optional, defaults to 'K8sfile.yaml'
                gitUrl: env.GIT_URL,        // optional, defaults to env.GIT_URL
                gitCommit: env.GIT_COMMIT,  // optional, defaults to env.GIT_COMMIT
                gitRef: env.VERSION,        // optional, defaults to env.GIT_COMMIT
                kubectl: kubectl
            )

//            k8sDocker.build(imageName: otbpDockerImageName);
//            milestone label: 'OTBP Docker image built', ordinal: 101
                def alloykafkaImage = docker.image("${dockerImageName}:${dockerImageTagC3}")
                alloykafkaImage.pull()
                sh "docker tag '${alloykafkaImage.id}' '${dockerImageName}'"
                sh "docker tag '${alloykafkaImage.id}' '${otbpDockerImageName}'"

            	//k8sDocker.push(imageName: dockerImageName, imageTag: env.VERSION)
                k8sDocker.push(imageName: otbpDockerImageName, imageTag: env.VERSION, registry: Registry.BROOKPARK)
            //}
        }

        
    }


        stage('Deploy To K8S, DEV') {

            try {
                deployments.deploy(
                        deployment: otbpDeployment,
                        kubectl: kubectl,
                        serviceNow: serviceNow,
                        namespace: Namespace.DEVELOPMENT,
                        rollingUpdate: true,     // optional, defaults to true
                        clusters: [ Cluster.OTBP ]
                )
            } catch (err) {
                currentBuild.result = "FAILURE";
                error "${err}"
            }
        }

}
