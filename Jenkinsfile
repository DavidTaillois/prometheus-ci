#!groovy
def ci_server = 'dc1-mon-dev-prom-ci.dev.dc1.kelkoo.net'

def prometheus_server
if (gitlabSourceRepoName == "prometheus-server-dev-config")  {
    prometheus_server = ['dc1-mon-dev-prom-01.dev.dc1.kelkoo.net','dc1-mon-dev-prom-02.dev.dc1.kelkoo.net']
} else if(gitlabSourceRepoName == "prometheus-server-prod-config") {
    prometheus_server = ['dc1-mon-prod-prom-01.prod.dc1.kelkoo.net','dc1-mon-prod-prom-02.prod.dc1.kelkoo.net']
} else {
    prometheus_server = "unknown"
}

def send_discord_notif() {

    discordSend(description: "${currentBuild.currentResult} for ${gitlabSourceRepoName}\nBuild: ${env.BUILD_NUMBER} \nMore info at: \n${env.BUILD_URL}", footer: "Gitlab: ${gitlabSourceRepoHttpUrl}", unstable: true, link: env.BUILD_URL, result: "${currentBuild.currentResult}", title: "${JOB_NAME}", webhookURL: 'https://discord.com/api/webhooks/887302475709829121/zWSO4V2paQ1TNI6fooqfZj2NuT-QaWFy8z_y5UG_WJI8yk0agVRTeeGUoW3OYJn5Y_qT')

}

pipeline {
    agent any
    environment {
        KOOKEL_CREDS = credentials('f0ebc6fa-410a-4207-8636-f267da6891ae')
    }
    stages {
        stage('Hello') {
            steps {
                echo "Git repo: ${gitlabSourceRepoName}"
                echo "Prometheus Servers: ${prometheus_server}"
            }
        }
        stage('Build') {
            steps {
                sh '''
                    ssh -i $KOOKEL_CREDS $KOOKEL_CREDS_USR@'''+ci_server+''' 'bash -s << 'ENDSSH'
                    if [[ -d "/tmp/prometheus" ]];
                    then
                      rm -rf /tmp/prometheus
                    fi
                    git clone '''+gitlabSourceRepoHttpUrl+''' /tmp/prometheus
ENDSSH'
                '''
            }           
        }
        stage('Lint of the code') {
            steps {
                sh '''
                    ssh -i $KOOKEL_CREDS $KOOKEL_CREDS_USR@'''+ci_server+''' 'bash -s  << 'ENDSSH'
                    cd /tmp/prometheus
                    yamllint *
ENDSSH'
                '''
                sh '''
                    ssh -i $KOOKEL_CREDS $KOOKEL_CREDS_USR@'''+ci_server+''' 'bash -s  << 'ENDSSH'
                    sudo promtool check config /tmp/prometheus/prometheus.yml
ENDSSH'
                '''
            }
        }
        stage('Integration Test') {
            steps {
                sh '''
                    ssh -i $KOOKEL_CREDS $KOOKEL_CREDS_USR@'''+ci_server+''' 'bash -s << 'ENDSSH'
                    sudo rm -rf /etc/prometheus/.git
                    sudo cp -rf /tmp/prometheus/.git /etc/prometheus/
                    rm -rf /tmp/prometheus
                    cd /etc/prometheus
                    sudo git checkout -f master
                    sudo git clean -fd
ENDSSH'
                '''
                sh '''
                    ssh -i $KOOKEL_CREDS $KOOKEL_CREDS_USR@'''+ci_server+''' 'bash -s << 'ENDSSH'
                    cd /etc/prometheus
                    sudo promtool check config prometheus.yml
                    sudo systemctl restart prometheus.service
ENDSSH'
                '''
                sh 'curl '+ci_server+':9090/graph'
            }
        }
        stage('Deployment') {
            steps {
                script {
                    for (server in prometheus_server) {
                        sh '''
                            ssh -i $KOOKEL_CREDS $KOOKEL_CREDS_USR@'''+server+''' 'bash -s << 'ENDSSH'
                            if [[ -d "/tmp/prometheus" ]];
                            then
                              rm -rf /tmp/prometheus
                            fi                            
                            git clone '''+gitlabSourceRepoHttpUrl+''' /tmp/prometheus
                            sudo rm -rf /etc/prometheus/.git
                            sudo cp -rf /tmp/prometheus/.git /etc/prometheus/
                            cd /etc/prometheus
                            sudo git checkout -f master
                            sudo git clean -fd
ENDSSH'
                        '''
                    }
                }
                script {
                    for (server in prometheus_server) {
                        sh '''
                            ssh -i $KOOKEL_CREDS $KOOKEL_CREDS_USR@'''+server+''' 'bash -s << 'ENDSSH'
                            cd /etc/prometheus
                            sudo promtool check config prometheus.yml
                            sudo systemctl reload prometheus.service
ENDSSH'
                        '''
                    }
                }
                script {
                    prometheus_server.each { server ->
                        sh "curl ${server}:9090/graph"
                    }
                }
            }
        }
    }
    post {
        always {
            send_discord_notif()
        }
    }
}
