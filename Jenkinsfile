#!groovy

def checkoutAppCode() {
    def scmVar = checkout ([
            changelog: false,
            $class : 'GitSCM',
            branches : scm.branches,
            extensions : scm.extensions + [
                    [$class: 'CleanBeforeCheckout'],
                    [$class: 'CheckoutOption', timeout: 30],
                    [$class: 'CloneOption', timeout: 30]
            ],
            userRemoteConfigs: scm.userRemoteConfigs
    ])
    return scmVar
}

def setupEnv(scmVar) {
    env.GIT_COMMIT = "${scmVar.GIT_COMMIT}".substring(0,10)
}

def getVersion(commitID, env) {
    commit = commitID.substring(0,10)
    return env+"_"+commit
}

def helmLint(String chart_dir) {
    sh "/usr/local/bin/helm lint ${chart_dir}"

}

def helmDeploy(Map args) {
    println "Running deployment"
    sh "/usr/local/bin/helm upgrade --install ${args.name} ${args.chart_dir} --set ImageTag=${args.tag} --namespace=${args.name}"

    echo "Application ${args.name} successfully deployed. Use helm status ${args.name} to check"
}

pipeline {
    agent any
    parameters {
        choice(choices: ['dev', 'integration', 'production'], description: '', name: 'Environment')
        choice(choices: ['test-ops'], description: '', name: 'App')
    }

    environment {
        REPO_NAME = "hub.docker.com/freeletics"
        DOCKER_HUB_CREDENTIAL = "freeletics_docker_hub"
        CHART_DIR = "$WORKSPACE/charts/app"
        APP_NAME = "test-ops"
    }

    options {
        disableConcurrentBuilds()
    }

    stages {
        stage ("Checkout Application Code"){
            steps{
                script {
                    scm = checkoutAppCode()
                    setupEnv(scm)
                    echo "Checkout"
                }
            }
        }

        stage ("Test cases and code coverage"){
            steps{
                // Specify test cases and coverage logic here
            }
        }

        stage ("Publish test results"){
            steps{
                // Specify logic related to publishing test results
            }
        }

        stage ("Building App") {
            steps {
                script {
                    IMAGENAME = "${env.REPO_NAME}"
                    buildArgs = "--build-arg RUBY_ENV=${params.Environment} "
                    withDockerRegistry(credentialsId: "${env.DOCKER_HUB_CREDENTIAL}") {
                        image = docker.build("${IMAGENAME}", "${buildArgs} .")
                        docker.image("${IMAGENAME}").push(getVersion(env.GIT_COMMIT, params.Environment))
                    }
                }
            }
        }

        stage ('Deploying Application') {

            // run helm chart linter
            helmLint(env.CHART_DIR)

            // run helm chart installation
            helmDeploy(
                    name          : env.APP_NAME,
                    chart_dir     : env.CHART_DIR,
                    tag           : getVersion(env.GIT_COMMIT, params.Environment)
            )
        }
    }

    // Post actions
    post {
        aborted {
            script {
                log.info('Build process is aborted.')
            }
        }
        failure {
            script {
                log.error('Build process failed.')
            }
        }
        success {
            script {
                log.success('Build process completed successfully.')
            }
        }
    }
}

