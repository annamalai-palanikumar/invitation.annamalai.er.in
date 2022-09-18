pipeline {
    parameters {
      choice choices: ['github.com'], name: 'REPOSITORY_SOURCE'
      choice choices: ['annamalai-palanikumar'], name: 'REPOSITORY_GROUP'
      choice choices: ['invitation.annamalai.er.in'], name: 'REPOSITORY_NAME'
    }
    agent any
    stages {
        stage('Verification and Installation') {
            steps {
                script {
                    sh "echo Verification and Installation Completed"
                }
            }
        }
        stage('Build & Unit Testing') {
            steps {
                script {
                    echo 'Build & Unit Testing Started'
                    def webServerApplicationRoot = '/var/tmp/www/' + params.REPOSITORY_GROUP
                    if(!fileExists(webServerApplicationRoot)) {
                        sh 'mkdir -p ' + webServerApplicationRoot
                    }
                    def gitReference = params.REPOSITORY_SOURCE +':' + params.REPOSITORY_GROUP + '/' + params.REPOSITORY_NAME
                    def gitURL = 'git@' + gitReference +'.git'
                    def srcFolder = webServerApplicationRoot + '/' + params.REPOSITORY_NAME
                    sh "sudo chown -R jenkins:jenkins " + webServerApplicationRoot
                    if(!fileExists(srcFolder)) {
                        sh 'git -C '+ webServerApplicationRoot +' clone ' + gitURL
                    } else {
                        sh "git -C "+ srcFolder +" pull"
                    }
                    def deploymentConfigFileName = srcFolder + '/deployment-descriptor.yml'
                    def deploymentConfig = readYaml(file: deploymentConfigFileName)
                    def hasDeploymentConfig = deploymentConfig != null;
                    def hasApplicationType = hasDeploymentConfig && deploymentConfig.application_type != null && !deploymentConfig.application_type.isEmpty()
                    if(hasApplicationType) {
                        switch(deploymentConfig.application_type) {
                            case 'SPRING_BOOT_APPLICATION':
                                sh 'mvn clean install -f ' + srcFolder + '/pom.xml'
                            break
                        }
                    }
                    sh "sudo chown -R www-data:www-data " + srcFolder
                    echo 'Build & Unit Testing Completed'
                }
            }
        }
        stage('Deployment to Test Environment') {
            steps {
                script {
                    echo 'Deployment to Test Environment Started'
                    def webServerTestRoot = '/var/tmp/www/'
                    def gitReference = params.REPOSITORY_GROUP + '/' + params.REPOSITORY_NAME
                    def srcFolder = webServerTestRoot + gitReference
                    sh "sudo chown -R www-data:www-data "+ srcFolder
                    def deploymentConfigFileName = srcFolder + '/deployment-descriptor.yml'
                    def deploymentConfig = readYaml(file: deploymentConfigFileName)
                    def hasDeploymentConfig = deploymentConfig != null;
                    def hasTestDomain = hasDeploymentConfig && deploymentConfig.test_domain != null && !deploymentConfig.test_domain.isEmpty()
                    def hasApplicationType = hasDeploymentConfig && deploymentConfig.application_type != null && !deploymentConfig.application_type.isEmpty()
                    def hasNginxConfigFileName = hasDeploymentConfig && deploymentConfig.nginx_config != null && !deploymentConfig.nginx_config.isEmpty()
                    if(hasTestDomain && hasApplicationType && hasNginxConfigFileName) {
                        def domain = deploymentConfig.test_domain
                        echo 'Deploying to '+ domain +' from ' + gitReference
                        def hasApplicationServiceConfigFileName = deploymentConfig.app_service_config != null && !deploymentConfig.app_service_config.isEmpty()
                        if(hasApplicationServiceConfigFileName) {
                            def appServiceConfig = readFile(file: deploymentConfig.app_service_config)
                            appServiceConfig = appServiceConfig.replace("#APPLICATION_ROOT_FOLDER#", srcFolder)
                            switch(deploymentConfig.application_type) {
                                case 'SPRING_BOOT_APPLICATION':
                                    appServiceConfig = appServiceConfig.replace("#APP_NAME#", deploymentConfig.appname)
                                    appServiceConfig = appServiceConfig.replace("#PORT#", deploymentConfig.port)
                                break
                                case 'PYTHON_APPLICATION':
                                    appServiceConfig = appServiceConfig.replace("#APP_NAME#", deploymentConfig.appname)
                                break
                            }
                            def serviceConfRoot = '/etc/systemd/system/'
                            sh 'sudo chown -R jenkins:jenkins ' + serviceConfRoot
                            writeFile(file: serviceConfRoot + deploymentConfig.appname + '.service', text: appServiceConfig)
                            sh "sudo chown -R root:root /etc/nginx/sites-available/"
                            sh 'if id -u "' + deploymentConfig.appname +'"; then echo "User Already Available"; else sudo useradd -M ' + deploymentConfig.appname + '; fi'
                            sh 'sudo usermod -s /bin/false ' + deploymentConfig.appname
                            sh 'sudo systemctl daemon-reload'
                            sh 'sudo systemctl start ' + deploymentConfig.appname
                            sh 'sudo systemctl status ' + deploymentConfig.appname
                        }
                        def webServerConfigRoot = '/etc/nginx/sites-available/'
                        def nginxConfig = readFile(file: deploymentConfig.nginx_config)
                        nginxConfig = nginxConfig.replace('#APPLICATION_ROOT_FOLDER#', srcFolder)
                        nginxConfig = nginxConfig.replace('#DOMAIN_NAME#', domain)
                        switch(deploymentConfig.application_type) {
                            case 'SPRING_BOOT_APPLICATION':
                                nginxConfig = nginxConfig.replace("#APP_NAME#", deploymentConfig.appname)
                                nginxConfig = nginxConfig.replace("#PORT#", deploymentConfig.port)
                            break
                            case 'PYTHON_APPLICATION':
                                nginxConfig = nginxConfig.replace("#APP_NAME#", deploymentConfig.appname)
                            break
                        }
                        sh 'sudo chown -R jenkins:jenkins ' + webServerConfigRoot
                        writeFile(file: webServerConfigRoot + domain, text: nginxConfig)
                        sh "sudo chown -R root:root /etc/nginx/sites-available/"
                        sh 'sudo ln -sf ' + webServerConfigRoot + domain + ' /etc/nginx/sites-enabled/'
                        sh 'sudo certbot --nginx -n -d ' + domain +' -d www.' + domain
                        sh 'sudo nginx -t'
                        sh 'sudo systemctl restart nginx'
                    }
                    echo 'Deployment to Test Environment Completed'
                }
            }
        }
        stage('Deployment to Production Environment') {
            steps {
                script {
                    echo 'Deployment to Production Environment Started'
                    def webServerTestRoot = '/var/tmp/www/'
                    def webServerRoot = '/var/www/'
                    def gitReference = params.REPOSITORY_GROUP + '/' + params.REPOSITORY_NAME
                    def testSrcFolder = webServerTestRoot + gitReference
                    def srcFolder = webServerRoot + params.REPOSITORY_GROUP
                    if(!fileExists(srcFolder)) {
                        sh 'sudo mkdir -p ' + srcFolder
                    }
                    sh 'sudo cp -r ' + testSrcFolder +' ' + srcFolder
                    sh "sudo chown -R www-data:www-data "+ srcFolder + '/' + params.REPOSITORY_NAME
                    def deploymentConfigFileName = srcFolder + '/' + params.REPOSITORY_NAME +'/deployment-descriptor.yml'
                    def deploymentConfig = readYaml(file: deploymentConfigFileName)
                    def hasDeploymentConfig = deploymentConfig != null;
                    def hasProdDomain = hasDeploymentConfig && deploymentConfig.prod_domain != null && !deploymentConfig.prod_domain.isEmpty()
                    def hasApplicationType = hasDeploymentConfig && deploymentConfig.application_type != null && !deploymentConfig.application_type.isEmpty()
                    if(hasProdDomain && hasApplicationType) {
                        def domain = deploymentConfig.prod_domain
                        echo 'Deploying to '+ domain +' from ' + gitReference
                        def hasApplicationServiceConfigFileName = deploymentConfig.app_service_config != null && !deploymentConfig.app_service_config.isEmpty()
                        if(hasApplicationServiceConfigFileName) {
                            def appServiceConfig = readFile(file: deploymentConfig.app_service_config)
                            appServiceConfig = appServiceConfig.replace("#APPLICATION_ROOT_FOLDER#", srcFolder + '/' + params.REPOSITORY_NAME)
                            appServiceConfig = appServiceConfig.replace("#APP_NAME#", deploymentConfig.appname)
                            appServiceConfig = appServiceConfig.replace("#PORT#", deploymentConfig.port)
                            def serviceConfRoot = '/etc/systemd/system/'
                            sh 'sudo chown -R jenkins:jenkins ' + serviceConfRoot
                            writeFile(file: serviceConfRoot + deploymentConfig.appname + '.service', text: appServiceConfig)
                            sh "sudo chown -R root:root /etc/nginx/sites-available/"
                            sh 'if id -u "' + deploymentConfig.appname +'"; then echo "User Already Available"; else sudo useradd -M ' + deploymentConfig.appname + '; fi'
                            sh 'sudo usermod -s /bin/false ' + deploymentConfig.appname
                            sh 'sudo systemctl daemon-reload'
                            sh 'sudo systemctl start ' + deploymentConfig.appname
                            sh 'sudo systemctl status ' + deploymentConfig.appname
                        }
                        def webServerConfigRoot = '/etc/nginx/sites-available/'
                        def nginxConfig = readFile(file: deploymentConfig.nginx_config)
                        nginxConfig = nginxConfig.replace('#APPLICATION_ROOT_FOLDER#', srcFolder + '/' + params.REPOSITORY_NAME)
                        nginxConfig = nginxConfig.replace('#DOMAIN_NAME#', domain)
                        switch(deploymentConfig.application_type) {
                            case 'SPRING_BOOT_APPLICATION':
                                nginxConfig = nginxConfig.replace("#APP_NAME#", deploymentConfig.appname)
                                nginxConfig = nginxConfig.replace("#PORT#", deploymentConfig.port)
                            break
                            case 'PYTHON_APPLICATION':
                                nginxConfig = nginxConfig.replace("#APP_NAME#", deploymentConfig.appname)
                            break
                        }
                        sh 'sudo chown -R jenkins:jenkins ' + webServerConfigRoot
                        writeFile(file: webServerConfigRoot + domain, text: nginxConfig)
                        sh "sudo chown -R root:root /etc/nginx/sites-available/"
                        sh 'sudo ln -sf ' + webServerConfigRoot + domain + ' /etc/nginx/sites-enabled/'
                        sh 'sudo certbot --nginx -n -d ' + domain +' -d www.' + domain
                        sh 'sudo nginx -t'
                        sh 'sudo systemctl restart nginx'
                    }
                    echo 'Deployment to Production Environment Completed'
                }
            }
        }
    }
}