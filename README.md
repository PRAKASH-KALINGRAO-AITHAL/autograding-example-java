name: CI Pipeline

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: testdb
          MYSQL_USER: testuser
          MYSQL_PASSWORD: testpass
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping -h localhost -u root --password=root" --health-interval=10s --health-timeout=5s --health-retries=5

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set Up JDK 17
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Wait for MySQL to be Ready
        run: |
          for i in {1..30}; do
            if mysqladmin ping -h 127.0.0.1 -u root --password=root --silent; then
              echo "MySQL is ready!"
              exit 0
            fi
            echo "Waiting for MySQL..."
            sleep 2
          done
          echo "MySQL failed to start"
          exit 1

      - name: Set Up Database Schema
        run: |
          mysql -h 127.0.0.1 -u root --password=root -e "CREATE DATABASE IF NOT EXISTS testdb;"

      - name: Run Tests
        run: mvn test
        env:
          SPRING_DATASOURCE_URL: jdbc:mysql://127.0.0.1:3306/testdb
          SPRING_DATASOURCE_USERNAME: testuser
          SPRING_DATASOURCE_PASSWORD: testpass

