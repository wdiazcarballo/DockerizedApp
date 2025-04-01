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
