# Fundamental DevOps Project
 
## Backgroud
Salah satu tantangan sebagai DevOps Engineer adalah bisa melakukan deployment project dengan baik. Pada project kali ini anda akan mendeploy sebuah aplikasi web “PacMail”. 


## Program Description
#### Dockerfile dan Docker Compose 
~~~
FROM python:3.10

WORKDIR /app

EXPOSE 5000

# Install pip requirements
COPY . /app
RUN python -m pip install -r requirements.txt

CMD ["python", "api.py"]
~~~

Menggunakan python versi 3.10 sebagai base image. (FROM python:3.10)

Menggunakan /app sebagai working direcotry (WORKDIR /app)

Membuka port 5000 saat image dijalankan dan docker akan mengizinkan koneksi menggunakan port tersebut (EXPOSE 5000)

Menyalin seluruh konten dari direktori saat ini ke direktori /app yang ada di dalam container. (COPY . /app)

Menjalankan perintah untuk menginstall dependencies yang telah didefinisikan sebelumnya di dalam file requirement.txt (RUN python -m pip install -r requirements.txt)

Memberikan perintah untuk menjalankan script api.py (CMD ["python", "api.py"])

~~~
version: '3.10'

services:
  flask-app:
    container_name: backend-compose
    image: backend:latest
    build:
      context: ./backend
      dockerfile: Dockerfile
    restart: always
    ports:
      - "5010:5000"
    depends_on:
      - postgres

  postgres:
    container_name: postgres-compose
    image: postgres:12
    restart: always
    ports:
      - "5433:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=qwerty123
      - POSTGRES_DB=pacmail
    volumes:
      - postgres-data:/var/lib/postgresql/data

  frontend:
    image: node:latest
    ports:
      - 8090:8080
    depends_on:
      - flask-app 

volumes:
  postgres-data:
~~~
1. Mendifinisikan format docker compose yang digunakan (version: '3.10')
2. Mendifinisikan layanan-layanan yang dijalankan oleh kontainer. (services:)
3. Mendifinisikan layanan yang dimiliki oleh backend. (flask-app:)
4. Container untuk backend diberi nama backend-compose (container_name: backend-compose)
4. Image yang digunakakn adalah backend:latest (image: backend:latest)
5. Mendifinisikan image yang akan dibangun ( build:)
6. Menetapkan agar kontainer akan selalu di restart jika berhenti (restart: always)
7. Mendifinisikan bahwa service tersebut bergantung dengan service lainnya dalam kasus ini flask-app bergantung pada postgres(depends_on:
      - postgres)
8. Mendifinisikan nama dari volume docker yang akan digunakan untuk menyimpan data PostgreSQL (volumes:
      - postgres-data:/var/lib/postgresql/data)


### CI/CD
Untuk dapat melakukan continuously pushing saat ada implementasi, maka dibuat CI/CD untuk memanstikan bahwa setiap perubahan dapat terus dilakukan.
~~~
name: CI (Continuous Integration)

on:
  pull_request:
    branches: [ "main" ]

jobs:

  build-testing:
    name: Build and Testing
    runs-on: windows-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
    
      - name: Install Docker Compose 
        run: |
          sudo apt-get update
          sudo apt-get install -y docker-compose
      
      - name: Build and Run Container
        run: |
          sudo docker compose up -d

      - name: Install Requirements for Testing
        run: |
          pip install -r testing\requirements.txt

      - name: Testing
        run: |
          sleep 20
          pytest testing/test_signup.py
~~~


~~~
name: CD

on: workflow_dispatch

jobs:
    
    build-push:
        name: Build Push Image to Docker
        runs-on: windows-latest

        steps:
            - name: Check Repository
              uses: actions/checkout@v2

            - name: Login Docker
              uses: docker/login-action@v2
              with:
                username: ${{ secrets.DOCKER_USERNAME }}
                password: ${{ secrets.DOCKER_PASS }}

            - name: Set up Docker
              uses: docker/setup-buildx-action@v2
              
            - name: Build and push image
              uses: docker/build-push-action@v4
              with:
                context: ./backend
                file: ./flask/dockerfile
                push: true
                tags: ${{ secrets.DOCKER_USERNAME }}/flask:${{ github.run_number }}/Flask:latest
                  
    deploy: 
        name: server deploy
        runs-on: self-hosted
        needs: build-push

        steps:
          - name : Pull latest images
            run: |
              docker pull ${{ secrets.DOCKER_USERNAME }}/Flask:latest
      
          - name: Stop and Remove Existing Containers and Networks
            run: |
              docker stop $(docker ps -a -q) && docker rm $(docker ps -a -q)
              docker network prune -f

          - name: Create Network and Run containers
            run : |               
              sudo docker compose up -d
              
          - name: Remove unused data
            run: |
               docker system prune -af
~~~

~~~
name: CD2

on: workflow_dispatch

jobs:
    deploy:
        name: Deploy to Server
        runs-on: self-hosted

        steps:
            - name : Pull newest images
              run: |
                docker pull ${{secrets.DOCKER_USERNAME}}//Flask:${{github.run_number}}

            - name : Stop and Remove Existing Containers and Networks
              run: |
                docker stop $(docker ps -a -q) && docker rm $(docker ps -a -q)
                docker network prune -f

            - name : Create Network and Run Containers
              run: |
                docker network create vite
                docker run -d -p 5000:5000 --network vite --hostname Flask --mount "type=volume,source=pgdata,destination=/var/lib" --name flask -e
               
            - name : Remove unused data
              run: |
                docker system prune -af
~~~

