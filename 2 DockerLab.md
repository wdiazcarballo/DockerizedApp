
# 🧪  การสร้าง Web Application แบบ Dockerized (DTI 201)

## 🎯 เป้าหมายของแลป

ให้นักศึกษาสามารถ:
- สร้าง Dockerfile สำหรับ Frontend และ Backend
- ใช้งาน Docker Compose เพื่อรันแอปพลิเคชันแบบหลาย Container
- ตั้งค่า Environment Variables และระบบ CI/CD
- ปรับปรุงประสิทธิภาพและความปลอดภัยของระบบ

---

## ✅ ขั้นตอนการปฏิบัติ

### Part 1: ทำความเข้าใจพื้นฐาน Docker

1. นักศึกษาศึกษาความหมายของคำต่อไปนี้:
   - Dockerfile
   - Image
   - Container
   - Docker Compose

2. ทดลองเปิดใช้งาน Docker Desktop แล้วตรวจสอบด้วยคำสั่ง:
   ```bash
   docker --version
   ```

---

### Part 2: Dockerizing Backend (Node.js/Express)

1. นักศึกษาสร้างไฟล์ `Dockerfile` ภายในโฟลเดอร์ backend ด้วยเนื้อหาดังนี้:

   ```Dockerfile
   FROM node:18-alpine
   WORKDIR /app
   COPY package*.json ./
   RUN npm install
   COPY . .
   EXPOSE 3001
   CMD ["npm", "start"]
   ```

2. สร้างไฟล์ `.dockerignore` เพื่อไม่ให้ Docker คัดลอกไฟล์ที่ไม่จำเป็น:
   ```
   node_modules
   .git
   .env
   ```

---

### Part 3: Dockerizing Frontend (React)

1. สร้าง `Dockerfile` สำหรับ React ในโฟลเดอร์ frontend:

   ```Dockerfile
   FROM node:18-alpine as build
   WORKDIR /app
   COPY package*.json ./
   RUN npm install
   COPY . .
   RUN npm run build

   FROM nginx:alpine
   COPY --from=build /app/dist /usr/share/nginx/html
   EXPOSE 80
   CMD ["nginx", "-g", "daemon off;"]
   ```

2. สร้าง `.dockerignore` อีกหนึ่งไฟล์ในโฟลเดอร์ frontend:
   ```
   node_modules
   .git
   dist
   ```

---

### Part 4: ใช้งาน Docker Compose

1. นักศึกษาสร้างไฟล์ `docker-compose.yml` ไว้ที่ root ของโปรเจกต์
2. กำหนดบริการ (services) ทั้ง 3 ได้แก่ backend, frontend และ db ตามตัวอย่างในคู่มือ

---

### Part 5: เริ่มรันระบบ

1. ที่ root folder ให้รันคำสั่ง:
   ```bash
   docker-compose up --build
   ```

2. ตรวจสอบการทำงานของระบบ:
   - Frontend ที่ http://localhost
   - Backend API ที่ http://localhost:3001
   - PostgreSQL ที่พอร์ต 5432

---

### Part 6: จัดการ Environment Variables

1. นักศึกษาสร้างไฟล์ `.env.production` ภายใน backend:
   ```
   NODE_ENV=production
   PORT=3001
   DATABASE_URL=postgres://postgres:password@db:5432/fullstack_db
   ```

2. อัปเดต `docker-compose.yml` ให้ใช้งานไฟล์ `.env.production`

---

### Part 7: สร้างระบบ CI/CD ด้วย GitHub Actions

1. นักศึกษาสร้างโฟลเดอร์ `.github/workflows/`
2. ภายใน สร้างไฟล์ `docker-ci.yml` และใส่เนื้อหาตามตัวอย่างในคู่มือ

---

### Part 8: วิเคราะห์และปรับปรุงระบบ

1. นักศึกษาสำรวจแนวทางเพิ่มประสิทธิภาพของ Docker Image เช่น:
   - การใช้ multi-stage builds
   - ลดขนาด layer
   - ไม่ใช้ root user

2. กำหนดทรัพยากรจำกัดใน `docker-compose.yml` เพื่อจัดการหน่วยความจำและ CPU

