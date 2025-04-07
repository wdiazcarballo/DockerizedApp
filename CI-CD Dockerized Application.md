# การพัฒนาแอปพลิเคชันใน Docker และ CI/CD บน AWS

## เป้าหมายของบทนี้

นักศึกษาจะสามารถ:

1. เข้าใจว่า Docker คืออะไร และทำไมเราถึงต้องใช้
2. สร้าง Docker Image ด้วยตนเองทั้งฝั่ง backend และ frontend
3. เข้าใจการใช้ Docker Compose เพื่อรันหลาย container พร้อมกัน
4. นำ Docker Image ไป deploy บน AWS โดยใช้ Elastic Beanstalk
5. ตั้งค่า GitHub Actions สำหรับ CI/CD เพื่อทดสอบและปรับใช้อัตโนมัติ

## ส่วนที่ 1: ทำความเข้าใจกับ Docker

ก่อนที่เราจะลงมือเขียน Dockerfile หรือ deploy ขึ้น AWS นักศึกษาต้องเข้าใจก่อนว่า **Docker** คืออะไร และทำหน้าที่อะไรในกระบวนการพัฒนา software สมัยใหม่

**Docker** คือระบบสำหรับการ "บรรจุ" ซอฟต์แวร์ของเราไว้ในสิ่งที่เรียกว่า **container** ซึ่งเปรียบเสมือน "กล่องอุปกรณ์พร้อมใช้งาน" ที่มีทุกอย่างพร้อม ไม่ว่าจะเป็น:

- ตัวโปรแกรม (source code)
- ไลบรารีหรือ dependencies ที่ต้องใช้
- คำสั่งที่จะใช้รันโปรแกรม

เมื่อเราสร้าง Docker image แล้ว เราสามารถนำ image นั้นไปรันที่เครื่องใดก็ได้ที่มี Docker โดยไม่ต้องติดตั้งอะไรเพิ่มเติมอีกเลย

### เปรียบเทียบให้เห็นภาพ:
- ถ้าเราเขียนโปรแกรมไว้ที่เครื่องตัวเอง แล้วส่งโค้ดให้เพื่อนเปิดที่เครื่องเขาเอง บ่อยครั้งมันจะ "รันไม่ได้" เพราะเครื่องเขาอาจไม่มี Node.js, MongoDB หรือบางไลบรารี
- แต่ถ้าเราใช้ Docker เปรียบเหมือนส่งกล่องพร้อมอุปกรณ์ทุกอย่างไปให้เพื่อน เมื่อเพื่อนเปิดกล่อง (run container) โปรแกรมก็พร้อมทำงานทันที เหมือนเปิดเครื่องแล้วใช้งานได้เลย

## ส่วนที่ 2: การสร้าง Docker Image สำหรับ Backend

### ขั้นตอนที่ 1: สร้าง `backend/Dockerfile`

```dockerfile
FROM node:20

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm install

COPY . .

CMD ["npm", "start"]
```

**คำอธิบายเชิงเทคนิค:**

- `FROM node:20`: บอก Docker ว่าเราจะเริ่มจาก image ที่ติดตั้ง Node.js รุ่น 20 แล้ว ซึ่งเป็น base image
- `WORKDIR /app`: กำหนด working directory ภายใน container ถ้ายังไม่มี Docker จะสร้างให้
- `COPY package.json package-lock.json ./`: copy เฉพาะไฟล์ที่จำเป็นต่อการติดตั้ง dependency
- `RUN npm install`: ติดตั้งไลบรารีทั้งหมดตาม package.json
- `COPY . .`: คัดลอกไฟล์ทั้งหมดจากโฟลเดอร์ backend เข้าสู่ container
- `CMD ["npm", "start"]`: เป็นคำสั่งที่ใช้เมื่อ container ถูก run โดยบอกให้เริ่มเซิร์ฟเวอร์ Node.js

### ขั้นตอนที่ 2: สร้าง `backend/.dockerignore`

```
node_modules
.env*
```

