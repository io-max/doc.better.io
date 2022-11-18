pipeline {
    agent any
    environment {
        JAVA_HOME = "/opt/jdk/jdk8u312-b07"
        APPLICATION_NAME = "doc.io-better.cn"
    }
    options {
        skipDefaultCheckout true
        buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '7', numToKeepStr: '2')
        disableConcurrentBuilds abortPrevious: true
    }

    stages {
        stage('拉取代码') {
            steps {
                echo 'pull code form gitlab start'
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: 'gogs', url: 'https://code.chenmc.cn/doc/doc.io.better.git']]])
                echo 'pull code form gitlab end'
            }
        }
        stage('安装依赖') {
            tools {
                nodejs 'nodejs'
            }
            steps {
                sh 'npm --registry https://registry.npm.taobao.org install'
            }
        }
        stage('构建') {
            tools {
                nodejs 'nodejs'
            }
            steps {
                sh 'hexo clean'
                sh 'hexo generate'
                sh 'pwd'
            }
        }
        stage('备份') {
            steps {
                sh 'rm -rf /www/wwwroot/www.chenmc.cn.back'
                sh 'mv /www/wwwroot/www.chenmc.cn /www/wwwroot/www.chenmc.cn.back'
            }
        }
        stage('部署') {
            tools {
                nodejs 'nodejs'
            }
            steps {
                sh 'mv public www.chenmc.cn'
                sh 'mv -f -b www.chenmc.cn /www/wwwroot/'
            }
        }
    }
    post {
        success {
            dingtalk(
                robot: 'Jenkins-DingDing',
                type: 'LINK',
                title:'${JOB_BASE_NAME}#${BUILD_ID} 构建成功',
                text: ["${JOB_BASE_NAME}#${BUILD_ID} 构建成功"],
                messageUrl: '${BUILD_URL}',
                picUrl: 'https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200918180559.png',
            )
        }
        failure {
            dingtalk(
                robot: 'Jenkins-DingDing',
                type: 'LINK',
                title: '${JOB_BASE_NAME}#${BUILD_ID} 构建失败',
                text: ["${JOB_BASE_NAME}#${BUILD_ID} 构建失败"],
                messageUrl: '${BUILD_URL}',
                picUrl: 'https://better-io-blog.oss-cn-beijing.aliyuncs.com/20200918180603.png',
            )
        }
    }
}
