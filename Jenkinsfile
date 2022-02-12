pipeline {
    agent any
    environment {
        ENV_PROD = 'prod'
        ENV_DEV = 'dev'
        ENV_TESTING = 'testing'
        PROJECT_NAME = 'OUCOnline.DataCenter'
        IMAGES_NAME = 'allinone-datacenter'
        ENV = 'prod'
        NAMESPACE = 'allinone'
    }
    tools {
        maven "Maven3"
    }        
    stages {
        stage('init') {
            steps {
                checkout scm;
                script{
                    def branchName = BRANCH_NAME 
                    def releaseRegex = /v[0-9]*.[0-9]*.[0-9]*(.[0-9]?)/;

                    if (branchName == "main" || branchName == "master") {
                        ENV = 'dev';
                    } else if (branchName.indexOf('Release') == 0) {
                        ENV = 'testing';
                    } else if (branchName.matches(releaseRegex)) {
                        ENV = 'prod';
                    }
                }
                sh "echo ${ENV}"
                sh 'pwd .'
                sh 'echo BRANCH_NAME：${BRANCH_NAME}'
                sh 'echo CHANGE_ID：${CHANGE_ID}'
                sh 'echo CHANGE_URL：${CHANGE_URL}'
                sh 'echo CHANGE_TITLE：${CHANGE_TITLE}'
                sh 'echo CHANGE_AUTHOR：${CHANGE_AUTHOR}'
                sh 'echo CHANGE_AUTHOR_DISPLAY_NAME：${CHANGE_AUTHOR_DISPLAY_NAME}'
                sh 'echo CHANGE_AUTHOR_EMAIL：${CHANGE_AUTHOR_EMAIL}'
                sh 'echo CHANGE_TARGET：${CHANGE_TARGET}'
                sh 'echo BUILD_NUMBER：${BUILD_NUMBER}'
                sh 'echo BUILD_TIMESTAMP: ${BUILD_TIMESTAMP}'
                sh 'echo BUILD_ID：${BUILD_ID}'
                sh 'echo BUILD_DISPLAY_NAME：${BUILD_DISPLAY_NAME}'
                sh 'echo JOB_NAME：${JOB_NAME}'
                sh 'echo JOB_BASE_NAME：${JOB_BASE_NAME}'
                sh 'echo BUILD_TAG：${BUILD_TAG}'
                sh 'echo EXECUTOR_NUMBER：${EXECUTOR_NUMBER}'
                sh 'echo NODE_NAME：${NODE_NAME}'
                sh 'echo NODE_LABELS：${NODE_LABELS}'
                sh 'echo WORKSPACE：${WORKSPACE}'
                sh 'echo JENKINS_HOME：${JENKINS_HOME}'
                sh 'echo JENKINS_URL：${JENKINS_URL}'
                sh 'echo BUILD_URL：${BUILD_URL}'
                sh 'echo JOB_URL：${JOB_URL}'
                sh 'echo JOB_URL：${JOB_URL}'
                sh 'echo GIT_COMMIT：${GIT_COMMIT}'
                sh 'echo GIT_PREVIOUS_COMMIT：${GIT_PREVIOUS_COMMIT}'
                sh 'echo GIT_PREVIOUS_SUCCESSFUL_COMMIT：${GIT_PREVIOUS_SUCCESSFUL_COMMIT}'
                sh 'echo GIT_BRANCH：${GIT_BRANCH}'
                sh 'echo GIT_BRANCH：${GIT_TAG}'
                sh 'echo GIT_LOCAL_BRANCH：${GIT_LOCAL_BRANCH}'
                sh 'echo GIT_URL：${GIT_URL}'
                sh 'echo GIT_COMMITTER_NAME：${GIT_COMMITTER_NAME}'
                sh 'echo GIT_AUTHOR_NAME：${GIT_AUTHOR_NAME}'
                sh 'echo GIT_COMMITTER_EMAIL：${GIT_COMMITTER_EMAIL}'
                sh 'echo GIT_AUTHOR_EMAIL：${GIT_AUTHOR_EMAIL}'
                sh 'echo SVN_REVISION：${SVN_REVISION}'
                sh 'echo SVN_URL：${SVN_URL}'
                dotnetRestore(sources: ["https://api.nuget.org/v3/index.json", "${CustomNugetUrl}"]);
            }
        }
        stage('build') {
            when {
                anyOf {
                    branch 'main';
                    branch 'master';
                    branch 'Release*';
                    tag pattern: 'v[0-9]*.[0-9]*.[0-9]*(.[0-9]?)', comparator: 'REGEXP'
                }
            }            
            stages() {
                stage('build source code') {
                    steps {
                        sh 'echo build source code'
                        dotnetPublish(configuration: "Release", outputDirectory: "./${PROJECT_NAME}.API/publish", project: "./${PROJECT_NAME}.API/${PROJECT_NAME}.API.csproj");
                    }
                }
                stage('push nuget pack') {
                    when { anyOf { branch 'main'; branch 'master'; branch 'Release*' } }
                    steps {
                        sh "find ./nupkgs/ -name *.nupkg -type f |xargs rm -f";
                        dotnetBuild(configuration: "Release", outputDirectory: "./nupkgs", project: "./${PROJECT_NAME}.Abstractions/${PROJECT_NAME}.Abstractions.csproj");
                        dotnetNuGetPush(apiKeyId: "CustomNuget", source: "${CustomNugetUrl}", root: "./nupkgs/*.nupkg");
                    }
                }
                stage('publish dev') {
                    when {
                        anyOf {
                            branch 'main';
                            branch 'master';
                        }
                    }
                    stages(){
                        stage('build docker'){
                            steps{
                                script{
                                    def localDocerImage = docker.build("gjkfdx2018/${IMAGES_NAME}dev:${BUILD_TIMESTAMP}", "./${PROJECT_NAME}.API/publish");
                                    docker.withRegistry("${HuaweiCloudSWRUrl}", 'HuaweiCloudSWR') {
                                        localDocerImage.push();
                                    }
                                }
                            }
                        }
                        stage("publish kubernetes"){
                            steps{
                                withKubeConfig([credentialsId: 'allinonekubeconfig',contextName:'external']) {
                                    sh "kubectl set image deployments/${IMAGES_NAME} ${IMAGES_NAME}=100.125.0.198:20202/gjkfdx2018/${IMAGES_NAME}dev:${BUILD_TIMESTAMP} -n ${NAMESPACE}"
                                }
                            }
                        }
                        stage('接口测试') {
                            steps {
                                timestamps {
                                    echo '执行等待前 '
                                }                                
                                sleep 20 
                                timestamps {
                                    echo '执行等待后 '
                                }                                
                                checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'githubapp.OUCOnline', url: 'https://github.com/OUCOnline/OUCOnline.AllInOne.Testing']]])
                                sh 'mvn -Dmaven.test.failure.ignore=true -Dmaven.testfile=datacenter.xml clean package'
                            }
                            post {
                                success {
                                    allure includeProperties: false, jdk: '', results: [[path: 'target/allure-results']]
                                }
                            }
                        }                        
                    }
                }
                stage('publish testing') {
                    when {
                        branch 'Release*'
                    }
                    stages(){
                        stage('build docker'){
                            steps{
                                script{
                                    def localDocerImage = docker.build("gjkfdx2018/${IMAGES_NAME}testing:${BRANCH_NAME.toLowerCase().replaceAll('release', '')}.${BUILD_TIMESTAMP}", "./${PROJECT_NAME}.API/publish");
                                    docker.withRegistry("${HuaweiCloud4SWRUrl}", 'HuaweiCloud4SyxySWR') {
                                        localDocerImage.push();
                                    }
                                }
                            }
                        }
                        stage("publish kubernetes"){
                            steps{
                                withKubeConfig([credentialsId: 'allinonekubeconfig',contextName:'external-shiyan']) {
                                    sh "kubectl set image deployments/${IMAGES_NAME} ${IMAGES_NAME}=swr.cn-north-4.myhuaweicloud.com/gjkfdx2018/${IMAGES_NAME}testing:${BRANCH_NAME.toLowerCase().replaceAll('release', '')}.${BUILD_TIMESTAMP} -n ${NAMESPACE}"
                                }
                            }
                        }
                    }
                }
                stage('publish release') {
                    when { tag pattern: 'v[0-9]*.[0-9]*.[0-9]*(.[0-9]?)', comparator: 'REGEXP' }
                    stages(){
                        stage('build docker'){
                            steps{
                                script{
                                    def localDocerImage = docker.build("ouconline/${IMAGES_NAME}:${BRANCH_NAME.toLowerCase().replaceAll('v', '')}", "./${PROJECT_NAME}.API/publish");
                                    docker.withRegistry("${HuaweiCloud4SWRUrl}", 'HuaweiCloud4CompanySyxySWR') {
                                        localDocerImage.push();
                                    }                                    
                                }
                            }
                        }
                        stage("publish kubernetes"){
                            steps{
                                sh 'echo 请到cce手动更新deployment'
                            }
                        }
                    }
                }

            }
        }
    }
}