**เหตุผล**: เพื่อไม่ให้ไฟล์ขนาดใหญ่หรือไฟล์ที่มีข้อมูลลับ (เช่น .env) ติดไปกับ image ซึ่งจะทำให้ขนาดใหญ่และไม่ปลอดภัย

### ขั้นตอนที่ 3: สร้าง Docker Image สำหรับ Backend

เปิด Terminal และรันคำสั่งต่อไปนี้:

```bash
docker image build -t blog-backend backend/
```

## ส่วนที่ 3: การสร้าง Docker Image สำหรับ Frontend

### ขั้นตอนที่ 1: สร้าง `Dockerfile` ที่ root ของโปรเจกต์

```dockerfile
FROM node:20 AS build

ARG VITE_BACKEND_URL=http://localhost:3001/api/v1

WORKDIR /build

COPY package.json .
COPY package-lock.json .

RUN npm install

COPY . .

RUN npm run build

FROM nginx AS final

WORKDIR /usr/share/nginx/html

COPY --from=build /build/dist .
```

**คำอธิบาย:**

Dockerfile นี้ใช้สิ่งที่เรียกว่า **multi-stage build** คือการแยกขั้นตอน build ออกจากขั้นตอน run:

- `AS build`: สร้าง stage ชื่อ build สำหรับติดตั้งและ build React (Vite)
- `ARG VITE_BACKEND_URL`: รับค่าตัวแปรเพื่อใช้ตอน build frontend ให้ชี้ไปที่ backend
- `FROM nginx`: สร้าง stage ใหม่ที่ใช้ nginx สำหรับเสิร์ฟ static files เช่น HTML/CSS/JS
- `COPY --from=build /build/dist .`: คัดลอกไฟล์ที่ build เสร็จแล้วมาใส่ไว้ใน nginx

### ขั้นตอนที่ 2: สร้าง `.dockerignore` สำหรับ frontend

```
node_modules
.env*
backend
.vscode
.git
.husky
.commitlintrc.json
```

### ขั้นตอนที่ 3: สร้าง Docker Image สำหรับ Frontend

```bash
docker build -t blog-frontend .
```

## ส่วนที่ 4: การใช้งาน Docker Compose

**Docker Compose** ช่วยให้เราสามารถรันหลาย container พร้อมกันได้ โดยไม่ต้องเปิดหลายหน้าต่าง terminal

### ขั้นตอนที่ 1: สร้างไฟล์ `compose.yaml`:

```yaml
version: '3.9'

services:
  blog-database:
    image: mongo
    ports:
      - '27017:27017'

  blog-backend:
    build: backend/
    environment:
      - PORT=3001
      - DATABASE_URL=mongodb://blog-database:27017/blog
    ports:
      - '3001:3001'
    depends_on:
      - blog-database

  blog-frontend:
    build:
      context: .
      args:
        VITE_BACKEND_URL: http://localhost:3001/api/v1
    ports:
      - '3000:80'
    depends_on:
      - blog-backend
```

### ขั้นตอนที่ 2: รันทุกบริการพร้อมกัน

```bash
docker compose up
```

จะทำให้ MongoDB, backend, frontend รันพร้อมกัน นักศึกษาสามารถเข้า `http://localhost:3000` เพื่อดู frontend ที่เชื่อมกับ backend และฐานข้อมูลได้

## ส่วนที่ 5: การปรับใช้แอปบน AWS

### ขั้นตอนที่ 1: สร้างฐานข้อมูล MongoDB บน Atlas

