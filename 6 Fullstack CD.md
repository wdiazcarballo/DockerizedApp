# Full-Stack Application Deployment with Docker and CI/CD on AWS

This comprehensive guide walks you through creating, containerizing, and deploying a full-stack blog application using Docker and AWS, complete with automated CI/CD pipelines.

## Table of Contents

- [Project Overview](#project-overview)
- [Prerequisites](#prerequisites)
- [Part 1: Understanding Docker](#part-1-understanding-docker)
- [Part 2: Project Structure Setup](#part-2-project-structure-setup)
- [Part 3: Backend Development](#part-3-backend-development)
- [Part 4: Frontend Development](#part-4-frontend-development)
- [Part 5: Containerizing the Application](#part-5-containerizing-the-application)
- [Part 6: Local Testing with Docker Compose](#part-6-local-testing-with-docker-compose)
- [Part 7: Setting Up AWS Infrastructure](#part-7-setting-up-aws-infrastructure)
- [Part 8: Deploying to AWS](#part-8-deploying-to-aws)
- [Part 9: Implementing CI/CD with GitHub Actions](#part-9-implementing-cicd-with-github-actions)
- [Troubleshooting](#troubleshooting)
- [Next Steps](#next-steps)

## Project Overview

We'll build a blog application with:

- **Backend**: Node.js/Express API with MongoDB
- **Frontend**: React (Vite) application
- **Containerization**: Docker for consistent environments
- **Cloud Deployment**: AWS Elastic Beanstalk
- **Automation**: CI/CD with GitHub Actions

## Prerequisites

- [Node.js](https://nodejs.org/) (v16+)
- [Docker](https://www.docker.com/get-started) and [Docker Compose](https://docs.docker.com/compose/install/)
- [AWS Account](https://aws.amazon.com/)
- [GitHub Account](https://github.com/)
- [MongoDB Atlas Account](https://www.mongodb.com/cloud/atlas)
- Basic familiarity with JavaScript, React, and Node.js

## Part 1: Understanding Docker

### What is Docker?

Docker is a platform that packages applications and their dependencies into isolated containers that can run anywhere. Think of containers as lightweight, standalone, executable packages that include everything needed to run an application.

### Key Benefits:

- **Consistency**: "Works on my machine" problems are eliminated
- **Isolation**: Applications run in their own environment
- **Portability**: Run anywhere Docker is installed
- **Efficiency**: Lightweight compared to virtual machines
- **Scalability**: Easy to replicate and scale in production

### Core Docker Concepts:

- **Images**: Blueprints/templates for containers
- **Containers**: Running instances of images
- **Dockerfile**: Instructions to build an image
- **Docker Compose**: Tool for defining multi-container applications

## Part 2: Project Structure Setup

Let's set up a directory structure for our application:

```bash
mkdir -p blog-app/{backend,frontend}
cd blog-app
```

Initialize Git:

```bash
git init
echo "node_modules\n.env\n.DS_Store" > .gitignore
```

## Part 3: Backend Development

### Step 1: Initialize Backend Project

```bash
cd backend
npm init -y
npm install express mongoose cors dotenv helmet morgan
npm install --save-dev nodemon eslint
```

### Step 2: Create Basic Express Server

Create `backend/server.js`:

```javascript
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
require('dotenv').config();

// Initialize express app
const app = express();
const PORT = process.env.PORT || 3001;

// Middleware
app.use(cors());
app.use(helmet());
app.use(morgan('dev'));
app.use(express.json());

// Connect to MongoDB
mongoose.connect(process.env.DATABASE_URL)
  .then(() => console.log('Connected to MongoDB'))
  .catch(err => console.error('MongoDB connection error:', err));

// Routes
app.get('/api/v1/health', (req, res) => {
  res.status(200).json({ status: 'ok', message: 'Server is running' });
});

// Blog post routes will go here
const blogRoutes = require('./routes/blogRoutes');
app.use('/api/v1/posts', blogRoutes);

// Error handling middleware
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});

// Start server
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

module.exports = app;  // For testing
```

### Step 3: Create Models

Create `backend/models/Post.js`:

```javascript
const mongoose = require('mongoose');

const postSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  content: {
    type: String,
    required: true
  },
  author: {
    type: String,
    required: true,
    default: 'Anonymous'
  }
}, { timestamps: true });

module.exports = mongoose.model('Post', postSchema);
```

### Step 4: Create Routes

Create `backend/routes/blogRoutes.js`:

```javascript
const express = require('express');
const router = express.Router();
const Post = require('../models/Post');

// GET all posts
router.get('/', async (req, res) => {
  try {
    const posts = await Post.find().sort({ createdAt: -1 });
    res.status(200).json(posts);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GET single post
router.get('/:id', async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    if (!post) return res.status(404).json({ message: 'Post not found' });
    res.status(200).json(post);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// CREATE post
router.post('/', async (req, res) => {
  try {
    const post = new Post(req.body);
    const savedPost = await post.save();
    res.status(201).json(savedPost);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// UPDATE post
router.put('/:id', async (req, res) => {
  try {
    const post = await Post.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );
    if (!post) return res.status(404).json({ message: 'Post not found' });
    res.status(200).json(post);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// DELETE post
router.delete('/:id', async (req, res) => {
  try {
    const post = await Post.findByIdAndDelete(req.params.id);
    if (!post) return res.status(404).json({ message: 'Post not found' });
    res.status(200).json({ message: 'Post deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Step 5: Configure Environment Variables

Create `backend/.env`:

```
PORT=3001
DATABASE_URL=mongodb://localhost:27017/blog
NODE_ENV=development
```

### Step 6: Update package.json Scripts

Update `backend/package.json`:

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "lint": "eslint .",
    "test": "echo \"Error: no test specified\" && exit 0"
  }
}
```

## Part 4: Frontend Development

### Step 1: Initialize Frontend with Vite

```bash
cd ../frontend
npm create vite@latest . -- --template react
npm install axios react-router-dom
```

### Step 2: Create Environment Configuration

Create `frontend/.env`:

```
VITE_BACKEND_URL=http://localhost:3001/api/v1
```

### Step 3: Create API Service

Create `frontend/src/services/api.js`:

```javascript
import axios from 'axios';

const baseURL = import.meta.env.VITE_BACKEND_URL;

const api = axios.create({
  baseURL
});

export const getPosts = async () => {
  const response = await api.get('/posts');
  return response.data;
};

export const getPost = async (id) => {
  const response = await api.get(`/posts/${id}`);
  return response.data;
};

export const createPost = async (post) => {
  const response = await api.post('/posts', post);
  return response.data;
};

export const updatePost = async (id, post) => {
  const response = await api.put(`/posts/${id}`, post);
  return response.data;
};

export const deletePost = async (id) => {
  const response = await api.delete(`/posts/${id}`);
  return response.data;
};

export default api;
```

### Step 4: Create Components

Create `frontend/src/components/PostList.jsx`:

```jsx
import { useState, useEffect } from 'react';
import { Link } from 'react-router-dom';
import { getPosts, deletePost } from '../services/api';

function PostList() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const fetchPosts = async () => {
    try {
      setLoading(true);
      const data = await getPosts();
      setPosts(data);
      setError(null);
    } catch (err) {
      setError('Failed to fetch posts');
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchPosts();
  }, []);

  const handleDelete = async (id) => {
    if (window.confirm('Are you sure you want to delete this post?')) {
      try {
        await deletePost(id);
        setPosts(posts.filter(post => post._id !== id));
      } catch (err) {
        setError('Failed to delete post');
        console.error(err);
      }
    }
  };

  if (loading) return <div>Loading posts...</div>;
  if (error) return <div className="error">{error}</div>;

  return (
    <div className="post-list">
      <h1>Blog Posts</h1>
      <Link to="/posts/new" className="button">Create New Post</Link>
      
      {posts.length === 0 ? (
        <p>No posts yet. Create your first post!</p>
      ) : (
        posts.map(post => (
          <div key={post._id} className="post-card">
            <h2>{post.title}</h2>
            <p>By {post.author} • {new Date(post.createdAt).toLocaleDateString()}</p>
            <div className="post-actions">
              <Link to={`/posts/${post._id}`}>View</Link>
              <Link to={`/posts/edit/${post._id}`}>Edit</Link>
              <button onClick={() => handleDelete(post._id)}>Delete</button>
            </div>
          </div>
        ))
      )}
    </div>
  );
}

export default PostList;
```

Create `frontend/src/components/PostForm.jsx`:

```jsx
import { useState, useEffect } from 'react';
import { useNavigate, useParams } from 'react-router-dom';
import { createPost, getPost, updatePost } from '../services/api';

function PostForm() {
  const navigate = useNavigate();
  const { id } = useParams();
  const isEditMode = Boolean(id);
  
  const [formData, setFormData] = useState({
    title: '',
    content: '',
    author: ''
  });
  
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (isEditMode) {
      const fetchPost = async () => {
        try {
          setLoading(true);
          const post = await getPost(id);
          setFormData(post);
          setError(null);
        } catch (err) {
          setError('Failed to fetch post');
          console.error(err);
        } finally {
          setLoading(false);
        }
      };
      
      fetchPost();
    }
  }, [id, isEditMode]);

  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      setLoading(true);
      if (isEditMode) {
        await updatePost(id, formData);
      } else {
        await createPost(formData);
      }
      navigate('/');
    } catch (err) {
      setError('Failed to save post');
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  if (loading && isEditMode) return <div>Loading post...</div>;
  if (error) return <div className="error">{error}</div>;

  return (
    <div className="post-form">
      <h1>{isEditMode ? 'Edit Post' : 'Create New Post'}</h1>
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label htmlFor="title">Title</label>
          <input
            type="text"
            id="title"
            name="title"
            value={formData.email}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label htmlFor="password">Password</label>
          <input
            type="password"
            id="password"
            name="password"
            value={formData.password}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-actions">
          <button type="submit" disabled={loading}>
            {loading ? 'Logging in...' : 'Login'}
          </button>
        </div>
        
        <p className="auth-redirect">
          Don't have an account? <Link to="/register">Register</Link>
        </p>
      </form>
    </div>
  );
}

export default Login;
```

### Step 10: Create Protected Route Component

Create `frontend/src/components/ProtectedRoute.jsx`:

```jsx
import { Navigate, Outlet } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

const ProtectedRoute = () => {
  const { isAuthenticated, loading } = useAuth();
  
  if (loading) {
    return <div>Loading...</div>;
  }
  
  return isAuthenticated ? <Outlet /> : <Navigate to="/login" />;
};

export default ProtectedRoute;
```

### Step 11: Update App.jsx with Auth Routes

Update `frontend/src/App.jsx` to include the authentication routes:

```jsx
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import PostList from './components/PostList';
import PostDetail from './components/PostDetail';
import PostForm from './components/PostForm';
import Login from './components/Login';
import Register from './components/Register';
import ProtectedRoute from './components/ProtectedRoute';
import Navbar from './components/Navbar';
import './App.css';

function App() {
  return (
    <AuthProvider>
      <Router>
        <div className="app-container">
          <Navbar />
          
          <main>
            <Routes>
              {/* Public Routes */}
              <Route path="/" element={<PostList />} />
              <Route path="/posts/:id" element={<PostDetail />} />
              <Route path="/login" element={<Login />} />
              <Route path="/register" element={<Register />} />
              
              {/* Protected Routes */}
              <Route element={<ProtectedRoute />}>
                <Route path="/posts/new" element={<PostForm />} />
                <Route path="/posts/edit/:id" element={<PostForm />} />
              </Route>
            </Routes>
          </main>
          
          <footer>
            <p>© 2025 Blog App</p>
          </footer>
        </div>
      </Router>
    </AuthProvider>
  );
}

export default App;
```

### Step 12: Create Navbar Component

Create `frontend/src/components/Navbar.jsx`:

```jsx
import { Link } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

function Navbar() {
  const { currentUser, isAuthenticated, logout } = useAuth();
  
  return (
    <header>
      <nav className="navbar">
        <div className="navbar-brand">
          <Link to="/">Blog App</Link>
        </div>
        
        <ul className="navbar-nav">
          <li className="nav-item">
            <Link to="/" className="nav-link">Home</Link>
          </li>
          
          {isAuthenticated ? (
            <>
              <li className="nav-item">
                <Link to="/posts/new" className="nav-link">New Post</Link>
              </li>
              <li className="nav-item user-menu">
                <span className="user-name">Welcome, {currentUser.username}</span>
                <button onClick={logout} className="logout-btn">Logout</button>
              </li>
            </>
          ) : (
            <>
              <li className="nav-item">
                <Link to="/login" className="nav-link">Login</Link>
              </li>
              <li className="nav-item">
                <Link to="/register" className="nav-link">Register</Link>
              </li>
            </>
          )}
        </ul>
      </nav>
    </header>
  );
}

export default Navbar;
```

### Step 13: Update API Service to Handle Auth

Update `frontend/src/services/api.js`:

```javascript
import axios from 'axios';

const baseURL = import.meta.env.VITE_BACKEND_URL;

const api = axios.create({
  baseURL
});

// Post API calls
export const getPosts = async () => {
  const response = await api.get('/posts');
  return response.data;
};

export const getPost = async (id) => {
  const response = await api.get(`/posts/${id}`);
  return response.data;
};

export const createPost = async (post) => {
  const response = await api.post('/posts', post);
  return response.data;
};

export const updatePost = async (id, post) => {
  const response = await api.put(`/posts/${id}`, post);
  return response.data;
};

export const deletePost = async (id) => {
  const response = await api.delete(`/posts/${id}`);
  return response.data;
};

// Intercept unauthorized responses
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response && error.response.status === 401) {
      // Clear token from localStorage
      localStorage.removeItem('token');
      // Redirect to login if unauthorized
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

### Step 14: Update CSS for Auth Components

Add the following to `frontend/src/App.css`:

```css
/* Auth Forms */
.auth-form {
  max-width: 500px;
  margin: 2rem auto;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 2rem;
}

.auth-form h1 {
  margin-bottom: 1.5rem;
  text-align: center;
  color: var(--primary-color);
}

.auth-redirect {
  margin-top: 1.5rem;
  text-align: center;
}

.auth-redirect a {
  color: var(--accent-color);
  text-decoration: none;
  font-weight: 500;
}

/* Navbar */
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 2rem;
  background-color: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.navbar-brand a {
  font-size: 1.5rem;
  font-weight: bold;
  color: var(--primary-color);
  text-decoration: none;
}

.navbar-nav {
  display: flex;
  list-style: none;
  gap: 1.5rem;
  align-items: center;
}

.nav-link {
  color: var(--text-color);
  text-decoration: none;
  font-weight: 500;
}

.nav-link:hover {
  color: var(--primary-color);
}

.user-menu {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.user-name {
  font-weight: 500;
}

.logout-btn {
  background: var(--accent-color);
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}
```

### Step 15: Update PostList and PostForm to Work with Auth

Modify `frontend/src/components/PostList.jsx` to show author name:

```jsx
import { useState, useEffect } from 'react';
import { Link } from 'react-router-dom';
import { getPosts, deletePost } from '../services/api';
import { useAuth } from '../context/AuthContext';

function PostList() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const { currentUser, isAuthenticated } = useAuth();

  const fetchPosts = async () => {
    try {
      setLoading(true);
      const data = await getPosts();
      setPosts(data);
      setError(null);
    } catch (err) {
      setError('Failed to fetch posts');
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchPosts();
  }, []);

  const handleDelete = async (id) => {
    if (window.confirm('Are you sure you want to delete this post?')) {
      try {
        await deletePost(id);
        setPosts(posts.filter(post => post._id !== id));
      } catch (err) {
        setError('Failed to delete post');
        console.error(err);
      }
    }
  };

  if (loading) return <div>Loading posts...</div>;
  if (error) return <div className="error">{error}</div>;

  return (
    <div className="post-list">
      <h1>Blog Posts</h1>
      
      {isAuthenticated && (
        <Link to="/posts/new" className="button">Create New Post</Link>
      )}
      
      {posts.length === 0 ? (
        <p>No posts yet. {isAuthenticated ? 'Create your first post!' : 'Login to create a post!'}</p>
      ) : (
        posts.map(post => (
          <div key={post._id} className="post-card">
            <h2>{post.title}</h2>
            <p>By {post.authorName} • {new Date(post.createdAt).toLocaleDateString()}</p>
            <div className="post-actions">
              <Link to={`/posts/${post._id}`}>View</Link>
              
              {/* Only show edit/delete for post author or admin */}
              {currentUser && (currentUser.id === post.author || currentUser.role === 'admin') && (
                <>
                  <Link to={`/posts/edit/${post._id}`}>Edit</Link>
                  <button onClick={() => handleDelete(post._id)}>Delete</button>
                </>
              )}
            </div>
          </div>
        ))
      )}
    </div>
  );
}

export default PostList;
```

Update `frontend/src/components/PostForm.jsx` to remove the author field (now automatically assigned):

```jsx
import { useState, useEffect } from 'react';
import { useNavigate, useParams } from 'react-router-dom';
import { createPost, getPost, updatePost } from '../services/api';

function PostForm() {
  const navigate = useNavigate();
  const { id } = useParams();
  const isEditMode = Boolean(id);
  
  const [formData, setFormData] = useState({
    title: '',
    content: ''
  });
  
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (isEditMode) {
      const fetchPost = async () => {
        try {
          setLoading(true);
          const post = await getPost(id);
          setFormData({
            title: post.title,
            content: post.content
          });
          setError(null);
        } catch (err) {
          setError('Failed to fetch post');
          console.error(err);
        } finally {
          setLoading(false);
        }
      };
      
      fetchPost();
    }
  }, [id, isEditMode]);

  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      setLoading(true);
      if (isEditMode) {
        await updatePost(id, formData);
      } else {
        await createPost(formData);
      }
      navigate('/');
    } catch (err) {
      setError('Failed to save post');
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  if (loading && isEditMode) return <div>Loading post...</div>;
  if (error) return <div className="error">{error}</div>;

  return (
    <div className="post-form">
      <h1>{isEditMode ? 'Edit Post' : 'Create New Post'}</h1>
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label htmlFor="title">Title</label>
          <input
            type="text"
            id="title"
            name="title"
            value={formData.title}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label htmlFor="content">Content</label>
          <textarea
            id="content"
            name="content"
            rows="10"
            value={formData.content}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-actions">
          <button type="button" onClick={() => navigate('/')}>Cancel</button>
          <button type="submit" disabled={loading}>
            {loading ? 'Saving...' : 'Save Post'}
          </button>
        </div>
      </form>
    </div>
  );
}

export default PostForm;
```

## Part 12: Testing and Deployment

### Step 1: Update Environment Variables

Update your backend environment variables to include the JWT secret:

```
PORT=3001
DATABASE_URL=mongodb://localhost:27017/blog
NODE_ENV=development
JWT_SECRET=your-secret-key-here
JWT_EXPIRE=30d
```

For production, make sure to set a secure JWT_SECRET.

### Step 2: Testing Locally with Docker Compose

Update your `docker-compose.yml` to include the new environment variables:

```yaml
version: '3.8'

services:
  mongodb:
    # ... existing config

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: blog-backend
    ports:
      - "3001:3001"
    environment:
      - PORT=3001
      - DATABASE_URL=mongodb://mongodb:27017/blog
      - NODE_ENV=development
      - JWT_SECRET=your-secret-key-here
      - JWT_EXPIRE=30d
    # ... rest of config

  # ... frontend config
```

Run your application:

```bash
docker-compose down
docker-compose up --build
```

### Step 3: Update Elastic Beanstalk Environment Variables

```bash
cd backend
eb setenv DATABASE_URL=mongodb+srv://username:password@cluster.mongodb.net/blog \
  NODE_ENV=production \
  JWT_SECRET=your-production-secret-key \
  JWT_EXPIRE=30d
```

### Step 4: Update CI/CD Workflows

Update your `.github/workflows/cd.yml` to include the new environment variables:

```yaml
# ... existing config

- name: Build and push backend image
  uses: docker/build-push-action@v4
  with:
    context: ./backend
    file: ./backend/Dockerfile
    push: true
    tags: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/blog-backend:latest
    build-args: |
      JWT_SECRET=${{ secrets.JWT_SECRET }}
      JWT_EXPIRE=${{ secrets.JWT_EXPIRE }}

# ... rest of config
```

Add these new secrets to your GitHub repository:
- `JWT_SECRET`: A secure random string
- `JWT_EXPIRE`: The JWT expiration period (e.g., "30d")

## Part 13: Next Steps

After completing this guide, here are some potential enhancements to consider:

1. **Add Testing**: Implement unit and integration tests using Jest and React Testing Library

2. **Implement Role-Based Access**: Enhance the permission system with more granular role-based access control

3. **Add Comment System**: Allow users to comment on blog posts

4. **Implement Image Uploads**: Add functionality to upload and store images for blog posts

5. **Add Search and Filtering**: Implement search functionality and post categorization

6. **Monitoring and Logging**: Set up monitoring and centralized logging for your application

7. **Performance Optimization**: Implement caching strategies and optimize database queries

8. **Security Enhancements**:
   - Rate limiting
   - CSRF protection
   - Content Security Policy
   - Regular security audits

9. **Add Pagination**: Implement pagination for the blog posts list

10. **Implement Social Authentication**: Add login options with Google, Facebook, etc.

## Conclusion

You've now built a complete full-stack blog application with:

- Modern frontend using React with Vite
- Robust backend API with Express and MongoDB
- Docker containerization for consistent environments
- CI/CD pipeline with GitHub Actions
- Secure user authentication with JWT
- Cloud deployment on AWS

This architecture provides a solid foundation that can scale with your needs and accommodate future enhancements. The containerized approach ensures your application will run consistently across different environments, from development to production.

Remember to follow security best practices, especially when handling user authentication and sensitive data. Regularly update your dependencies to address security vulnerabilities and keep your application running smoothly..title}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label htmlFor="author">Author</label>
          <input
            type="text"
            id="author"
            name="author"
            value={formData.author}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label htmlFor="content">Content</label>
          <textarea
            id="content"
            name="content"
            rows="10"
            value={formData.content}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-actions">
          <button type="button" onClick={() => navigate('/')}>Cancel</button>
          <button type="submit" disabled={loading}>
            {loading ? 'Saving...' : 'Save Post'}
          </button>
        </div>
      </form>
    </div>
  );
}

export default PostForm;
```

Create `frontend/src/components/PostDetail.jsx`:

```jsx
import { useState, useEffect } from 'react';
import { useParams, Link, useNavigate } from 'react-router-dom';
import { getPost, deletePost } from '../services/api';

function PostDetail() {
  const { id } = useParams();
  const navigate = useNavigate();
  
  const [post, setPost] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchPost = async () => {
      try {
        setLoading(true);
        const data = await getPost(id);
        setPost(data);
        setError(null);
      } catch (err) {
        setError('Failed to fetch post');
        console.error(err);
      } finally {
        setLoading(false);
      }
    };
    
    fetchPost();
  }, [id]);

  const handleDelete = async () => {
    if (window.confirm('Are you sure you want to delete this post?')) {
      try {
        await deletePost(id);
        navigate('/');
      } catch (err) {
        setError('Failed to delete post');
        console.error(err);
      }
    }
  };

  if (loading) return <div>Loading post...</div>;
  if (error) return <div className="error">{error}</div>;
  if (!post) return <div>Post not found</div>;

  return (
    <div className="post-detail">
      <div className="post-header">
        <h1>{post.title}</h1>
        <p className="post-meta">
          By {post.author} • {new Date(post.createdAt).toLocaleDateString()}
        </p>
      </div>
      
      <div className="post-content">
        <p>{post.content}</p>
      </div>
      
      <div className="post-actions">
        <Link to="/">Back to Posts</Link>
        <Link to={`/posts/edit/${post._id}`}>Edit</Link>
        <button onClick={handleDelete}>Delete</button>
      </div>
    </div>
  );
}

export default PostDetail;
```

### Step 5: Set Up Routing

Update `frontend/src/App.jsx`:

```jsx
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import PostList from './components/PostList';
import PostDetail from './components/PostDetail';
import PostForm from './components/PostForm';
import './App.css';

function App() {
  return (
    <Router>
      <div className="app-container">
        <header>
          <div className="logo">Blog App</div>
        </header>
        
        <main>
          <Routes>
            <Route path="/" element={<PostList />} />
            <Route path="/posts/:id" element={<PostDetail />} />
            <Route path="/posts/new" element={<PostForm />} />
            <Route path="/posts/edit/:id" element={<PostForm />} />
          </Routes>
        </main>
        
        <footer>
          <p>© 2025 Blog App</p>
        </footer>
      </div>
    </Router>
  );
}

export default App;
```

### Step 6: Add Basic Styling

Create `frontend/src/App.css`:

```css
:root {
  --primary-color: #4a6fa5;
  --secondary-color: #166088;
  --accent-color: #4fc3a1;
  --text-color: #333;
  --light-bg: #f8f9fa;
  --border-color: #ddd;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  line-height: 1.6;
  color: var(--text-color);
  background-color: var(--light-bg);
}

.app-container {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

header {
  background-color: white;
  padding: 1rem 2rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.logo {
  font-size: 1.5rem;
  font-weight: bold;
  color: var(--primary-color);
}

main {
  flex: 1;
  padding: 2rem;
  max-width: 1200px;
  margin: 0 auto;
  width: 100%;
}

footer {
  background-color: white;
  padding: 1rem 2rem;
  text-align: center;
  border-top: 1px solid var(--border-color);
}

/* Post List */
.post-list h1 {
  margin-bottom: 1.5rem;
}

.button {
  display: inline-block;
  background: var(--accent-color);
  color: white;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  text-decoration: none;
  margin-bottom: 2rem;
}

.post-card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 1.5rem;
  margin-bottom: 1.5rem;
}

.post-card h2 {
  margin-bottom: 0.5rem;
  color: var(--primary-color);
}

.post-actions {
  margin-top: 1rem;
  display: flex;
  gap: 1rem;
}

.post-actions a, .post-actions button {
  padding: 0.5rem 1rem;
  border-radius: 4px;
  text-decoration: none;
}

.post-actions a {
  background: var(--secondary-color);
  color: white;
}

.post-actions button {
  background: #f44336;
  color: white;
  border: none;
  cursor: pointer;
}

/* Post Form */
.post-form {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 2rem;
}

.post-form h1 {
  margin-bottom: 1.5rem;
}

.form-group {
  margin-bottom: 1.5rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 500;
}

.form-group input, .form-group textarea {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  font-family: inherit;
  font-size: 1rem;
}

.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 1rem;
}

.form-actions button {
  padding: 0.5rem 1.5rem;
  border-radius: 4px;
  border: none;
  font-size: 1rem;
  cursor: pointer;
}

.form-actions button[type="button"] {
  background: #f5f5f5;
  color: var(--text-color);
}

.form-actions button[type="submit"] {
  background: var(--accent-color);
  color: white;
}

/* Post Detail */
.post-detail {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  padding: 2rem;
}

.post-header {
  margin-bottom: 2rem;
}

.post-header h1 {
  margin-bottom: 0.5rem;
  color: var(--primary-color);
}

.post-meta {
  color: #666;
}

.post-content {
  margin-bottom: 2rem;
  line-height: 1.8;
}

.error {
  background: #ffebee;
  color: #c62828;
  padding: 1rem;
  border-radius: 4px;
  margin-bottom: 1rem;
}
```

## Part 5: Containerizing the Application

### Step 1: Create Backend Dockerfile

Create `backend/Dockerfile`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3001

CMD ["npm", "start"]
```

Create `backend/.dockerignore`:

```
node_modules
npm-debug.log
.env
.git
.gitignore
```

### Step 2: Create Frontend Dockerfile

Create `frontend/Dockerfile`:

```dockerfile
# Build stage
FROM node:20-alpine AS build

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

ARG VITE_BACKEND_URL=http://localhost:3001/api/v1
ENV VITE_BACKEND_URL=${VITE_BACKEND_URL}

RUN npm run build

# Production stage
FROM nginx:alpine AS production

COPY --from=build /app/dist /usr/share/nginx/html

COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Create `frontend/.dockerignore`:

```
node_modules
npm-debug.log
.env
.git
.gitignore
dist
```

Create `frontend/nginx.conf`:

```
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    # Disable caching for service worker
    location /service-worker.js {
        add_header Cache-Control "no-cache";
        proxy_cache_bypass $http_pragma;
        proxy_cache_revalidate on;
        expires off;
        access_log off;
    }
}
```

## Part 6: Local Testing with Docker Compose

Create `docker-compose.yml` in the root directory:

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:latest
    container_name: blog-mongodb
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    networks:
      - blog-network

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: blog-backend
    ports:
      - "3001:3001"
    environment:
      - PORT=3001
      - DATABASE_URL=mongodb://mongodb:27017/blog
      - NODE_ENV=development
    depends_on:
      - mongodb
    networks:
      - blog-network

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        - VITE_BACKEND_URL=http://localhost:3001/api/v1
    container_name: blog-frontend
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - blog-network

networks:
  blog-network:
    driver: bridge

volumes:
  mongodb_data:
```

Run the application locally:

```bash
docker-compose up --build
```

## Part 7: Setting Up AWS Infrastructure

### Step 1: Create MongoDB Atlas Cluster

1. Sign up or log in to [MongoDB Atlas](https://www.mongodb.com/cloud/atlas)
2. Create a new project
3. Create a cluster (M0 free tier is fine for testing)
4. Configure security:
   - Create a database user with read/write permissions
   - Set up network access (IP whitelist)
5. Get your connection string (click "Connect" -> "Connect your application")

### Step 2: Set Up AWS CLI

Install AWS CLI:

```bash
# For macOS using Homebrew
brew install awscli

# For Windows using Chocolatey
choco install awscli

# For Linux
pip install awscli
```

Configure AWS CLI:

```bash
aws configure
```

Enter your AWS Access Key ID, Secret Access Key, region (e.g., us-east-1, ap-southeast-1), and output format (json).

### Step 3: Install Elastic Beanstalk CLI

```bash
pip install awsebcli
```

## Part 8: Deploying to AWS

### Step 1: Push Docker Images to ECR

Create ECR repositories:

```bash
aws ecr create-repository --repository-name blog-backend
aws ecr create-repository --repository-name blog-frontend
```

Authenticate Docker to ECR:

```bash
aws ecr get-login-password --region YOUR_REGION | docker login --username AWS --password-stdin YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com
```

Build and push images:

```bash
# Backend
docker build -t YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/blog-backend:latest ./backend
docker push YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/blog-backend:latest

# Frontend (with production backend URL)
docker build -t YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/blog-frontend:latest \
  --build-arg VITE_BACKEND_URL=https://YOUR_BACKEND_URL/api/v1 \
  ./frontend
docker push YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/blog-frontend:latest
```

### Step 2: Deploy Backend to Elastic Beanstalk

Create `backend/Dockerrun.aws.json`:

```json
{
  "AWSEBDockerrunVersion": "1",
  "Image": {
    "Name": "YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/blog-backend:latest",
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

Initialize Elastic Beanstalk application:

```bash
cd backend
eb init blog-backend --platform docker --region YOUR_REGION
```

Create the environment:

```bash
eb create blog-backend-env --single --instance-type t2.micro
```

Set environment variables:

```bash
eb setenv DATABASE_URL=mongodb+srv://USERNAME:PASSWORD@YOUR_MONGODB_CLUSTER_URL/blog NODE_ENV=production
```

### Step 3: Deploy Frontend to Elastic Beanstalk

Create `frontend/Dockerrun.aws.json`:

```json
{
  "AWSEBDockerrunVersion": "1",
  "Image": {
    "Name": "YOUR_AWS_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/blog-frontend:latest",
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

Initialize Elastic Beanstalk application:

```bash
cd ../frontend
eb init blog-frontend --platform docker --region YOUR_REGION
```

Create the environment:

```bash
eb create blog-frontend-env --single --instance-type t2.micro
```

## Part 9: Implementing CI/CD with GitHub Actions

### Step 1: Set Up GitHub Repository Secrets

Go to your GitHub repository → Settings → Secrets and variables → Actions and add the following secrets:

- `AWS_ACCESS_KEY_ID`: Your AWS access key
- `AWS_SECRET_ACCESS_KEY`: Your AWS secret key
- `AWS_REGION`: Your AWS region
- `AWS_ACCOUNT_ID`: Your AWS account ID
- `MONGODB_CONNECTION_STRING`: Your MongoDB Atlas connection string

### Step 2: Create CI Workflow for Testing

Create a directory structure for GitHub Actions:

```bash
mkdir -p .github/workflows
```

Create a CI workflow file for testing both frontend and backend in `.github/workflows/ci.yml`:

```yaml
name: CI - Test Application

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test-backend:
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
        cache-dependency-path: './backend/package-lock.json'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Lint code
      run: npm run lint
    
    - name: Run tests
      run: npm test
  
  test-frontend:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    
    defaults:
      run:
        working-directory: ./frontend
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Use Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
        cache-dependency-path: './frontend/package-lock.json'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Build
      run: npm run build
```

### Step 3: Create CD Workflow for Deployment

Create a CD workflow file for automatic deployment in `.github/workflows/cd.yml`:

```yaml
name: CD - Deploy Application

on:
  push:
    branches: [ main ]
    paths-ignore:
      - '**.md'
      - '.github/workflows/ci.yml'

jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: [build-and-push-backend, build-and-push-frontend]
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}
    
    - name: Deploy backend to Elastic Beanstalk
      working-directory: ./backend
      run: |
        pip install awsebcli
        aws s3 cp s3://elasticbeanstalk-${{ secrets.AWS_REGION }}-${{ secrets.AWS_ACCOUNT_ID }}/saved_configs/blog-backend-config.cfg ./backend-config.cfg || true
        
        # Update Dockerrun.aws.json with latest image
        cat > Dockerrun.aws.json << EOL
        {
          "AWSEBDockerrunVersion": "1",
          "Image": {
            "Name": "${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/blog-backend:${GITHUB_SHA::7}",
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
        EOL
        
        eb init blog-backend --region ${{ secrets.AWS_REGION }} --platform docker
        eb deploy blog-backend-env --staged
    
    - name: Get backend URL
      id: get-backend-url
      run: |
        BACKEND_URL=$(aws elasticbeanstalk describe-environments --environment-names blog-backend-env --query "Environments[0].CNAME" --output text)
        echo "::set-output name=url::http://${BACKEND_URL}"
    
    - name: Deploy frontend to Elastic Beanstalk
      working-directory: ./frontend
      run: |
        pip install awsebcli
        aws s3 cp s3://elasticbeanstalk-${{ secrets.AWS_REGION }}-${{ secrets.AWS_ACCOUNT_ID }}/saved_configs/blog-frontend-config.cfg ./frontend-config.cfg || true
        
        # Update Dockerrun.aws.json with latest image and backend URL
        cat > Dockerrun.aws.json << EOL
        {
          "AWSEBDockerrunVersion": "1",
          "Image": {
            "Name": "${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/blog-frontend:${GITHUB_SHA::7}",
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
        EOL
        
        eb init blog-frontend --region ${{ secrets.AWS_REGION }} --platform docker
        eb deploy blog-frontend-env --staged
  
  build-and-push-backend:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Build, tag, and push backend image to Amazon ECR
      working-directory: ./backend
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: blog-backend
        IMAGE_TAG: ${{ github.sha }}
      run: |
        # Add MongoDB connection string to .env for build
        echo "DATABASE_URL=${{ secrets.MONGODB_CONNECTION_STRING }}" > .env
        echo "NODE_ENV=production" >> .env
        
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA::7} .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA::7}
  
  build-and-push-frontend:
    runs-on: ubuntu-latest
    needs: build-and-push-backend
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Get backend URL
      id: get-backend-url
      run: |
        BACKEND_URL=$(aws elasticbeanstalk describe-environments --environment-names blog-backend-env --query "Environments[0].CNAME" --output text)
        echo "::set-output name=url::http://${BACKEND_URL}"
    
    - name: Build, tag, and push frontend image to Amazon ECR
      working-directory: ./frontend
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: blog-frontend
        IMAGE_TAG: ${{ github.sha }}
        BACKEND_URL: ${{ steps.get-backend-url.outputs.url }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA::7} --build-arg VITE_BACKEND_URL=$BACKEND_URL/api/v1 .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA::7}
```

## Part 10: Troubleshooting Common Issues

### MongoDB Connection Issues

If you're experiencing MongoDB connection errors like the one you encountered:

```
MongoDB connection error: MongooseError: The `uri` parameter to `openUri()` must be a string, got "undefined".
```

Try these solutions:

1. **Check your .env file location**: Ensure the .env file is in the same directory as server.js

2. **Modify how dotenv is loaded**: Place this at the very top of your server.js file:
   ```javascript
   require('dotenv').config({ path: __dirname + '/.env' });
   ```

3. **Verify .env file content**: Make sure your .env file contains the correct DATABASE_URL variable:
   ```
   PORT=3001
   DATABASE_URL=mongodb://localhost:27017/blog
   NODE_ENV=development
   ```

4. **Check MongoDB is running**: If using a local MongoDB instance, ensure it's actually running

5. **Temporary hardcoded connection**: For testing, temporarily hardcode the connection string:
   ```javascript
   mongoose.connect('mongodb://localhost:27017/blog')
     .then(() => console.log('Connected to MongoDB'))
     .catch(err => console.error('MongoDB connection error:', err));
   ```

### Docker Networking Issues

If your services can't communicate with each other:

1. **Check container names**: Ensure container names match the hostnames used in your code

2. **Inspect network**: Use `docker network inspect blog-network` to verify all containers are on the same network

3. **Service discovery**: In Docker Compose, services should use the service name as the hostname:
   ```
   // When connecting from backend to MongoDB
   DATABASE_URL=mongodb://mongodb:27017/blog
   ```

### AWS Deployment Issues

Common issues with AWS Elastic Beanstalk:

1. **Permission errors**: Ensure your IAM user has sufficient permissions for ECR and Elastic Beanstalk

2. **Elastic Beanstalk logs**: Check logs in the AWS Console or using `eb logs` command

3. **Environment variables**: Verify environment variables are correctly set with `eb printenv`

4. **Health checks**: Check if health checks are passing; your application must respond to HTTP requests at the root path

## Part 11: Adding Authentication with JWT

Let's enhance the blog application by adding user authentication with JSON Web Tokens (JWT).

### Step 1: Update Backend Dependencies

```bash
cd backend
npm install jsonwebtoken bcryptjs
```

### Step 2: Create User Model

Create `backend/models/User.js`:

```javascript
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true,
    minlength: 3
  },
  email: {
    type: String,
    required: true,
    unique: true,
    trim: true,
    lowercase: true,
    match: [/^\S+@\S+\.\S+$/, 'Please enter a valid email address']
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  }
}, { timestamps: true });

// Hash password before saving
userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  
  try {
    const salt = await bcrypt.genSalt(10);
    this.password = await bcrypt.hash(this.password, salt);
    next();
  } catch (error) {
    next(error);
  }
});

// Method to compare passwords
userSchema.methods.comparePassword = async function(candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', userSchema);
```

### Step 3: Create Authentication Middleware

Create `backend/middleware/auth.js`:

```javascript
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Get token secret from environment variables
const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

exports.protect = async (req, res, next) => {
  let token;
  
  // Check if token exists in headers
  if (req.headers.authorization && req.headers.authorization.startsWith('Bearer')) {
    token = req.headers.authorization.split(' ')[1];
  }
  
  if (!token) {
    return res.status(401).json({ 
      success: false, 
      message: 'Not authorized to access this route' 
    });
  }
  
  try {
    // Verify token
    const decoded = jwt.verify(token, JWT_SECRET);
    
    // Find user by id
    const user = await User.findById(decoded.id).select('-password');
    
    if (!user) {
      return res.status(404).json({
        success: false,
        message: 'User not found'
      });
    }
    
    // Attach user to request object
    req.user = user;
    next();
  } catch (error) {
    return res.status(401).json({
      success: false,
      message: 'Not authorized to access this route'
    });
  }
};

// Role-based authorization
exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        message: `User role '${req.user.role}' is not authorized to access this route`
      });
    }
    next();
  };
};
```

### Step 4: Create Auth Controller

Create `backend/controllers/authController.js`:

```javascript
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Get token secret from environment variables
const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';
const JWT_EXPIRE = process.env.JWT_EXPIRE || '30d';

// Generate JWT token
const generateToken = (id) => {
  return jwt.sign({ id }, JWT_SECRET, {
    expiresIn: JWT_EXPIRE
  });
};

// @desc    Register user
// @route   POST /api/v1/auth/register
// @access  Public
exports.register = async (req, res) => {
  try {
    const { username, email, password } = req.body;
    
    // Check if user already exists
    const userExists = await User.findOne({ $or: [{ email }, { username }] });
    
    if (userExists) {
      return res.status(400).json({
        success: false,
        message: 'User already exists'
      });
    }
    
    // Create user
    const user = await User.create({
      username,
      email,
      password
    });
    
    // Generate token
    const token = generateToken(user._id);
    
    res.status(201).json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message
    });
  }
};

// @desc    Login user
// @route   POST /api/v1/auth/login
// @access  Public
exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;
    
    // Validate input
    if (!email || !password) {
      return res.status(400).json({
        success: false,
        message: 'Please provide email and password'
      });
    }
    
    // Check if user exists
    const user = await User.findOne({ email });
    
    if (!user) {
      return res.status(401).json({
        success: false,
        message: 'Invalid credentials'
      });
    }
    
    // Check if password matches
    const isMatch = await user.comparePassword(password);
    
    if (!isMatch) {
      return res.status(401).json({
        success: false,
        message: 'Invalid credentials'
      });
    }
    
    // Generate token
    const token = generateToken(user._id);
    
    res.status(200).json({
      success: true,
      token,
      user: {
        id: user._id,
        username: user.username,
        email: user.email,
        role: user.role
      }
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message
    });
  }
};

// @desc    Get current logged in user
// @route   GET /api/v1/auth/me
// @access  Private
exports.getMe = async (req, res) => {
  try {
    const user = await User.findById(req.user.id).select('-password');
    
    res.status(200).json({
      success: true,
      user
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message
    });
  }
};
```

### Step 5: Create Auth Routes

Create `backend/routes/authRoutes.js`:

```javascript
const express = require('express');
const router = express.Router();
const { register, login, getMe } = require('../controllers/authController');
const { protect } = require('../middleware/auth');

router.post('/register', register);
router.post('/login', login);
router.get('/me', protect, getMe);

module.exports = router;
```

### Step 6: Update Blog Post Model and Routes

Update `backend/models/Post.js` to include user reference:

```javascript
const mongoose = require('mongoose');

const postSchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
    trim: true
  },
  content: {
    type: String,
    required: true
  },
  author: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  authorName: {
    type: String,
    required: true
  }
}, { timestamps: true });

module.exports = mongoose.model('Post', postSchema);
```

Update `backend/routes/blogRoutes.js` to use authentication:

```javascript
const express = require('express');
const router = express.Router();
const Post = require('../models/Post');
const { protect, authorize } = require('../middleware/auth');

// GET all posts - public
router.get('/', async (req, res) => {
  try {
    const posts = await Post.find().sort({ createdAt: -1 });
    res.status(200).json(posts);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// GET single post - public
router.get('/:id', async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    if (!post) return res.status(404).json({ message: 'Post not found' });
    res.status(200).json(post);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// CREATE post - protected
router.post('/', protect, async (req, res) => {
  try {
    const post = new Post({
      ...req.body,
      author: req.user._id,
      authorName: req.user.username
    });
    const savedPost = await post.save();
    res.status(201).json(savedPost);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// UPDATE post - protected (only post owner or admin)
router.put('/:id', protect, async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    
    if (!post) return res.status(404).json({ message: 'Post not found' });
    
    // Check if user is post owner or admin
    if (post.author.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Not authorized to update this post' });
    }
    
    const updatedPost = await Post.findByIdAndUpdate(
      req.params.id,
      req.body,
      { new: true, runValidators: true }
    );
    
    res.status(200).json(updatedPost);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// DELETE post - protected (only post owner or admin)
router.delete('/:id', protect, async (req, res) => {
  try {
    const post = await Post.findById(req.params.id);
    
    if (!post) return res.status(404).json({ message: 'Post not found' });
    
    // Check if user is post owner or admin
    if (post.author.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Not authorized to delete this post' });
    }
    
    await Post.findByIdAndDelete(req.params.id);
    res.status(200).json({ message: 'Post deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Step 7: Update Server.js to Include Auth Routes

Update `backend/server.js` to include the new auth routes:

```javascript
// ... existing imports
const authRoutes = require('./routes/authRoutes');

// ... existing middleware

// Routes
app.get('/api/v1/health', (req, res) => {
  res.status(200).json({ status: 'ok', message: 'Server is running' });
});

// Auth routes
app.use('/api/v1/auth', authRoutes);

// Blog post routes
app.use('/api/v1/posts', blogRoutes);

// ... rest of the file
```

### Step 8: Add Authentication to Frontend

Install required packages:

```bash
cd ../frontend
npm install jwt-decode
```

Create `frontend/src/context/AuthContext.jsx`:

```jsx
import { createContext, useState, useEffect, useContext } from 'react';
import api from '../services/api';
import jwtDecode from 'jwt-decode';

const AuthContext = createContext();

export const useAuth = () => useContext(AuthContext);

export const AuthProvider = ({ children }) => {
  const [currentUser, setCurrentUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    const initializeAuth = async () => {
      if (token) {
        try {
          // Set token in axios headers
          api.defaults.headers.common['Authorization'] = `Bearer ${token}`;
          
          // Verify token is valid
          const { data } = await api.get('/auth/me');
          setCurrentUser(data.user);
        } catch (error) {
          console.error('Token validation failed:', error);
          logout();
        }
      }
      setLoading(false);
    };
    
    initializeAuth();
  }, [token]);
  
  const register = async (userData) => {
    const { data } = await api.post('/auth/register', userData);
    setAuthData(data);
    return data;
  };
  
  const login = async (credentials) => {
    const { data } = await api.post('/auth/login', credentials);
    setAuthData(data);
    return data;
  };
  
  const logout = () => {
    localStorage.removeItem('token');
    delete api.defaults.headers.common['Authorization'];
    setToken(null);
    setCurrentUser(null);
  };
  
  const setAuthData = (data) => {
    const { token, user } = data;
    localStorage.setItem('token', token);
    api.defaults.headers.common['Authorization'] = `Bearer ${token}`;
    setToken(token);
    setCurrentUser(user);
  };
  
  const isAuthenticated = !!token;
  
  const isTokenExpired = () => {
    if (!token) return true;
    
    try {
      const decoded = jwtDecode(token);
      return decoded.exp < Date.now() / 1000;
    } catch (error) {
      return true;
    }
  };
  
  const value = {
    currentUser,
    isAuthenticated,
    isTokenExpired,
    register,
    login,
    logout,
    loading
  };
  
  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
};
```

### Step 9: Create Authentication Components

Create `frontend/src/components/Register.jsx`:

```jsx
import { useState } from 'react';
import { useNavigate, Link } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

function Register() {
  const [formData, setFormData] = useState({
    username: '',
    email: '',
    password: '',
    confirmPassword: ''
  });
  
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(false);
  const navigate = useNavigate();
  const { register } = useAuth();
  
  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
  };
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    
    // Validate form
    if (formData.password !== formData.confirmPassword) {
      setError('Passwords do not match');
      return;
    }
    
    if (formData.password.length < 6) {
      setError('Password must be at least 6 characters long');
      return;
    }
    
    try {
      setLoading(true);
      setError(null);
      
      await register({
        username: formData.username,
        email: formData.email,
        password: formData.password
      });
      
      navigate('/');
    } catch (err) {
      setError(err.response?.data?.message || 'Registration failed');
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <div className="auth-form">
      <h1>Register</h1>
      
      {error && <div className="error">{error}</div>}
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label htmlFor="username">Username</label>
          <input
            type="text"
            id="username"
            name="username"
            value={formData.username}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label htmlFor="email">Email</label>
          <input
            type="email"
            id="email"
            name="email"
            value={formData.email}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label htmlFor="password">Password</label>
          <input
            type="password"
            id="password"
            name="password"
            value={formData.password}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-group">
          <label htmlFor="confirmPassword">Confirm Password</label>
          <input
            type="password"
            id="confirmPassword"
            name="confirmPassword"
            value={formData.confirmPassword}
            onChange={handleChange}
            required
          />
        </div>
        
        <div className="form-actions">
          <button type="submit" disabled={loading}>
            {loading ? 'Registering...' : 'Register'}
          </button>
        </div>
        
        <p className="auth-redirect">
          Already have an account? <Link to="/login">Login</Link>
        </p>
      </form>
    </div>
  );
}

export default Register;
```

Create `frontend/src/components/Login.jsx`:

```jsx
import { useState } from 'react';
import { useNavigate, Link } from 'react-router-dom';
import { useAuth } from '../context/AuthContext';

function Login() {
  const [formData, setFormData] = useState({
    email: '',
    password: ''
  });
  
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(false);
  const navigate = useNavigate();
  const { login } = useAuth();
  
  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
  };
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    
    try {
      setLoading(true);
      setError(null);
      
      await login(formData);
      navigate('/');
    } catch (err) {
      setError(err.response?.data?.message || 'Login failed');
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <div className="auth-form">
      <h1>Login</h1>
      
      {error && <div className="error">{error}</div>}
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label htmlFor="email">Email</label>
          <input
            type="email"
            id="email"
            name="email"
            value={formData
