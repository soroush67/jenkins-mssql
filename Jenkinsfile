// Jenkins Pipeline job: "Pipeline script from SCM" pointing at this
// repo, with Script Path = Jenkinsfile.
//
// "Deploy to any destination" = set TARGET_HOST to any IP/hostname on
// every run - it does NOT need to already exist in inventory/hosts.ini.
// playbooks/deploy.yml and status.yml dynamically register it into the
// mssql inventory group for that one run (see their own comments).
//
// One-time Jenkins setup required before this works:
//   1. Set AGENT_NODE_LABEL below to the real label of the agent node
//      that has docker + (docker compose plugin OR the standalone
//      docker-compose binary - roles/preflight detects whichever is
//      present) + ansible-core installed, and has its OS user in the
//      `docker` group.
//   2. Create two "Secret text" credentials - one for the SA password
//      (ID below: SA_PASSWORD_CREDENTIALS_ID) - must meet SQL Server's
//      own password policy (at least 8 characters, 3 of {uppercase,
//      lowercase, digit, symbol}); roles/preflight checks this and
//      fails cleanly if it doesn't - and one for the mssql_exporter
//      login's password (EXPORTER_PASSWORD_CREDENTIALS_ID). Both are
//      required - roles/preflight refuses to deploy with either one
//      empty.
//   3. inventory/hosts.ini's placeholder localhost entry never needs to
//      be touched for real deployments - just set TARGET_HOST per run.

def AGENT_NODE_LABEL = 'inf-18-jenk-3'
def SA_PASSWORD_CREDENTIALS_ID = 'SA_PASSWORD_CREDENTIALS_ID'
def EXPORTER_PASSWORD_CREDENTIALS_ID = 'EXPORTER_PASSWORD_CREDENTIALS_ID'

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
            description: 'deploy: idempotent MSSQL deploy (safe to rerun). status: read-only health/connectivity check.'
        )
        string(
            name: 'TARGET_HOST',
            defaultValue: '',
            description: 'IP or hostname to deploy to - any reachable destination, does NOT need to be pre-added to inventory/hosts.ini. Leave blank to use whatever is in inventory/hosts.ini instead (local/manual testing).'
        )
        string(
            name: 'DEPLOY_PATH',
            defaultValue: '',
            description: 'Absolute path on TARGET_HOST where docker-compose.yml gets created. The actual data VOLUME lands separately, under /data/<the last folder of this path> (e.g. DEPLOY_PATH=/opt/servers/mssql-prod-1 -> volume at /data/mssql-prod-1). Leave blank to use the Jenkins workspace checkout directory instead (local/manual testing).'
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
                DEPLOY_PATH = "${params.DEPLOY_PATH}"
            }
            steps {
                withCredentials([
                    string(credentialsId: SA_PASSWORD_CREDENTIALS_ID, variable: 'SA_PASSWORD'),
                    string(credentialsId: EXPORTER_PASSWORD_CREDENTIALS_ID, variable: 'EXPORTER_PASSWORD'),
                ]) {
                    sh '''
                        set -e
                        TARGET_HOST_ARG=""
                        [ -n "$TARGET_HOST" ] && TARGET_HOST_ARG="-e target_host=$TARGET_HOST"
                        DEPLOY_PATH_ARG=""
                        [ -n "$DEPLOY_PATH" ] && DEPLOY_PATH_ARG="-e mssql_deploy_path=$DEPLOY_PATH"
                        ansible-playbook playbooks/deploy.yml \
                          $TARGET_HOST_ARG \
                          $DEPLOY_PATH_ARG \
                          -e mssql_sa_password="$SA_PASSWORD" \
                          -e mssql_exporter_password="$EXPORTER_PASSWORD"
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
                withCredentials([
                    string(credentialsId: SA_PASSWORD_CREDENTIALS_ID, variable: 'SA_PASSWORD'),
                ]) {
                    sh '''
                        set -e
                        TARGET_HOST_ARG=""
                        [ -n "$TARGET_HOST" ] && TARGET_HOST_ARG="-e target_host=$TARGET_HOST"
                        ansible-playbook playbooks/status.yml $TARGET_HOST_ARG -e mssql_sa_password="$SA_PASSWORD"
                    '''
                }
            }
        }
    }
}
