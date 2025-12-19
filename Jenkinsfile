pipeline {
    agent any

    tools {
        maven 'Maven'
        jdk 'JDK8'
    }

    environment {
        // 构建信息
        BUILD_VERSION = "${BUILD_ID}-${env.BUILD_TIMESTAMP}"
        REPO_URL = 'git@github.com:backend-ex/spot-main.git'

        // 根据不同环境设置变量
        DEPLOY_CONFIG = readJSON file: "jenkins/configs/test-config.json"
    }

    stages {
        stage('初始化') {
            steps {
                script {
                    echo "开始部署 ${SERVICE_NAME} 到 test 环境"
                    echo "构建版本: ${BUILD_VERSION}"
                    echo "Git分支: ${BRANCH}"

                    // 读取服务配置
                    def servicesConfig = readJSON file: 'jenkins/configs/services.json'
                    def allServices = servicesConfig.collect { it.toString() }
                    env.SERVICES_TO_DEPLOY = params.SERVICE_NAME == 'ALL' ? allServices : [params.SERVICE_NAME]
                    
                    // 调试信息
                    echo "所有服务: ${allServices}"
                    echo "当前部署服务: ${env.SERVICES_TO_DEPLOY}"
                }
            }
        }

        stage('代码检出') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${BRANCH}"]],
                    extensions: [],
                    userRemoteConfigs: [[
                        url: REPO_URL,
                        credentialsId: 'github-jenkins-key'
                    ]]
                ])

                // 保存提交信息
                sh '''
                    git log -1 --pretty=format:"%H | %an | %ad | %s" > commit_info.txt
                '''
            }
        }

        stage('依赖构建') {
            steps {
                dir('.') {
                    sh 'mvn clean install -DskipTests=true -Dmaven.test.skip=true'
                }
            }
        }

        stage('服务构建') {
            steps {
                script {
                    // 确保 SERVICES_TO_DEPLOY 是正确的数组格式
                    def servicesToDeploy = env.SERVICES_TO_DEPLOY instanceof String ? 
                                          new groovy.json.JsonSlurper().parseText(env.SERVICES_TO_DEPLOY) : 
                                          env.SERVICES_TO_DEPLOY
                    
                    servicesToDeploy.each { serviceName ->
                        // 确保 serviceName 是字符串而非数组
                        def service = serviceName instanceof String ? serviceName : serviceName.toString()
                        
                        stage("构建 ${service}") {
                            echo "开始构建服务: ${service}"
                            dir("services/${service}") {
                                if (params.SKIP_TESTS) {
                                    sh "mvn clean package -DskipTests"
                                } else {
                                    sh "mvn clean package"
                                }

                                // 归档制品
                                archiveArtifacts artifacts: "target/*.jar", fingerprint: true

                                // 生成部署包
                                sh """
                                    mkdir -p ${WORKSPACE}/deploy-packages/${service}
                                    cp target/*.jar ${WORKSPACE}/deploy-packages/${service}/
                                    cp src/main/resources/application-prod.properties ${WORKSPACE}/deploy-packages/${service}/ 2>/dev/null || true
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('分发部署') {
            steps {
                script {
                    // 确保 SERVICES_TO_DEPLOY 是正确的数组格式
                    def servicesToDeploy = env.SERVICES_TO_DEPLOY instanceof String ? 
                                          new groovy.json.JsonSlurper().parseText(env.SERVICES_TO_DEPLOY) : 
                                          env.SERVICES_TO_DEPLOY
                    
                    servicesToDeploy.each { serviceName ->
                        // 确保 serviceName 是字符串而非数组
                        def service = serviceName instanceof String ? serviceName : serviceName.toString()
                        
                        stage("部署 ${service}") {
                            // 获取该服务的服务器列表
                            def servers = DEPLOY_CONFIG.services[service]?.servers ?: []
                            if (servers.isEmpty()) {
                                echo "警告: ${service} 在 test 环境中没有配置服务器"
                                return
                            }

                            servers.each { server ->
                                stage("部署到 ${server}") {
                                    deployToServer(service, server)
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

// 部署到单台服务器的方法
def deployToServer(serviceName, serverConfig) {
    def host = serverConfig.host
    def port = serverConfig.port ?: 22
    def deployPath = serverConfig.deployPath ?: "/opt/apps/${serviceName}"
    def rs = serverConfig.restartScript

    echo "部署 ${serviceName} 到服务器 ${host}"

    // 使用SSH连接服务器执行部署
    sshagent(['server-ssh-credentials']) {
        sh """
            # 上传部署包
            scp -P ${port} -o StrictHostKeyChecking=no \
                ${WORKSPACE}/deploy-packages/${serviceName}/* \
                jenkins@${host}:${deployPath}/tmp/

            # 执行远程部署脚本
            ssh -p ${port} -o StrictHostKeyChecking=no jenkins@${host} \
                "bash ${deployPath}/${rs}.sh ${serviceName} ${BUILD_VERSION}"
        """
    }
}