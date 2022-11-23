#!groovy
AnsibleRepo = 'https://repo.west.com/Conferencing-and-Collaboration/ansibleUM.git'
AnsiblePlaybook = 'dolbycontroller.yml'
AnsibleBranchName = "master"

CloudFoundryCredentialID = "StormCloudPCFCreds"
targetedDeploy = ''

pipeline {
    agent { label 'RHEL7' }
    options {
        timeout(time: 500, unit: 'SECONDS')
    }
    environment {
        to_email = "c&c_devops@west.com"
        CF_HOME = "${WORKSPACE}"
        CF_DIAL_TIMEOUT = 10

    }

    parameters {
        choice(name: 'Territory', choices: ['Dev/devtest-us-central1', 'Test/devtest-us-central1', 'Stage/us-central1', 'Prod/us-central1'],
                    description: 'Please select the Environment to deploy the application')
        string(name: 'Version', defaultValue: "MR", description: 'Desired Version to deploy')

    }
    stages {
        stage ('Deploy Prerequisites ') {
            steps {
                script {

                    if (Territory == null) {
                        echo "Did not pick an environment"
                    } else {
                        (Space, Region) = Territory.split('/')
                    }
                    sh "echo ${Territory} and ${Region}"

                    if (Territory =="Dev/devtest-us-central1") {
                        TargetInventory = 'devbat'
                        AnsibleVaultCredentialID = "AnsibleVaultDevPassword"
                    }
                    if (Territory =="Test/devtest-us-central1") {
                        TargetInventory = 'mainbat'
                        AnsibleVaultCredentialID = "AnsibleVaultDevPassword"

                    }
                    if (Territory =="Stage/us-central1") {
                        TargetInventory = 'B'
                        AnsibleVaultCredentialID = "AnsibleVaultProd"
                        targetedDeploy = ''
                    }
                    if (Territory =="Prod/us-central1") {
                        TargetInventory = 'A'
                        AnsibleVaultCredentialID = "AnsibleVaultProd"
                        targetedDeploy = ''

                    }

                }
            }
        }
        stage('Deploy Dolby Controller App') {
            when { expression {Version!=null && Version.length()>2 }}
            steps {
                deleteDir()
                checkout([
                        $class                           : 'GitSCM',
                        branches                         : [[name: "*/${AnsibleBranchName}"]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions                       : [[$class: 'CloneOption', depth: 0, noTags: true, reference: '', shallow: true]],
                        userRemoteConfigs                : [[url: "${AnsibleRepo}"]]
                ])
                withCredentials([string(credentialsId: "${CloudFoundryCredentialID}", variable: 'PCFTOKEN'), string(credentialsId: "${AnsibleVaultCredentialID}", variable: 'AnsibleVaultCreds')]) {
                    script {
                        PCFFoundry = "${Region}"
                        def sitePCF = readJSON text: "${PCFTOKEN}"
                        def sitePCFUser = sitePCF."${PCFFoundry}".username
                        def sitePCFPass = sitePCF."${PCFFoundry}".password
                        sh "sed -i '/log_path/c\\log_path = ./ansible_deployment.log' ansible.cfg;sed -i '/vault_password_file/c\\' ansible.cfg;"

                        ansiblePlaybook(installation: 'ansible-2.9.12', inventory: "inventories/${TargetInventory}", playbook: "${AnsiblePlaybook}",
                    extraVars: [ver     : "${Version}",
                                noc_user: "Jenkins",
                                ansible_python_interpreter: "/usr/bin/python2",
                                PCF_USER: [value: "${sitePCFUser}", hidden:true],
                                PCF_PASS: [value: "${sitePCFPass}", hidden:true]],
                    limit: "${targetedDeploy}",
                    extras: '-vvv',
                    vaultCredentialsId: "${AnsibleVaultCredentialID}")
                    }
                }
            }
        }
    }
    post {
        // Always runs. And it runs before any of the other post conditions.
        always {
            // Let's wipe out the workspace before we finish!
            deleteDir()
        }
        success {
            script {
                if (Version!=null && Version.length()>2 ) {
                    emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                            mimeType: 'text/html',
                            subject: "[Jenkins] ${JOB_NAME} $Version - Deployment Success to $Territory)",
                            to: "${to_email}",
                            recipientProviders: [[$class: 'CulpritsRecipientProvider']]
                }
            }

        }
        failure {
            script {
                    emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                            mimeType: 'text/html',
                            subject: "[Jenkins] ${JOB_NAME} $Version - Deployment Failed to $Territory ",
                            to: "${to_email}",
                            attachLog: true,
                            recipientProviders: [[$class: 'CulpritsRecipientProvider']]
            }
        }
    }
}

