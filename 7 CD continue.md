# Full-Stack Application Deployment with Docker and CI/CD on AWS

Let me complete the missing sections from the tutorial:

## Part 11: Adding Authentication with JWT (continued)

### Step 9: Create Authentication Components (continued)

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

// Auth API calls
export const register = async (userData) => {
  const response = await api.post('/auth/register', userData);
  return response.data;
};

export const login = async (credentials) => {
  const response = await api.post('/auth/login', credentials);
  return response.data;
};

export const getCurrentUser = async () => {
  const response = await api.get('/auth/me');
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

// Set auth token for every request
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
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

Update `frontend/src/components/PostForm.jsx` to work with authentication:

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

## Part 12: Testing and Deployment with Auth

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

## Part 13: Security Enhancements

Let's add some important security features to our application:

### Step 1: Add CORS Configuration

Update `backend/server.js` with more specific CORS settings:

```javascript
// CORS configuration
const corsOptions = {
  origin: process.env.NODE_ENV === 'production' 
    ? process.env.FRONTEND_URL 
    : ['http://localhost:3000', 'http://127.0.0.1:3000'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400 // 24 hours
};

app.use(cors(corsOptions));
```

### Step 2: Add Rate Limiting

Install required package:

```bash
cd backend
npm install express-rate-limit
```

Add rate limiting to `backend/server.js`:

```javascript
const rateLimit = require('express-rate-limit');

// Rate limiting
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  standardHeaders: true, // Return rate limit info in the `RateLimit-*` headers
  legacyHeaders: false, // Disable the `X-RateLimit-*` headers
  message: 'Too many requests from this IP, please try again after 15 minutes'
});

// Apply rate limiting to auth routes
app.use('/api/v1/auth', apiLimiter);
```

### Step 3: Add Password Reset Functionality

Create `backend/controllers/resetController.js`:

```javascript
const crypto = require('crypto');
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Get token secret from environment variables
const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';

// Generate reset token
exports.forgotPassword = async (req, res) => {
  try {
    const { email } = req.body;
    
    // Find user by email
    const user = await User.findOne({ email });
    
    if (!user) {
      return res.status(404).json({
        success: false,
        message: 'User not found with this email'
      });
    }
    
    // Generate reset token
    const resetToken = crypto.randomBytes(20).toString('hex');
    
    // Set token and expiry on user
    user.resetPasswordToken = crypto
      .createHash('sha256')
      .update(resetToken)
      .digest('hex');
    user.resetPasswordExpire = Date.now() + 10 * 60 * 1000; // 10 minutes
    
    await user.save();
    
    // In a real application, you would send an email with the reset link
    // For this tutorial, we'll just return the token
    
    res.status(200).json({
      success: true,
      message: 'Password reset token generated',
      resetToken
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: error.message
    });
  }
};

// Reset password
exports.resetPassword = async (req, res) => {
  try {
    // Get token from params
    const { token } = req.params;
    const { password } = req.body;
    
    // Hash token from params
    const resetPasswordToken = crypto
      .createHash('sha256')
      .update(token)
      .digest('hex');
    
    // Find user by token and check if token is expired
    const user = await User.findOne({
      resetPasswordToken,
      resetPasswordExpire: { $gt: Date.now() }
    });
    
    if (!user) {
      return res.status(400).json({
        success: false,
        message: 'Invalid or expired token'
      });
    }
    
    // Set new password
    user.password = password;
    user.resetPasswordToken = undefined;
    user.resetPasswordExpire = undefined;
    
    await user.save();
    
    // Generate JWT token for auto login
    const jwtToken = jwt.sign({ id: user._id }, JWT_SECRET, {
      expiresIn: process.env.JWT_EXPIRE || '30d'
    });
    
    res.status(200).json({
      success: true,
      message: 'Password reset successfully',
      token: jwtToken,
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
```

Update `backend/models/User.js` to include reset token fields:

```javascript
const userSchema = new mongoose.Schema({
  // ... existing fields
  
  resetPasswordToken: String,
  resetPasswordExpire: Date
}, { timestamps: true });
```

Create `backend/routes/resetRoutes.js`:

```javascript
const express = require('express');
const router = express.Router();
const { forgotPassword, resetPassword } = require('../controllers/resetController');

router.post('/forgot-password', forgotPassword);
router.post('/reset-password/:token', resetPassword);

module.exports = router;
```

Update `backend/server.js` to include reset routes:

```javascript
const resetRoutes = require('./routes/resetRoutes');

// ... existing routes
app.use('/api/v1/auth/reset', resetRoutes);
```

## Part 14: Adding Pagination and Filtering

Let's enhance the backend API with pagination and filtering features:

### Step 1: Update the Blog Routes for Pagination

Modify `backend/routes/blogRoutes.js` to include pagination:

```javascript
// GET all posts with pagination and filtering
router.get('/', async (req, res) => {
  try {
    const { page = 1, limit = 10, search, sortBy = 'createdAt', order = 'desc' } = req.query;
    
    // Build query
    const query = {};
    
    // Add search functionality
    if (search) {
      query.$or = [
        { title: { $regex: search, $options: 'i' } },
        { content: { $regex: search, $options: 'i' } }
      ];
    }
    
    // Count total documents
    const total = await Post.countDocuments(query);
    
    // Build sort options
    const sortOptions = {};
    sortOptions[sortBy] = order === 'asc' ? 1 : -1;
    
    // Fetch posts with pagination
    const posts = await Post.find(query)
      .sort(sortOptions)
      .limit(Number(limit))
      .skip((Number(page) - 1) * Number(limit));
    
    res.status(200).json({
      posts,
      totalPages: Math.ceil(total / limit),
      currentPage: Number(page),
      totalPosts: total
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Step 2: Update Frontend to Support Pagination

Create `frontend/src/components/Pagination.jsx`:

```jsx
import { Link } from 'react-router-dom';

function Pagination({ currentPage, totalPages, onPageChange }) {
  const pages = [];
  
  // Create page numbers array
  for (let i = 1; i <= totalPages; i++) {
    pages.push(i);
  }
  
  // Limit number of visible page links
  let visiblePages = pages;
  if (totalPages > 5) {
    if (currentPage <= 3) {
      visiblePages = [...pages.slice(0, 5), '...', totalPages];
    } else if (currentPage >= totalPages - 2) {
      visiblePages = [1, '...', ...pages.slice(totalPages - 5)];
    } else {
      visiblePages = [1, '...', currentPage - 1, currentPage, currentPage + 1, '...', totalPages];
    }
  }
  
  return (
    <div className="pagination">
      <button 
        onClick={() => onPageChange(currentPage - 1)}
        disabled={currentPage === 1}
        className="pagination-arrow"
      >
        &laquo; Prev
      </button>
      
      <div className="pagination-numbers">
        {visiblePages.map((page, index) => (
          page === '...' ? (
            <span key={`ellipsis-${index}`} className="ellipsis">...</span>
          ) : (
            <button
              key={page}
              onClick={() => onPageChange(page)}
              className={`pagination-number ${currentPage === page ? 'active' : ''}`}
            >
              {page}
            </button>
          )
        ))}
      </div>
      
      <button 
        onClick={() => onPageChange(currentPage + 1)}
        disabled={currentPage === totalPages}
        className="pagination-arrow"
      >
        Next &raquo;
      </button>
    </div>
  );
}

export default Pagination;
```

Update `frontend/src/components/PostList.jsx` to include pagination:

```jsx
import { useState, useEffect } from 'react';
import { Link, useSearchParams } from 'react-router-dom';
import { getPosts, deletePost } from '../services/api';
import { useAuth } from '../context/AuthContext';
import Pagination from './Pagination';

function PostList() {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [totalPages, setTotalPages] = useState(1);
  const [searchParams, setSearchParams] = useSearchParams();
  const { currentUser, isAuthenticated } = useAuth();

  // Get current page from URL or default to 1
  const currentPage = Number(searchParams.get('page') || 1);
  const searchQuery = searchParams.get('search') || '';

  const fetchPosts = async () => {
    try {
      setLoading(true);
      
      // Build query parameters
      const params = new URLSearchParams();
      params.append('page', currentPage);
      params.append('limit', 10);
      
      if (searchQuery) {
        params.append('search', searchQuery);
      }
      
      const response = await api.get(`/posts?${params.toString()}`);
      setPosts(response.data.posts);
      setTotalPages(response.data.totalPages);
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
  }, [currentPage, searchQuery]);

  const handlePageChange = (page) => {
    // Update URL with new page number
    const newParams = new URLSearchParams(searchParams);
    newParams.set('page', page);
    setSearchParams(newParams);
  };

  const handleSearch = (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);
    const searchValue = formData.get('search');
    
    // Update URL with search query and reset to page 1
    const newParams = new URLSearchParams();
    if (searchValue) {
      newParams.set('search', searchValue);
    }
    newParams.set('page', 1);
    setSearchParams(newParams);
  };

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
      <div className="post-list-header">
        <h1>Blog Posts</h1>
        
        {isAuthenticated && (
          <Link to="/posts/new" className="button">Create New Post</Link>
        )}
      </div>
      
      <div className="search-bar">
        <form onSubmit={handleSearch}>
          <input 
            type="text"
            name="search"
            placeholder="Search posts..."
            defaultValue={searchQuery}
          />
          <button type="submit">Search</button>
        </form>
      </div>
      
      {posts.length === 0 ? (
        <p>No posts found. {isAuthenticated ? 'Create your first post!' : 'Login to create a post!'}</p>
      ) : (
        <>
          <div className="posts-container">
            {posts.map(post => (
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
            ))}
          </div>
          
          {totalPages > 1 && (
            <Pagination 
              currentPage={currentPage}
              totalPages={totalPages}
              onPageChange={handlePageChange}
            />
          )}
        </>
      )}
    </div>
  );
}

