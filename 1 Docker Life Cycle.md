## 🐳 Workshop: Docker Lifecycle, Networking & Persistence

### สำหรับนักศึกษารายวิชา DTI 201 – Full Stack Software Development Skills

**ระยะเวลา:** 30 นาที   
**ลักษณะกิจกรรม:** Hands-on + อภิปรายมุมมองแบบนักพัฒนา
**เป้าหมาย:**

- เข้าใจวงจรชีวิตของ Docker container (Lifecycle)
    
- สร้าง container-based web app แบบ full-stack
    
- เชื่อมโยง container networking และ volume persistence เข้ากับ workflow จริง
    
- เชื่อมโยงกับแนวคิด DevOps/CI-CD ที่จะเรียนต่อ
    

---

## Part 1: เข้าใจ Docker Lifecycle แบบ Full-Stack Developer

### 🔁 วงจรชีวิตของ Docker Container

|สถานะ|คำอธิบาย|คำสั่งที่เกี่ยวข้อง|
|---|---|---|
|**Created**|สร้าง container จาก image ยังไม่รัน|`docker create`|
|**Running**|Container ทำงานอยู่|`docker start`, `docker run`|
|**Paused**|Container หยุดชั่วคราว (ใช้ CPU 0%)|`docker pause`, `docker unpause`|
|**Stopped / Exited**|Container จบการทำงานแล้ว|`docker stop`, `docker kill`|
|**Deleted**|Container ถูกลบ|`docker rm`|

🎯 **กิจกรรมที่ 1:** Lifecycle Simulation  
ให้ใช้คำสั่ง `docker run`, `docker stop`, `docker start`, `docker pause`, `docker rm` เพื่อดูผลกระทบจริงในระบบ (ใช้ nginx image)

```bash
docker run -d --name demo-nginx -p 8080:80 nginx
```

```bash
docker pause demo-nginx
```

```bash
docker unpause demo-nginx
```

```bash
docker stop demo-nginx
```

```bash
docker start demo-nginx
```

```bash
docker rm demo-nginx
```


📝 ให้บันทึกผลลัพธ์จาก `docker ps -a` และวาด diagram วงจร lifecycle พร้อมคำอธิบายของแต่ละสถานะ

---

## Part 2: Custom Image – สร้าง Containerized Web Application

### 👷‍♀️ กิจกรรมที่ 2: เขียน Dockerfile ด้วยตัวเอง

```Dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]
```

📌 นักศึกษาควร:

- อธิบายว่าแต่ละคำใน Dockerfile มีผลกับ container lifecycle อย่างไร
    
- วิเคราะห์ว่า container ที่ใช้ `CMD` ต่างจากที่ใช้ `ENTRYPOINT` อย่างไรใน context ของ DevOps
    

---

## Part 3: Container Networking – ให้ Container คุยกันเอง

### 🔗 แนวคิดสำคัญ

- ทุก container มี network namespace เป็นของตัวเอง
    
- Docker bridge network ใช้เชื่อม container เหมือนอยู่ใน LAN เดียวกัน
    

### 🌐 กิจกรรมที่ 3: สร้าง Node.js API + React Frontend + Docker Network

1. สร้าง container backend (Node.js API) ที่รับค่าจาก REST และ return JSON
    
2. สร้าง container frontend (React app) ที่ fetch ค่าจาก backend
    
3. เชื่อมผ่าน `docker network`
    

```bash
docker network create fullstack-net
```

```bash
docker run -d --name backend --network fullstack-net my-api
```

```bash
docker run -d --name frontend --network fullstack-net -p 3000:3000 my-frontend
```

🧠 คำถามฝึกคิด: ถ้าเราไม่ใช้ `docker network` แล้วใช้ `localhost` แทนจะเกิดอะไรขึ้น?
<details>
  <summary>💡 Click to reveal the solution</summary>

  ### ✅ Solution

  If you use `localhost` in a container, it refers to itself, not to another container. You must use `docker network` and refer to the backend container by its name.

</details>
---

## Part 4: Volume & Bind Mounts – ทำให้ข้อมูลไม่หายเมื่อ Container ตาย

### 💾 Volume vs Bind Mount

|ประเภท|Mount ที่ไหน|Use case|
|---|---|---|
|**Volume**|Docker จัดการ path ให้|ใช้กับ container ที่ต้องเก็บ log หรือไฟล์ถาวร|
|**Bind Mount**|เรากำหนด path เอง|ใช้กับ dev-mode ที่แก้ไฟล์จาก host แล้วให้ container เห็นทันที|

### 💡 กิจกรรมที่ 4: ทดลอง Persistence

1. ใช้ volume กับ backend API เขียน log ลง `/app/logs`
    
2. หยุดและลบ container แล้วเช็คว่า log ยังอยู่หรือไม่
    
3. ทดลอง bind mount frontend folder แล้วแก้ไข React code จาก VS Code โดยไม่ rebuild
    

```bash
docker volume create mydata
```

```bash
docker run -d --name persistent-backend -v mydata:/app/logs my-api
```

---

## Part 5: Mini Challenge – Docker Compose

🛠 นักศึกษาแต่ละกลุ่มต้องเขียน `docker-compose.yml` สำหรับระบบ full-stack ที่มี

- Node.js backend
    
- React frontend
    
- PostgreSQL (ใช้ official image)
    
- volume สำหรับเก็บข้อมูลฐานข้อมูล
    

```yaml
version: "3"
services:
  db:
    image: postgres
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: example
  backend:
    build: ./backend
    depends_on:
      - db
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
volumes:
  pgdata:
```

✅ **เป้าหมาย**

- เข้าใจ multi-container orchestration
    
- ทำความเข้าใจ dependency และ lifecycle ของ service ต่าง ๆ
    
- เชื่อมโยงกับ CI/CD ที่จะเรียนในสัปดาห์ถัดไป
    

---

## 🔚 สรุปสิ่งที่ต้องเข้าใจ

### นักศึกษาต้อง:

1. อธิบาย lifecycle ของ container พร้อมตัวอย่างคำสั่ง
    
2. เปรียบเทียบ network แบบต่าง ๆ และอธิบายว่าทำไมเราจึงใช้ `bridge` network
    
3. แสดงตัวอย่างการใช้ volume/bind mount และผลกระทบเมื่อ container ถูกลบ
    
4. ส่ง `Dockerfile`, `docker-compose.yml` และ `README.md` อธิบาย architecture ของระบบ
    

---

