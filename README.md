name: Master Branch

on:
  push:
    branches:
      - 'master'

jobs:

  test:
    name: Test - Units & Integrations
    runs-on: ubuntu-18.04

    steps:
      - uses: actions/checkout@v1
      - name: Set up JDK 11
        uses: actions/setup-java@v1
        with:
          java-version: 11.0.4
      - name: Maven Package
        run: mvn -B clean package -DskipTests
      - name: Maven Verify
        run: mvn -B clean verify -Pintegration-test

  sonar:
    name: Test - SonarCloud Scan
    runs-on: ubuntu-18.04

    steps:
      - uses: actions/checkout@v1
      - name: Set up JDK 11
        uses: actions/setup-java@v1
        with:
          java-version: 11.0.4
      - name: SonarCloud Scan
        run: mvn -B clean verify -Psonar -Dsonar.login=${{ secrets.SONAR_TOKEN }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          
  artifact:
    name: Publish - GitHub Packages
    runs-on: ubuntu-18.04
    needs: [test, sonar]

    steps:
      - uses: actions/checkout@v1
      - name: Set up JDK 11
        uses: actions/setup-java@v1
        with:
          java-version: 11.0.4
      - name: Publish artifact on GitHub Packages
        run: mvn -B clean deploy -DskipTests
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}


  docker:
    name: Publish - Docker Hub
    runs-on: ubuntu-18.04
    needs: [test, sonar]
    env:
      REPO: ${{ secrets.DOCKER_REPO }}

    steps:
      - uses: actions/checkout@v1
      - name: Set up JDK 11
        uses: actions/setup-java@v1
        with:
          java-version: 11.0.4
      - name: Login to Docker Hub
        run: docker login -u ${{ secrets.DOCKER_USER }} -p ${{ secrets.DOCKER_PASS }}
      - name: Build Docker image
        run: docker build -t $REPO:latest -t $REPO:${GITHUB_SHA::8} .
      - name: Publish Docker image
        run: docker push $REPO
