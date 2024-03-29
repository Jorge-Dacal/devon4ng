pipeline{
    agent any

    options {
        buildDiscarder(logRotator(artifactDaysToKeepStr: '1', artifactNumToKeepStr: '1', daysToKeepStr: '5', numToKeepStr: '50'))
        // Disable concurrent builds. It will wait until the pipeline finish before start a new one
        disableConcurrentBuilds()
    }

    tools {
        nodejs "NodeJS 10.14.0"
        
        oc "OpenShiftv3.11.0"
        
    }

    environment {
        // Script for build the application. Defined at package.json
        buildScript = 'build'
        // Script for lint the application. Defined at package.json
        lintScript = 'lint'
        // Script for test the application. Defined at package.json
        testScript = 'test:ci'
        // SRC folder.
        srcDir = 'src'
        // Name of the custom tool for chrome stable
        chrome = 'Chrome-stable'

        // sonarQube
        // Name of the sonarQube tool
        sonarTool = 'SonarQube-scanner'
        // Name of the sonarQube environment
        sonarEnv = "SharesSonar"

        // Nexus
        // Artifact groupId
        groupId = 'com.capgemini.octest'
        // Nexus repository ID
        repositoryId = 'nexus'
        // Nexus internal URL
        repositoryUrl = 'http://nexus3-core:8081/nexus3/repository/'
        // Maven global settings configuration ID
        globalSettingsId = 'MavenSettings'
        // Maven tool id
        mavenInstallation = 'Maven3'

        
        // Docker
        dockerFileName = 'Dockerfile.ci'
        dockerRegistry = 'docker-registry-shared-services.pl.s2-eu.capgemini.com'
        dockerRegistryCredentials = 'nexusDeployer'
        
        

        
        // Openshift
        openshiftUrl = 'https://ocp.itaas.s2-eu.capgemini.com'
        openShiftCredentials = 'ocadmin'
        openShiftNamespace = 's2portaldev'
        
    }

    stages {
        stage ('Loading Custom Tools') {
            when {
               anyOf {
                   branch 'master'
                   branch 'develop'
                   branch 'release/*'
                   branch 'feature/*'
                   branch 'hotfix/*'
                   changeRequest()
               }
            }
            steps {
                tool chrome
                


                script {
                    if (env.BRANCH_NAME.startsWith('release')) {
                        dockerTag = "release"
                        repositoryName = 'maven-releases'
                        dockerEnvironment = "-uat"
                        sonarProjectKey = '-release'
                    }

                    if (env.BRANCH_NAME == 'develop') {
                        dockerTag = "latest"
                        repositoryName = 'maven-snapshots'
                        dockerEnvironment = "-dev"
                        sonarProjectKey = '-develop'
                    }

                    if (env.BRANCH_NAME == 'master') {
                        dockerTag = "production"
                        repositoryName = 'maven-releases'
                        dockerEnvironment = '-prod'
                        sonarProjectKey = ''
                        
                    }
                }
            }
        }

        stage ('Fresh Dependency Installation') {
            when {
                anyOf {
                    branch 'master'
                    branch 'develop'
                    branch 'release/*'
                    branch 'feature/*'
                    branch 'hotfix/*'
                    changeRequest()
                }
            }
            steps {
                sh "yarn"
            }
        }

        stage ('Code Linting') {
            when {
                anyOf {
                    branch 'master'
                    branch 'develop'
                    branch 'release/*'
                    branch 'feature/*'
                    branch 'hotfix/*'
                    changeRequest()
                }
            }
            steps {
                sh """yarn ${lintScript}"""
            }
        }

        stage ('Execute Angular tests') {
            when {
                anyOf {
                    branch 'master'
                    branch 'develop'
                    branch 'release/*'
                    branch 'feature/*'
                    branch 'hotfix/*'
                    changeRequest()
                }
            }
            steps {
                sh """yarn ${testScript}"""
            }
        }

        stage ('SonarQube code analysis') {
            when {
                anyOf {
                    branch 'master'
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps {
                script {
                    def scannerHome = tool sonarTool
                    def props = readJSON file: 'package.json'
                    withSonarQubeEnv(sonarEnv) {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=${props.name}${sonarProjectKey} \
                                -Dsonar.projectName=${props.name}${sonarProjectKey} \
                                -Dsonar.projectVersion=${props.version} \
                                -Dsonar.sources=${srcDir} \
                                -Dsonar.typescript.lcov.reportPaths=coverage/lcov.info
                        """
                    }
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage ('Build Application') {
            when {
                anyOf {
                    branch 'master'
                    branch 'develop'
                    branch 'release/*'
                    branch 'feature/*'
                    branch 'hotfix/*'
                    changeRequest()
                }
            }
            steps {
                sh """
                    yarn ${buildScript}
                    cp ${dockerFileName} dist/Dockerfile
                    cp nginx.conf dist/nginx.conf
                """
            }
        }

        stage ('Deliver application into Nexus') {
            when {
                anyOf {
                    branch 'master'
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps {
                script {
                    def props = readJSON file: 'package.json'
                    zip dir: 'dist/', zipFile: """${props.name}.zip"""
                    version = props.version
                    if (!version.endsWith("-SNAPSHOT") && env.BRANCH_NAME == 'develop') {
                        version = "${version}-SNAPSHOT"
                        version = version.replace("-RC", "")
                    }

                    if (!version.endsWith("-RC") && env.BRANCH_NAME.startsWith('release')) {
                        version = "${version}-RC"
                        version = version.replace("-SNAPSHOT", "")
                    }

                    if (env.BRANCH_NAME == 'master' && (version.endsWith("-RC") || version.endsWith("-SNAPSHOT"))){
                        version = version.replace("-RC", "")
                        version = version.replace("-SNAPSHOT", "")
                    }

                    withMaven(globalMavenSettingsConfig: globalSettingsId, maven: mavenInstallation) {
                        sh """
                            mvn deploy:deploy-file \
                                -DgroupId=${groupId} \
                                -DartifactId=${props.name} \
                                -Dversion=${version} \
                                -Dpackaging=zip \
                                -Dfile=${props.name}.zip \
                                -DrepositoryId=${repositoryId} \
                                -Durl=${repositoryUrl}${repositoryName}
                        """
                    }
                }
            }
        }
        
        

        
        stage ('Create the Docker image') {
            when {
                anyOf {
                    branch 'master'
                    branch 'develop'
                    branch 'release/*'

                }
            }
            steps {
                script {
                    props = readJSON file: 'package.json'
                    dir('dist') {
                        withCredentials([usernamePassword(credentialsId: "${openShiftCredentials}", passwordVariable: 'pass', usernameVariable: 'user')]) {
                        
                            sh "oc login -u ${user} -p ${pass} ${openshiftUrl} --insecure-skip-tls-verify"
                            try {
                                sh "oc start-build ${props.name}${dockerEnvironment} --namespace=${openShiftNamespace} --from-dir=. --wait"
                            } catch (e) {
                                sh """
                                    oc logs \$(oc get builds -l build=${props.name}${dockerEnvironment} --namespace=${openShiftNamespace} --sort-by=.metadata.creationTimestamp -o name | tail -n 1) --namespace=${namespace}
                                    throw e
                                """
                            }
                        }
                    }
                }
            }
        }

        stage ('Deploy the new image') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps{
                script {
                    props = readJSON file: 'package.json'
                    withCredentials([usernamePassword(credentialsId: "${openShiftCredentials}", passwordVariable: 'pass', usernameVariable: 'user')]) {
                        sh "oc login -u ${user} -p ${pass} ${openshiftUrl} --insecure-skip-tls-verify"
                        try {
                            sh "oc import-image ${props.name}${dockerEnvironment}:${dockerTag} --namespace=${openShiftNamespace} --from=${dockerRegistry}/${props.name}:${dockerTag} --confirm"
                        } catch (e) {
                            sh """
                                oc logs \$(oc get builds -l build=${props.name}${dockerEnvironment} --namespace=${openShiftNamespace} --sort-by=.metadata.creationTimestamp -o name | tail -n 1) --namespace=${openShiftNamespace}
                                throw e
                            """
                        }
                    }
                }
            }
        }
        
        stage ('Check pod status') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps{
                script {
                    props = readJSON file: 'package.json'
                    sleep 60
                    withCredentials([usernamePassword(credentialsId: "${openShiftCredentials}", passwordVariable: 'pass', usernameVariable: 'user')]) {
                        sh "oc login -u ${user} -p ${pass} ${openshiftUrl} --insecure-skip-tls-verify"
                        sh "oc project ${openShiftNamespace}"
                        
                        def oldRetry = -1;
                        def oldState = "";
                        
                        sh "oc get pods -l app=${props.name}${dockerEnvironment} > out"
                        def status = sh (
                            script: "sed 's/[\t ][\t ]*/ /g' < out | sed '2q;d' | cut -d' ' -f3",
                            returnStdout: true
                        ).trim()
                        
                        def retry = sh (
                            script: "sed 's/[\t ][\t ]*/ /g' < out | sed '2q;d' | cut -d' ' -f4",
                            returnStdout: true
                        ).trim().toInteger();
                        
                        while (retry < 5 && (oldRetry != retry || oldState != status)) {
                            sleep 30
                            oldRetry = retry
                            oldState = status
                            
                            sh """oc get pods -l app=${props.name}${dockerEnvironment} > out"""
                            status = sh (
                                script: "sed 's/[\t ][\t ]*/ /g' < out | sed '2q;d' | cut -d' ' -f3",
                                returnStdout: true
                            ).trim()
                            
                            retry = sh (
                                script: "sed 's/[\t ][\t ]*/ /g' < out | sed '2q;d' | cut -d' ' -f4",
                                returnStdout: true
                            ).trim().toInteger();
                        }
                        
                        if(status != "Running"){
                            try {
                                sh """oc logs \$(oc get pods -l app=${props.name}${dockerEnvironment} --sort-by=.metadata.creationTimestamp -o name | tail -n 1)"""
                            } catch (e) {
                                sh "echo error reading logs"
                            }
                            error("The pod is not running, cause: " + status)
                        }
                    }
                }
            }
        }
        
    }

    post {
        cleanup {
            cleanWs()
        }
    }
}
