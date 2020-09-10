pipeline {
    agent any
    tools{
            maven 'maven'
            gradle 'gradle'
    }
    stages {
        //stage('Code Fetch'){
            //steps {
                //checkout([$class: 'GitSCM', branches: [[name: '${git_branch}']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'bb13d13b-8580-4953-99b4-ecf92e6daa90', url: 'git@github.com:BetterXT/pipeline-demo.git']]])
            //}
        //}

        stage('Code Test'){
            steps {
                sh 'gradle clean test'
            }
        }

        stage('Code Build'){
            steps {
                sh 'gradle clean build'
            }
        }

        stage('Deploy Confirm') {
            steps {
                timeout(time: 1, unit:'DAYS') {
                    input 'Code Deploy?'
                }
            }
        }

        
    }
}
