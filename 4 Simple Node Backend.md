## ✅ Step 1: Prerequisites
You need:
- Node.js installed (https://nodejs.org/)
- A code editor (like VS Code)
- A terminal or command prompt

Check Node.js and npm:
```bash
node -v
npm -v
```

---

## ✅ Step 2: Set up your project
In your terminal:
```bash
mkdir my-node-app
cd my-node-app
npm init -y
```
This creates a `package.json` file with default settings.

---

## ✅ Step 3: Install Express
```bash
npm install express
```

---

## ✅ Step 4: Create the app
Create a file called `index.js` (or `app.js`) and add:

```js
// index.js
const express = require('express');
const app = express();
const port = 3000;

// Define a simple route
app.get('/', (req, res) => {
  res.send('Hello, Node.js world!');
});

// Start the server
app.listen(port, () => {
  console.log(`App listening at http://localhost:${port}`);
});
```

---

## ✅ Step 5: Run the app
```bash
node index.js
```
Open your browser and go to: [http://localhost:3000](http://localhost:3000)

You should see: **"Hello, Node.js world!"**

---

## 🧠 Bonus: Add a second route
You can add more pages like this:
```js
app.get('/about', (req, res) => {
  res.send('This is the About page.');
});
```

---

Next Step:
- Add HTML rendering
- Use a frontend (like React)
- Store data (with a database)
- Add in Docker

# Let's Dockerize your simple Node.js + Express app step-by-step 🚀

---

## 🧱 Your Project Structure (Before Docker)
Assuming your project looks like this:

```
my-node-app/
├── index.js
├── package.json
├── package-lock.json
```

---

## 📦 Step 1: Create a `Dockerfile`
Inside your project folder, create a file named `Dockerfile` **(no extension)**:

```Dockerfile
# Use an official Node.js image as the base
FROM node:22

# Set the working directory inside the container
WORKDIR /app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm install

# Copy the rest of the app files
COPY . .

# Expose port 3000 to the outside
EXPOSE 3000

# Run the app
CMD [ "node", "index.js" ]
```

---

## 📄 Step 2: Create `.dockerignore`
Prevent copying unnecessary files like `node_modules`:

```
node_modules
npm-debug.log
Dockerfile
.dockerignore
```

---

## 🛠️ Step 3: Build the Docker image
In your terminal, run:

```bash
docker build -t my-node-app .
```

---

## 🚀 Step 4: Run the container
Run the container and map port 3000:

```bash
docker run -p 3000:3000 my-node-app
```

Then open your browser:  
👉 [http://localhost:3000](http://localhost:3000)  
You’ll see: `Hello, Node.js world!`

---

## 🔁 Optional: Run with automatic restart
For development:
```bash
docker run -p 3000:3000 -v $(pwd):/app my-node-app
```

---

## 🧹 To Stop & Clean Up
- Stop container: Press `Ctrl+C`
- List containers: `docker ps`
- Stop a container: `docker stop <container_id>`
- Remove image: `docker rmi my-node-app`

---


