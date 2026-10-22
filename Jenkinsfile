// Jenkinsfile voor dotnet-demo-app (stap 1.9) — zet in de root van je fork.
// Geen docker compose: de compose-plugin zit niet in de Jenkins-container,
// dus doen we hetzelfde met gewone docker-commando's.

pipeline {
    agent any

    // ⭐ CD: zelfde idee als bij de sample app
    triggers {
        pollSCM('H/2 * * * *')
    }

    environment {
        NET      = 'todo_net'
        DB_NAME  = 'todoappdb'
        APP_NAME = 'todoapp'
        // Poort 8081 op de VM: 8080 is al bezet door Jenkins!
        APP_PORT = '8081'
        CONN     = 'Server=todoappdb;Port=3306;Database=todo_db;User=todo_usr;Password=letmeinplz;'
    }

    stages {
        stage('Preparation') {
            steps {
                // Eigen netwerk: app vindt DB via containernaam (Docker DNS),
                // werkt niet op het default bridge-netwerk.
                sh 'docker network inspect $NET >/dev/null 2>&1 || docker network create $NET'
            }
        }

        stage('Database') {
            steps {
                // DB blijft draaien tussen builds (data in volume mariadb-data).
                // schema.sql NIET bind-mounten: het $WORKSPACE-pad bestaat enkel
                // IN de Jenkins-container, niet op de VM waar de Docker daemon
                // draait. Daarom: create -> cp -> start.
                sh '''
                    if [ -z "$(docker ps -q -f name=^${DB_NAME}$)" ]; then
                        docker rm -f $DB_NAME 2>/dev/null || true
                        docker create --name $DB_NAME --network $NET \
                          -e MARIADB_ROOT_PASSWORD=sekrit \
                          -e MARIADB_DATABASE=todo_db \
                          -e MARIADB_USER=todo_usr \
                          -e MARIADB_PASSWORD=letmeinplz \
                          -v mariadb-data:/var/lib/mysql \
                          --health-cmd "healthcheck.sh --connect --innodb_initialized" \
                          --health-interval 5s --health-retries 20 \
                          mariadb:11
                        docker cp TodoApp/schema.sql $DB_NAME:/docker-entrypoint-initdb.d/schema.sql
                        docker start $DB_NAME
                    fi

                    # Wachten tot de DB "healthy" is (= depends_on/service_healthy uit compose)
                    for i in $(seq 1 30); do
                        STATUS=$(docker inspect -f '{{.State.Health.Status}}' $DB_NAME)
                        echo "DB status: $STATUS"
                        [ "$STATUS" = "healthy" ] && exit 0
                        sleep 3
                    done
                    echo "DB niet healthy geraakt"; exit 1
                '''
            }
        }

        stage('Build') {
            steps {
                // docker build mag wél met workspace-pad: de CLI stuurt de
                // build context zelf door naar de daemon.
                sh 'docker build -t todoapp:latest TodoApp'
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker rm -f $APP_NAME 2>/dev/null || true
                    docker run -d --name $APP_NAME --network $NET \
                      -p ${APP_PORT}:8080 \
                      -e ConnectionStrings__TodoDb="$CONN" \
                      -e ASPNETCORE_ENVIRONMENT=Development \
                      todoapp:latest
                '''
            }
        }

        stage('Test') {
            steps {
                // Via host-IP + gepubliceerde poort, zoals een gebruiker.
                // curl -f faalt bij HTTP 500 (bv. DB-verbinding stuk).
                sh '''
                    for i in $(seq 1 15); do
                        if curl -fsS http://172.16.0.10:${APP_PORT}/ | grep -q "Todo List"; then
                            echo "TEST OK: Todo List pagina bereikbaar"; exit 0
                        fi
                        echo "Poging $i mislukt, 2s wachten..."; sleep 2
                    done
                    echo "TEST FAALT"; docker logs --tail 50 $APP_NAME; exit 1
                '''
            }
        }
    }
}