1. ไปที่ [MongoDB Atlas](https://www.mongodb.com/atlas) และสร้างบัญชีหรือเข้าสู่ระบบ
2. สร้าง cluster ใหม่ (ฟรี tier M0 เพียงพอสำหรับการทดลอง)
3. ตั้งค่าผู้ใช้ฐานข้อมูลและรหัสผ่าน
4. อนุญาต IP ทั้งหมดในรายการอนุญาต Network Access (0.0.0.0/0)
5. หลังจากสร้าง cluster ให้คัดลอก connection string สำหรับใช้ในขั้นตอนต่อไป

### ขั้นตอนที่ 2: สร้างบัญชี AWS และตั้งค่า

1. สร้างบัญชี AWS หรือเข้าสู่ระบบที่ [AWS Console](https://aws.amazon.com/)
2. ติดตั้ง AWS CLI บนเครื่องของคุณและตั้งค่า credentials:

```bash
aws configure
```

3. กรอก Access Key ID, Secret Access Key และภูมิภาคที่ต้องการใช้ (เช่น ap-southeast-1 สำหรับสิงคโปร์)

### ขั้นตอนที่ 3: ปรับใช้ Docker Images บน ECR

1. สร้าง repository บน ECR สำหรับแบ็กเอนด์และฟรอนต์เอนด์:

```bash
aws ecr create-repository --repository-name blog-backend
aws ecr create-repository --repository-name blog-frontend
```

2. เข้าสู่ระบบ ECR:

```bash
aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin YOUR_AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com
```

3. สร้างและอัปโหลด images:

```bash
# สำหรับแบ็กเอนด์
docker build -t YOUR_AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/blog-backend:latest backend/
docker push YOUR_AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/blog-backend:latest

# สำหรับฟรอนต์เอนด์
docker build -t YOUR_AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/blog-frontend:latest --build-arg VITE_BACKEND_URL=http://YOUR_BACKEND_URL/api/v1 .
docker push YOUR_AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/blog-frontend:latest
```

### ขั้นตอนที่ 4: ปรับใช้บน AWS Elastic Beanstalk

#### สำหรับแบ็กเอนด์

1. ติดตั้ง Elastic Beanstalk CLI:

```bash
pip install awsebcli
```

2. สร้างไฟล์ `backend/Dockerrun.aws.json`:

```json
{
  "AWSEBDockerrunVersion": "1",
  "Image": {
    "Name": "YOUR_AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/blog-backend:latest",
    "Update": "true"
  },
  "Ports": [
    {
      "ContainerPort": 3001,
      "HostPort": 80
    }
  ],
  "Volumes": [],
  "Logging": "/var/log/app"
}
```

3. สร้างแอปพลิเคชัน Elastic Beanstalk:

```bash
cd backend
eb init blog-backend --platform docker --region ap-southeast-1
```

4. สร้างสภาพแวดล้อมและปรับใช้:

```bash
eb create blog-backend-env --single --instance-type t2.micro
```

5. ตั้งค่าตัวแปรสภาพแวดล้อม:

```bash
eb setenv DATABASE_URL=mongodb+srv://YOUR_MONGODB_CONNECTION_STRING PORT=3001
```

#### สำหรับฟรอนต์เอนด์

1. สร้างไฟล์ `Dockerrun.aws.json` ที่โฟลเดอร์หลัก:

```json
{
  "AWSEBDockerrunVersion": "1",
  "Image": {
    "Name": "YOUR_AWS_ACCOUNT_ID.dkr.ecr.ap-southeast-1.amazonaws.com/blog-frontend:latest",
    "Update": "true"
  },
  "Ports": [
    {
      "ContainerPort": 80,
      "HostPort": 80
    }
  ],
  "Volumes": [],
  "Logging": "/var/log/nginx"
}
```

2. สร้างแอปพลิเคชัน Elastic Beanstalk:

```bash
eb init blog-frontend --platform docker --region ap-southeast-1
```

3. สร้างสภาพแวดล้อมและปรับใช้:

```bash
eb create blog-frontend-env --single --instance-type t2.micro
```

## ส่วนที่ 6: การตั้งค่า CI เพื่ออัตโนมัติการทดสอบ

เราจะใช้ GitHub Actions เพื่อตั้งค่าการทดสอบอัตโนมัติเมื่อมีการ push หรือสร้าง pull request.

### ขั้นตอนที่ 1: สร้างไฟล์ CI สำหรับฟรอนต์เอนด์

1. สร้างโฟลเดอร์ `.github/workflows`
2. สร้างไฟล์ `.github/workflows/frontend-ci.yaml`:

```yaml
name: Blog Frontend CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  lint-and-build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    steps:
      - uses: actions/checkout@v3
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - name: Install dependencies
        run: npm install
      - name: Run linter on frontend
        run: npm run lint
      - name: Build frontend
        run: npm run build
```

### ขั้นตอนที่ 2: สร้างไฟล์ CI สำหรับแบ็กเอนด์

สร้างไฟล์ `.github/workflows/backend-ci.yaml`:

```yaml
name: Blog Backend CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    defaults:
      run:
        working-directory: ./backend
    steps:
      - uses: actions/checkout@v3
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - name: Install dependencies
        run: npm install
      - name: Run linter on backend
        run: npm run lint
      - name: Run backend tests
        run: npm test
```

## ส่วนที่ 7: การตั้งค่า CD เพื่ออัตโนมัติการปรับใช้

### ขั้นตอนที่ 1: เพิ่ม Secrets ใน GitHub Repository

1. ไปที่ Settings > Secrets and variables > Actions ในโปรเจค GitHub ของคุณ
2. เพิ่ม secret ต่อไปนี้:
   - `AWS_ACCESS_KEY_ID`: Access Key ของคุณ
   - `AWS_SECRET_ACCESS_KEY`: Secret Access Key ของคุณ
   - `AWS_REGION`: ภูมิภาคของคุณ (เช่น ap-southeast-1)
   - `AWS_ACCOUNT_ID`: หมายเลขบัญชี AWS ของคุณ
   - `MONGODB_CONNECTION_STRING`: Connection string ของ MongoDB

### ขั้นตอนที่ 2: สร้างไฟล์ CD Workflow

สร้างไฟล์ `.github/workflows/cd.yaml`:

```yaml
name: Deploy Blog Application

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: ${{ steps.deploy-frontend.outputs.url }}
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Build and push backend image
        uses: docker/build-push-action@v4
        with:
          context: ./backend
          file: ./backend/Dockerfile
          push: true
          tags: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/blog-backend:latest
      
      - name: Deploy backend to Elastic Beanstalk
        id: deploy-backend
        run: |
          cd backend
          pip install awsebcli
          eb init blog-backend --platform docker --region ${{ secrets.AWS_REGION }}
          eb deploy blog-backend-env
          echo "url=http://blog-backend-env.eba-xxx.${{ secrets.AWS_REGION }}.elasticbeanstalk.com" >> $GITHUB_OUTPUT
      
      - name: Build and push frontend image
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/blog-frontend:latest
          build-args: VITE_BACKEND_URL=${{ steps.deploy-backend.outputs.url }}/api/v1
      
      - name: Deploy frontend to Elastic Beanstalk
        id: deploy-frontend
        run: |
          pip install awsebcli
          eb init blog-frontend --platform docker --region ${{ secrets.AWS_REGION }}
          eb deploy blog-frontend-env
          echo "url=http://blog-frontend-env.eba-xxx.${{ secrets.AWS_REGION }}.elasticbeanstalk.com" >> $GITHUB_OUTPUT
```

## สรุป

ในบทเรียนนี้เราได้เรียนรู้วิธีการ:

1. ทำความเข้าใจแนวคิดของ Docker และ containerization
2. สร้าง Docker images สำหรับแอปพลิเคชัน full-stack ของเรา
3. ใช้ Docker Compose เพื่อรันหลายบริการพร้อมกัน
4. ปรับใช้แอปพลิเคชันบน AWS แทน Google Cloud Run
5. ตั้งค่า CI เพื่อทดสอบโค้ดโดยอัตโนมัติ
6. ตั้งค่า CD เพื่อปรับใช้โดยอัตโนมัติเมื่อมีการ push ไปยัง branch main

โดยใช้ AWS Elastic Beanstalk เราสามารถปรับใช้คอนเทนเนอร์ Docker ได้อย่างง่ายดาย และด้วยการตั้งค่า CI/CD นี้จะช่วยให้เราสามารถพัฒนาแอปพลิเคชันได้อย่างมีประสิทธิภาพมากขึ้น

ในบทต่อไป เราจะเรียนรู้เกี่ยวกับการเพิ่มระบบการยืนยันตัวตนด้วย JWT ซึ่งจะช่วยให้เราสามารถควบคุมว่าใครสามารถสร้างบทความในบล็อกของเราได้
