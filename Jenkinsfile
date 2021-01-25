#!/usr/bin/env groovy

// Prefix values used elsewhere in the script
def genericPrefix = env.JOB_NAME.toLowerCase().replaceAll(' ', '-')
def kubePrefix = genericPrefix
if (kubePrefix.length() > 26) {
  kubePrefix = genericPrefix.substring(0, 26)
}

// Label used when launching the pod in Kubernetes to run the containers
def label = "${kubePrefix}-${UUID.randomUUID().toString()}"
def datestamp = new Date().format('yyyyMMddHHmmss')

def targetCloud = 'preprod-ohio-application'
def environmentName = 'dev'
def podName = 'dealer-pod'
def namespace = "${podName}-${environmentName}"
def releaseName = "mq-dealer-${environmentName}"

def kubeContext = 'preprod-ohio-app'

def helmSecretName = 'chartmuseum-credentials'
def helmVersion    = '3.4.0'
def helmChartName    = 'stable/rabbitmq-ha'
def helmChartVersion = '1.46.4'
def helmValuesFile = 'rabbitmq.yaml'

def slackSuccessChannel = '#champ-builds'
def slackSuccessColor   = 'good'

def slackErrorChannel = '#champ-build-errors'
def slackErrorColor   = 'danger'

customMessage = "Starting Helm deploy for ${currentBuild.fullDisplayName}"
slackSend(channel: slackSuccessChannel,
            color: slackSuccessColor,
            message: customMessage)

// See https://github.com/jenkinsci/kubernetes-plugin/blob/master/src/main/java/org/csanchez/jenkins/plugins/kubernetes/PodTemplate.java
// for a detailed list of options
podTemplate(
    label: label,
    cloud: targetCloud,
    serviceAccount: 'jenkins',
    namespace: 'devops',
    containers: [
        containerTemplate(name: 'helm',
            image: "",
            alwaysPullImage: true,
            ttyEnabled: true,
            command: 'cat',
            envVars: [
                // TODO: Parameterize or calculate the next two.
                envVar(key: 'AWS_DEFAULT_REGION', value: 'us-east-1'),
                envVar(key: 'AWS_ACCESS_KEY_ID', value: ''),
                envVar(key: 'AWS_SECRET_ACCESS_KEY', value: ''),
                envVar(key: 'RELEASE_NAME', value: "${env.RELEASE_NAME}"),
                envVar(key: 'KUBE_CONTEXT', value: kubeContext),
                envVar(key: 'ENV_NAME', value: environmentName),
                envVar(key: 'POD_NAME', value: podName),
                envVar(key: 'NAMESPACE', value: namespace),
                envVar(key: 'RELEASE_NAME', value: releaseName),
                envVar(key: 'HELM_CHART_NAME', value: helmChartName),
                envVar(key: 'HELM_VALUES_FILE', value: helmValuesFile),
                secretEnvVar(key: 'HELM_REPO_USERNAME', secretName: helmSecretName, secretKey: 'username'),
                secretEnvVar(key: 'HELM_REPO_PASSWORD', secretName: helmSecretName, secretKey: 'password')
            ]),
    ],
    volumes: [
        secretVolume(secretName: 'kubernetes-access', mountPath: '/root/.kube'),
        secretVolume(secretName: 'kubernetes-access', mountPath: '/home/jenkins/.kube'),
    ]

){
    node(label) {
        def repo = checkout(changelog: false,
                    poll: false,
                    scm: [$class: 'GitSCM',
                            branches: [[name: '*/master']],
                            doGenerateSubmoduleConfigurations: false,
                            extensions: [],
                            submoduleCfg: [],
                            userRemoteConfigs: [[credentialsId: 'github-ownum-bot-private-key', url: 'git@github.com:champtitles/devops-global-helmchart-config.git']]
                    ])

        // These are here because the values cannot be calculated before the pod template is setup
        // and thus cannot be injected into the pod at startup.
        withEnv([
            "BUILD_ID=${env.BUILD_ID}",
            "BUILD_NUMBER=${env.BUILD_NUMBER}",

        ]) {
            stage('Deploy Services') {
                container('helm') {
                    try {
                        stage('Environment Setup') {
                            sh """#!/bin/bash -e
                               helm repo add \
                                 stable \
                                 https://charts.helm.sh/stable

                               helm repo update
                               """
                        }
                    } catch (Exception ex) {
                        customMessage = "FAILURE during Helm setup for ${currentBuild.fullDisplayName} see ${currentBuild.absoluteUrl} for details"
                        slackSend(channel: slackErrorChannel,
                                    color: slackErrorColor,
                                    message: customMessage)
                        throw ex
                    }
                    try {
                        stage('Helm Execution') {
                            sh """#!/bin/bash -ex
                                if helm ls --namespace \${NAMESPACE} --kube-context=\${KUBE_CONTEXT} -q | grep \${RELEASE_NAME} > /dev/null ; then
                                    printf 'Performing ugprade for %s\n' "\${RELEASE_NAME}"
                                    HELM_VERB='upgrade'
                                    export ERLANGCOOKIE=\$(kubectl get secrets --context=\${KUBE_CONTEXT} -n \${NAMESPACE} \${RELEASE_NAME}-rabbitmq-ha -o jsonpath="{.data.rabbitmq-erlang-cookie}" | base64 --decode)
                                    export ERLANGOPT="--set rabbitmqErlangCookie=\${ERLANGCOOKIE}"
                                else
                                    printf 'Performing initial install deployment for %s\n' "\${RELEASE_NAME}"
                                    HELM_VERB='install'
                                    export ERLANGOPT=''
                                fi
                                VERSION_OPTION=''
                                if [ "${helmChartVersion}" != "latest" ] ; then
                                    VERSION_OPTION="--version ${helmChartVersion}"
                                fi

                                helm --kube-context=\${KUBE_CONTEXT} \
                                  \${HELM_VERB} \
                                  \${RELEASE_NAME} \${ERLANGOPT} \
                                  --namespace \${NAMESPACE} \
                                  -f ./\${ENV_NAME}/\${POD_NAME}/\${HELM_VALUES_FILE} \
                                  \${HELM_CHART_NAME} \${VERSION_OPTION}
                            """
                        }
                    } catch (Exception ex) {
                        customMessage = "FAILURE during Helm deploy for ${currentBuild.fullDisplayName} see ${currentBuild.absoluteUrl} for details"
                        slackSend(channel: slackErrorChannel,
                                    color: slackErrorColor,
                                    message: customMessage)
                        throw ex
                    }
                }
            }
        }
    }
}

customMessage = "Helm deploy for ${currentBuild.fullDisplayName} completed, please confirm deploy status in logz.io or using kubectl"
slackSend(channel: slackSuccessChannel,
            color: slackSuccessColor,
            message: customMessage)