export default PostList;
```

Add styles for pagination in `frontend/src/App.css`:

```css
/* Pagination */
.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 2rem;
}

.pagination-numbers {
  display: flex;
  gap: 0.5rem;
}

.pagination-number, .pagination-arrow {
  padding: 0.5rem 0.75rem;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  background: white;
  cursor: pointer;
}

.pagination-number.active {
  background: var(--primary-color);
  color: white;
  border-color: var(--primary-color);
}

.pagination-arrow:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.ellipsis {
  padding: 0.5rem 0.25rem;
}

/* Search Bar */
.search-bar {
  margin: 1.5rem 0;
}

.search-bar form {
  display: flex;
  gap: 0.5rem;
}

.search-bar input {
  flex: 1;
  padding: 0.75rem;
  border: 1px solid var(--border-color);
  border-radius: 4px;
}

.search-bar button {
  padding: 0.75rem 1.5rem;
  background: var(--secondary-color);
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.post-list-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
```

## Part 15: Implementing Comment System

Let's add a comment system to our blog:

### Step 1: Create Comment Model

Create `backend/models/Comment.js`:

```javascript
const mongoose = require('mongoose');

const commentSchema = new mongoose.Schema({
  content: {
    type: String,
    required: true,
    trim: true
  },
  post: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Post',
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

module.exports = mongoose.model('Comment', commentSchema);
```

### Step 2: Create Comment Routes

Create `backend/routes/commentRoutes.js`:

```javascript
const express = require('express');
const router = express.Router();
const Comment = require('../models/Comment');
const Post = require('../models/Post');
const { protect } = require('../middleware/auth');

// GET all comments for a post
router.get('/post/:postId', async (req, res) => {
  try {
    const comments = await Comment.find({ post: req.params.postId })
      .sort({ createdAt: -1 });
    
    res.status(200).json(comments);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// CREATE comment - protected
router.post('/', protect, async (req, res) => {
  try {
    // Check if post exists
    const post = await Post.findById(req.body.post);
    if (!post) {
      return res.status(404).json({ message: 'Post not found' });
    }
    
    const comment = new Comment({
      content: req.body.content,
      post: req.body.post,
      author: req.user._id,
      authorName: req.user.username
    });
    
    const savedComment = await comment.save();
    res.status(201).json(savedComment);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// DELETE comment - protected (only comment owner or admin)
router.delete('/:id', protect, async (req, res) => {
  try {
    const comment = await Comment.findById(req.params.id);
    
    if (!comment) {
      return res.status(404).json({ message: 'Comment not found' });
    }
    
    // Check if user is comment owner or admin
    if (comment.author.toString() !== req.user.id && req.user.role !== 'admin') {
      return res.status(403).json({ message: 'Not authorized to delete this comment' });
    }
    
    await Comment.findByIdAndDelete(req.params.id);
    res.status(200).json({ message: 'Comment deleted successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### Step 3: Update Server.js to Include Comment Routes

```javascript
const commentRoutes = require('./routes/commentRoutes');

// ... existing routes
app.use('/api/v1/comments', commentRoutes);
```

### Step 4: Update Frontend API Service

Add to `frontend/src/services/api.js`:

```javascript
// Comment API calls
export const getComments = async (postId) => {
  const response = await api.get(`/comments/post/${postId}`);
  return response.data;
};

export const createComment = async (comment) => {
  const response = await api.post('/comments', comment);
  return response.data;
};

export const deleteComment = async (id) => {
  const response = await api.delete(`/comments/${id}`);
  return response.data;
};
```

### Step 5: Create Comment Components

Create `frontend/src/components/CommentList.jsx`:

```jsx
import { useState, useEffect } from 'react';
import { getComments, deleteComment } from '../services/api';
import CommentForm from './CommentForm';
import { useAuth } from '../context/AuthContext';

function CommentList({ postId }) {
  const [comments, setComments] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const { currentUser, isAuthenticated } = useAuth();

  const fetchComments = async () => {
    try {
      setLoading(true);
      const data = await getComments(postId);
      setComments(data);
      setError(null);
    } catch (err) {
      setError('Failed to fetch comments');
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchComments();
  }, [postId]);

  const handleDeleteComment = async (id) => {
    if (window.confirm('Are you sure you want to delete this comment?')) {
      try {
        await deleteComment(id);
        setComments(comments.filter(comment => comment._id !== id));
      } catch (err) {
        setError('Failed to delete comment');
        console.error(err);
      }
    }
  };

  const handleAddComment = (newComment) => {
    setComments([newComment, ...comments]);
  };

  if (loading) return <div>Loading comments...</div>;
  if (error) return <div className="error">{error}</div>;

  return (
    <div className="comments-section">
      <h3>Comments ({comments.length})</h3>
      
      {isAuthenticated && (
        <CommentForm postId={postId} onCommentAdded={handleAddComment} />
      )}
      
      {!isAuthenticated && (
        <p className="login-prompt">Please <a href="/login">login</a> to leave a comment.</p>
      )}
      
      {comments.length === 0 ? (
        <p>No comments yet. Be the first to comment!</p>
      ) : (
        <div className="comments-list">
          {comments.map(comment => (
            <div key={comment._id} className="comment">
              <div className="comment-header">
                <span className="comment-author">{comment.authorName}</span>
                <span className="comment-date">{new Date(comment.createdAt).toLocaleDateString()}</span>
              </div>
              <div className="comment-content">{comment.content}</div>
              
              {/* Only show delete for comment owner or admin */}
              {currentUser && (currentUser.id === comment.author || currentUser.role === 'admin') && (
                <button 
                  onClick={() => handleDeleteComment(comment._id)}
                  className="delete-comment"
                >
                  Delete
                </button>
              )}
            </div>
          ))}
        </div>
      )}
    </div>
  );
}

export default CommentList;
```

Create `frontend/src/components/CommentForm.jsx`:

```jsx
import { useState } from 'react';
import { createComment } from '../services/api';

function CommentForm({ postId, onCommentAdded }) {
  const [content, setContent] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    if (!content.trim()) return;
    
    try {
      setLoading(true);
      setError(null);
      
      const newComment = await createComment({
        content,
        post: postId
      });
      
      setContent('');
      onCommentAdded(newComment);
    } catch (err) {
      setError('Failed to add comment');
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="comment-form">
      {error && <div className="error">{error}</div>}
      
      <form onSubmit={handleSubmit}>
        <div className="form-group">
          <label htmlFor="comment">Add a comment</label>
          <textarea
            id="comment"
            value={content}
            onChange={(e) => setContent(e.target.value)}
            rows="3"
            required
            disabled={loading}
          />
        </div>
        
        <button type="submit" disabled={loading || !content.trim()}>
          {loading ? 'Posting...' : 'Post Comment'}
        </button>
      </form>
    </div>
  );
}

export default CommentForm;
```

### Step 6: Update PostDetail to Include Comments

Update `frontend/src/components/PostDetail.jsx`:

```jsx
import { useState, useEffect } from 'react';
import { useParams, Link, useNavigate } from 'react-router-dom';
import { getPost, deletePost } from '../services/api';
import { useAuth } from '../context/AuthContext';
import CommentList from './CommentList';

function PostDetail() {
  const { id } = useParams();
  const navigate = useNavigate();
  const { currentUser } = useAuth();
  
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
          By {post.authorName} • {new Date(post.createdAt).toLocaleDateString()}
        </p>
      </div>
      
      <div className="post-content">
        <p>{post.content}</p>
      </div>
      
      <div className="post-actions">
        <Link to="/">Back to Posts</Link>
        
        {/* Only show edit/delete for post owner or admin */}
        {currentUser && (currentUser.id === post.author || currentUser.role === 'admin') && (
          <>
            <Link to={`/posts/edit/${post._id}`}>Edit</Link>
            <button onClick={handleDelete}>Delete</button>
          </>
        )}
      </div>
      
      {/* Comments section */}
      <div className="divider"></div>
      <CommentList postId={post._id} />
    </div>
  );
}

export default PostDetail;
```

### Step 7: Add CSS for Comments

Add to `frontend/src/App.css`:

```css
/* Comments */
.comments-section {
  margin-top: 2rem;
}

.comments-section h3 {
  margin-bottom: 1rem;
}

.comment-form {
  margin-bottom: 2rem;
}

.comment-form textarea {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  font-family: inherit;
  font-size: 1rem;
  margin-bottom: 0.5rem;
}

.comment-form button {
  padding: 0.5rem 1.5rem;
  background: var(--accent-color);
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.comment-form button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.comments-list {
  margin-top: 1.5rem;
}

.comment {
  background: #f8f9fa;
  border-radius: 8px;
  padding: 1rem;
  margin-bottom: 1rem;
}

.comment-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.5rem;
}

.comment-author {
  font-weight: 500;
}

.comment-date {
  color: #666;
  font-size: 0.9rem;
}

.comment-content {
  line-height: 1.5;
}

.delete-comment {
  margin-top: 0.5rem;
  background: none;
  border: none;
  color: #d32f2f;
  cursor: pointer;
  font-size: 0.9rem;
  padding: 0;
}

.login-prompt {
  margin: 1.5rem 0;
  font-style: italic;
}

.divider {
  border-top: 1px solid var(--border-color);
  margin: 2rem 0;
}
```

## Part 16: Optimizing for Production

Let's add some final optimizations for production:

### Step 1: Frontend Optimization

Update `frontend/vite.config.js`:

```javascript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { compression } from 'vite-plugin-compression';

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [
    react(),
    compression({
      algorithm: 'gzip',
      ext: '.gz',
    }),
    compression({
      algorithm: 'brotliCompress',
      ext: '.br',
    }),
  ],
  build: {
    minify: 'esbuild',
    sourcemap: false,
    chunkSizeWarningLimit: 1000,
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', 'react-router-dom'],
          utils: ['axios'],
        },
      },
    },
  },
  server: {
    port: 3000,
    strictPort: true,
    proxy: {
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true,
      },
    },
  },
});
```

Install required plugin:

```bash
cd frontend
npm install vite-plugin-compression --save-dev
```

### Step 2: Backend Optimization

Install required packages:

```bash
cd backend
npm install compression helmet
```

Update `backend/server.js`:

```javascript
const compression = require('compression');

// Add compression middleware for all routes
app.use(compression());

// Update helmet configuration for better security
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:'],
      connectSrc: ["'self'", process.env.FRONTEND_URL || 'http://localhost:3000'],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"],
    },
  },
  xssFilter: true,
  noSniff: true,
  referrerPolicy: { policy: 'same-origin' },
}));
```

### Step 3: Implement Error Logging

Install required package:

```bash
cd backend
npm install winston
```

Create `backend/utils/logger.js`:

```javascript
const winston = require('winston');
const path = require('path');
const fs = require('fs');

// Create logs directory if it doesn't exist
const logDir = 'logs';
if (!fs.existsSync(logDir)) {
  fs.mkdirSync(logDir);
}

// Define log format
const logFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  winston.format.errors({ stack: true }),
  winston.format.splat(),
  winston.format.json()
);

// Create logger
const logger = winston.createLogger({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  format: logFormat,
  defaultMeta: { service: 'blog-api' },
  transports: [
    // Write logs with level 'error' and below to error.log
    new winston.transports.File({ 
      filename: path.join(logDir, 'error.log'), 
      level: 'error' 
    }),
    // Write all logs to combined.log
    new winston.transports.File({ 
      filename: path.join(logDir, 'combined.log') 
    }),
  ],
});

// Add console transport in development
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple()
    ),
  }));
}

module.exports = logger;
```

Update error handling in `backend/server.js`:

```javascript
const logger = require('./utils/logger');

// Error handling middleware
app.use((err, req, res, next) => {
  logger.error(`${err.status || 500} - ${err.message} - ${req.originalUrl} - ${req.method} - ${req.ip}`);
  
  res.status(err.status || 500).json({ 
    error: process.env.NODE_ENV === 'production' 
      ? 'Something went wrong!' 
      : err.message 
  });
});

// 404 handler
app.use((req, res) => {
  logger.warn(`404 - ${req.originalUrl} - ${req.method} - ${req.ip}`);
  res.status(404).json({ message: 'Route not found' });
});
```

## Part 17: Final Docker Compose Setup

Update `docker-compose.yml` for production:

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:latest
    container_name: blog-mongodb
    volumes:
      - mongodb_data:/data/db
    restart: always
    networks:
      - blog-network
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_USERNAME}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD}
      - MONGO_INITDB_DATABASE=blog
    ports:
      - "27017:27017"

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: blog-backend
    restart: always
    depends_on:
      - mongodb
    environment:
      - PORT=3001
      - DATABASE_URL=mongodb://${MONGO_USERNAME}:${MONGO_PASSWORD}@mongodb:27017/blog?authSource=admin
      - NODE_ENV=${NODE_ENV:-production}
      - JWT_SECRET=${JWT_SECRET}
      - JWT_EXPIRE=30d
      - FRONTEND_URL=${FRONTEND_URL:-http://localhost:3000}
    ports:
      - "3001:3001"
    networks:
      - blog-network

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        - VITE_BACKEND_URL=${BACKEND_URL:-http://localhost:3001/api/v1}
    container_name: blog-frontend
    restart: always
    depends_on:
      - backend
    ports:
      - "3000:80"
    networks:
      - blog-network

networks:
  blog-network:
    driver: bridge

volumes:
  mongodb_data:
```

Create `.env` file in the root directory:

```
# MongoDB
MONGO_USERNAME=blog_user
MONGO_PASSWORD=securepassword123

# Backend
NODE_ENV=production
JWT_SECRET=your-super-secure-and-long-secret-key-here

# URLs
FRONTEND_URL=http://localhost:3000
BACKEND_URL=http://localhost:3001/api/v1
```

## Conclusion

Congratulations! You've successfully built a complete full-stack blog application with:

- Modern frontend using React with Vite
- Robust backend API with Express and MongoDB
- User authentication with JWT
- Comment system with proper authorization
- Pagination and search functionality
- Docker containerization for consistent environments
- CI/CD pipeline with GitHub Actions
- Cloud deployment on AWS

This application provides a solid foundation that you can build upon and extend with additional features like:

1. **Image uploads**: Add ability to upload and store images for blog posts
2. **Rich text editor**: Replace plain text areas with a WYSIWYG editor
3. **Categories and tags**: Add ability to categorize posts
4. **Social sharing**: Add buttons to share posts on social media
5. **Analytics**: Implement page view tracking and user analytics
6. **Email notifications**: Send notifications for new comments
7. **Two-factor authentication**: Add extra security layer for user accounts

The containerized approach ensures your application will run consistently across different environments, making it easier to develop, test, and deploy. The CI/CD pipeline automates the testing and deployment process, allowing you to focus on adding features and fixing bugs.

Remember to regularly update your dependencies to keep your application secure and perform regular backups of your database to prevent data loss.
