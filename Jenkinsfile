// Jenkins Pipeline job: "Pipeline script from SCM" pointing at this
// repo, with Script Path = Jenkinsfile.
//
// "Deploy to any destination" = set TARGET_HOST to any IP/hostname on
// every run - it does NOT need to already exist in inventory/hosts.ini.
// playbooks/deploy.yml and status.yml dynamically register it into the
// postgres inventory group for that one run (see their own comments for
// why this isn't just `-i "<ip>,"` or `--limit`).
//
// One-time Jenkins setup required before this works:
//   1. Set AGENT_NODE_LABEL below to the real label of the agent node
//      that has docker + (docker compose plugin OR the standalone
//      docker-compose binary - roles/preflight detects whichever is
//      present) + ansible-core installed, and has its OS user in the
//      `docker` group.
//   2. Create two "Secret text" credentials - one for the appuser
//      password (ID below: PG_PASSWORD_CREDENTIALS_ID), one for the
//      postgres_exporter password (PG_EXPORTER_PASSWORD_CREDENTIALS_ID).
//      Both are required - roles/preflight refuses to deploy with either
//      one empty.
//   3. inventory/hosts.ini's placeholder localhost entry never needs to
//      be touched for real deployments - just set TARGET_HOST per run.
//      It still matters as the fallback used when TARGET_HOST is left
//      blank (local/manual testing).

def AGENT_NODE_LABEL = 'CHANGE_ME_POSTGRES_AGENT_LABEL'
def PG_PASSWORD_CREDENTIALS_ID = 'pg-stack-appuser-password'
def PG_EXPORTER_PASSWORD_CREDENTIALS_ID = 'pg-stack-exporter-password'

pipeline {
    agent { label AGENT_NODE_LABEL }

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    parameters {
        choice(
            name: 'ACTION',
            choices: ['deploy', 'status'],
            description: 'deploy: idempotent PostgreSQL + postgres_exporter deploy (safe to rerun). status: read-only health/connectivity check.'
        )
        string(
            name: 'TARGET_HOST',
            defaultValue: '',
            description: 'IP or hostname to deploy to - any reachable destination, does NOT need to be pre-added to inventory/hosts.ini. Leave blank to use whatever is in inventory/hosts.ini instead (local/manual testing).'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Deploy') {
            when { expression { params.ACTION == 'deploy' } }
            environment {
                TARGET_HOST = "${params.TARGET_HOST}"
            }
            steps {
                withCredentials([
                    string(credentialsId: PG_PASSWORD_CREDENTIALS_ID, variable: 'PG_PASSWORD'),
                    string(credentialsId: PG_EXPORTER_PASSWORD_CREDENTIALS_ID, variable: 'PG_EXPORTER_PASSWORD'),
                ]) {
                    sh '''
                        set -e
                        TARGET_HOST_ARG=""
                        [ -n "$TARGET_HOST" ] && TARGET_HOST_ARG="-e target_host=$TARGET_HOST"
                        ansible-playbook playbooks/deploy.yml \
                          $TARGET_HOST_ARG \
                          -e pg_password="$PG_PASSWORD" \
                          -e pg_exporter_password="$PG_EXPORTER_PASSWORD"
                    '''
                }
            }
        }

        stage('Status') {
            when { expression { params.ACTION == 'status' } }
            environment {
                TARGET_HOST = "${params.TARGET_HOST}"
            }
            steps {
                sh '''
                    set -e
                    TARGET_HOST_ARG=""
                    [ -n "$TARGET_HOST" ] && TARGET_HOST_ARG="-e target_host=$TARGET_HOST"
                    ansible-playbook playbooks/status.yml $TARGET_HOST_ARG
                '''
            }
        }
    }
}
