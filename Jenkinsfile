node {
  //
  def pullRequest = false
  def repo = ''
  def org = ''
  def groupId = ''
  def artifactId = ''
  def version = ''
  //
  echo "SONARQUBE_SERVER: ${SONARQUBE_SERVER}"
  echo "SONARQUBE_SCANNER: ${SONARQUBE_SCANNER}"
  echo "SONARQUBE_ACCESS_TOKEN: ${SONARQUBE_ACCESS_TOKEN}"
  echo "GITHUB_ACCESS_TOKEN: ${GITHUB_ACCESS_TOKEN}"
  echo "NEXUS_HOST: ${NEXUS_HOST}"
  echo "NEXUS_REPO: ${NEXUS_REPO}"
  echo "NEXUS_USER: ${NEXUS_USER}"
  echo "NEXUS_PASS: ${NEXUS_PASS}"
  echo "SERVICES_GJ_PORT: ${SERVICES_GJ_PORT}" 
  
  stage('Clone sources') {
    //
    def scmVars = checkout scm
    if (params.containsKey('sha1')){
      pullRequest = true
      echo "Pull request build sha1: ${sha1}"
      sh "git fetch --tags --progress origin +refs/pull/*:refs/remotes/origin/pr/*"
      sh "git checkout ${ghprbActualCommit}"
    } else {
      echo "Build push branch: ${scmVars.GIT_BRANCH}, sha: ${scmVars.GIT_COMMIT}"
      sh "git checkout ${scmVars.GIT_COMMIT}"
    }
    echo sh(returnStdout: true, script: 'env')
    //
    dir('services/grizzly-jersey'){
      groupId = sh(returnStdout: true, script:'''mvn help:evaluate -Dexpression=project.groupId | grep -e "^[^\\[]"''').trim()
      artifactId = sh(returnStdout: true, script:'''mvn help:evaluate -Dexpression=project.artifactId | grep -e "^[^\\[]"''').trim()
      version = sh(returnStdout: true, script:'''mvn help:evaluate -Dexpression=project.version | grep -e "^[^\\[]"''').trim()
    }
    //
    repo = sh(returnStdout: true, script:'''git config --get remote.origin.url | rev | awk -F'[./:]' '{print $2}' | rev''').trim()
    org = sh(returnStdout: true, script:'''git config --get remote.origin.url | rev | awk -F'[./:]' '{print $3}' | rev''').trim()
    //
    echo "groupId: ${groupId} artifactId: ${artifactId} version: ${version}"
    echo "org: ${org} repo: ${repo}"
    //
    sh "envsubst < .env.template > .env"
    sh "cat ./.env"
    sh "envsubst < settings.xml.template > settings.xml"
    sh "cat ./settings.xml"
  }  
  //
  stage('Build & Unit tests') {
    sh './down.sh || echo "already stop"'
    sh './build.sh'
  }
  //
  stage('SonarQube analysis') {
    def scannerHome = tool "${SONARQUBE_SCANNER}"
    withSonarQubeEnv("${SONARQUBE_SERVER}") {
      if (pullRequest){
        sh "${scannerHome}/bin/sonar-scanner -Dsonar.analysis.mode=preview -Dsonar.github.pullRequest=${ghprbPullId} -Dsonar.github.repository=${org}/${repo} -Dsonar.github.oauth=${GITHUB_ACCESS_TOKEN} -Dsonar.login=${SONARQUBE_ACCESS_TOKEN}"
      } else {
        sh "${scannerHome}/bin/sonar-scanner"
        // check SonarQube Quality Gates
        //// Pipeline Utility Steps
        def props = readProperties  file: '.scannerwork/report-task.txt'
        echo "properties=${props}"
        def sonarServerUrl=props['serverUrl']
        def ceTaskUrl= props['ceTaskUrl']
        def ceTask
        //// HTTP Request Plugin
        timeout(time: 1, unit: 'MINUTES') {
          waitUntil {
            def response = httpRequest "${ceTaskUrl}"
            println('Status: '+response.status)
            println('Response: '+response.content)
            ceTask = readJSON text: response.content
            return (response.status == 200) && ("SUCCESS".equals(ceTask['task']['status']))
          }
        }
        //
        def qgResponse = httpRequest sonarServerUrl + "/api/qualitygates/project_status?analysisId=" + ceTask['task']['analysisId']
        def qualitygate = readJSON text: qgResponse.content
        echo qualitygate.toString()
        if ("ERROR".equals(qualitygate["projectStatus"]["status"])) {
          currentBuild.description = "Quality Gate failure"
          error currentBuild.description
        }
      }
    }
  }
  //
  //
  stage('Publish') {
    if (pullRequest){
    } else {
      sh "./upload.sh ${groupId} ${artifactId} ${version} ./services/grizzly-jersey/target"
    }
    //archiveArtifacts artifacts: 'mobile/platforms/android/build/outputs/apk/*.apk'
  }
  //
  stage('Deploy to QA') {
    sh './up.sh -d'
  }

  stage("Integration testing") {

    git url: 'https://github.com/auto-qa-course/python-framework-example.git', branch: 'lesson3'

    sh('''
        rm out_docker_results/**/*.json || echo 'no json files'

        docker build -t automation_demo .

        mkdir out_docker_results || echo 'out_docker_results dir exists'

        docker run -v $(pwd)/out_docker_results:/out:rw -e "ENVIRONMENT=$QA" -e -i automation_demo /bin/bash -c "pytest test/api/test_healthcheck.py --alluredir /out"

        docker run -v $(pwd)/out_docker_results:/out:rw -e "ENVIRONMENT=$QA" -e -i automation_demo /bin/bash -c "pytest test/api/contacts --alluredir /out"
    ''')

    allure includeProperties: false, jdk: '', results: [[path: 'out_docker_results']]
  }

  stage('Request approval for deploy to Stage') {

    script {
        timeout(time:10, unit:'MINUTES') {
            while (true) {
                userPasswordInput = input(id: 'userPasswordInput',
                    message: 'Please approve deploy to Stage. Enter password to proceed.',
                    parameters: [[$class: 'StringParameterDefinition', defaultValue: '',  name: 'Password']])
                if (userPasswordInput=='Yes') { break }
            }
        }
    }

  }

  stage('Deploy to Stage') {
    // TBD
  }

}
