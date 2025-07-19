# Building Your First Django Backend - Complete Tutorial

## What is Django?
Django is a high-level Python web framework that encourages rapid development and clean, pragmatic design. It follows the "batteries included" philosophy, providing many built-in features.

## Prerequisites
- Python 3.8+ installed
- Basic understanding of Python
- Command line familiarity

## Step 1: Setup Environment

### Create a Virtual Environment
```powershell
# Create a new directory for your project
mkdir my_blog_project
cd my_blog_project

# Create virtual environment
python -m venv blog_env

# Activate virtual environment (Windows PowerShell)
blog_env\Scripts\activate

# If you get execution policy error, run:
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### Install Django
```powershell
pip install django
```

## Step 2: Create Django Project

```powershell
# Create Django project
django-admin startproject core .

# Create Django app
python manage.py startapp blog
```

## Step 3: Project Structure
After creation, your structure will look like:
```
my_blog_project/
├── blog_env/
├── blog_project/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
├── blog/
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── models.py
│   ├── tests.py
│   ├── views.py
│   └── migrations/
└── manage.py
```

## Step 4: Configure Settings

### Add App to Settings
Edit `blog_project/settings.py`:
```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog',  # Add your app here
]
```

## Step 5: Create Models (Database Structure)

Edit `blog/models.py`:
```python
from django.db import models
from django.contrib.auth.models import User
from django.utils import timezone

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    created_date = models.DateTimeField(default=timezone.now)
    published_date = models.DateTimeField(blank=True, null=True)

    def publish(self):
        self.published_date = timezone.now()
        self.save()

    def __str__(self):
        return self.title

    class Meta:
        ordering = ['-created_date']

class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.CharField(max_length=100)
    content = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)

    def __str__(self):
        return f'Comment by {self.author} on {self.post.title}'
```

## Step 6: Create and Apply Migrations

```powershell
# Create migration files
python manage.py makemigrations

# Apply migrations to database
python manage.py migrate

# Create superuser for admin access
python manage.py createsuperuser
```

## Step 7: Register Models in Admin

Edit `blog/admin.py`:
```python
from django.contrib import admin
from .models import Post, Comment

@admin.register(Post)
class PostAdmin(admin.ModelAdmin):
    list_display = ['title', 'author', 'created_date', 'published_date']
    list_filter = ['created_date', 'published_date', 'author']
    search_fields = ['title', 'content']
    prepopulated_fields = {'slug': ('title',)}
    raw_id_fields = ['author']
    date_hierarchy = 'published_date'
    ordering = ['published_date']

@admin.register(Comment)
class CommentAdmin(admin.ModelAdmin):
    list_display = ['author', 'post', 'created_date']
    list_filter = ['created_date']
    search_fields = ['author', 'content']
```

## Step 8: Create Views (Business Logic)

Edit `blog/views.py`:
```python
from django.shortcuts import render, get_object_or_404
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator
from django.views import View
import json
from .models import Post, Comment

# Function-based views
def post_list(request):
    """Return all published posts"""
    posts = Post.objects.filter(published_date__isnull=False)
    posts_data = []
    
    for post in posts:
        posts_data.append({
            'id': post.id,
            'title': post.title,
            'content': post.content,
            'author': post.author.username,
            'created_date': post.created_date.isoformat(),
            'published_date': post.published_date.isoformat() if post.published_date else None,
        })
    
    return JsonResponse({'posts': posts_data})

def post_detail(request, post_id):
    """Return specific post with comments"""
    post = get_object_or_404(Post, id=post_id, published_date__isnull=False)
    comments = post.comments.all()
    
    post_data = {
        'id': post.id,
        'title': post.title,
        'content': post.content,
        'author': post.author.username,
        'created_date': post.created_date.isoformat(),
        'published_date': post.published_date.isoformat(),
        'comments': [
            {
                'id': comment.id,
                'author': comment.author,
                'content': comment.content,
                'created_date': comment.created_date.isoformat()
            }
            for comment in comments
        ]
    }
    
    return JsonResponse(post_data)

# Class-based view for handling POST requests
@method_decorator(csrf_exempt, name='dispatch')
class PostCreateView(View):
    def post(self, request):
        """Create a new post"""
        try:
            data = json.loads(request.body)
            post = Post.objects.create(
                title=data['title'],
                content=data['content'],
                author_id=data['author_id']
            )
            return JsonResponse({
                'message': 'Post created successfully',
                'post_id': post.id
            }, status=201)
        except Exception as e:
            return JsonResponse({'error': str(e)}, status=400)

@method_decorator(csrf_exempt, name='dispatch')
class CommentCreateView(View):
    def post(self, request):
        """Add comment to a post"""
        try:
            data = json.loads(request.body)
            comment = Comment.objects.create(
                post_id=data['post_id'],
                author=data['author'],
                content=data['content']
            )
            return JsonResponse({
                'message': 'Comment added successfully',
                'comment_id': comment.id
            }, status=201)
        except Exception as e:
            return JsonResponse({'error': str(e)}, status=400)
```

## Step 9: Configure URLs

Create `blog/urls.py`:
```python
from django.urls import path
from . import views

app_name = 'blog'

urlpatterns = [
    path('posts/', views.post_list, name='post_list'),
    path('posts/<int:post_id>/', views.post_detail, name='post_detail'),
    path('posts/create/', views.PostCreateView.as_view(), name='post_create'),
    path('comments/create/', views.CommentCreateView.as_view(), name='comment_create'),
]
```

Edit `core/urls.py`:
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.urls')),
]
```

## Step 10: Test Your Backend

### Start the Development Server
```powershell
python manage.py runserver
```

### Test Endpoints

1. **Admin Interface**: http://127.0.0.1:8000/admin/
2. **API Endpoints**:
   - GET http://127.0.0.1:8000/api/posts/ (List all posts)
   - GET http://127.0.0.1:8000/api/posts/1/ (Get specific post)

### Create Test Data via Admin
1. Go to http://127.0.0.1:8000/admin/
2. Login with superuser credentials
3. Create some posts and publish them
4. Test your API endpoints

## Step 11: Advanced Features

### Add Serializers (Optional - using Django REST Framework)
```powershell
pip install djangorestframework
```

Add to `settings.py`:
```python
INSTALLED_APPS = [
    # ... existing apps
    'rest_framework',
]
```

Create `blog/serializers.py`:
```python
from rest_framework import serializers
from .models import Post, Comment

class CommentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Comment
        fields = '__all__'

class PostSerializer(serializers.ModelSerializer):
    comments = CommentSerializer(many=True, read_only=True)
    
    class Meta:
        model = Post
        fields = '__all__'
```

## Step 12: Environment Variables (Production Ready)

Create `.env` file:
```
SECRET_KEY=your-secret-key-here
DEBUG=True
DATABASE_URL=sqlite:///db.sqlite3
```

Install python-decouple:
```powershell
pip install python-decouple
```

Update `settings.py`:
```python
from decouple import config

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
```

## Project Structure Summary

```
my_blog_project/
├── blog_env/                 # Virtual environment
├── blog_project/            # Main project settings
│   ├── settings.py         # Project configuration
│   ├── urls.py            # Main URL routing
│   └── wsgi.py            # WSGI configuration
├── blog/                   # Blog application
│   ├── models.py          # Database models
│   ├── views.py           # Business logic
│   ├── urls.py            # App URL routing
│   ├── admin.py           # Admin configuration
│   └── migrations/        # Database migrations
├── manage.py              # Django management script
└── db.sqlite3            # SQLite database
```

## Key Concepts Explained

### Models
- Define your database structure
- Each model class represents a database table
- Fields define columns and data types

### Views
- Handle HTTP requests and responses
- Contain business logic
- Can be function-based or class-based

### URLs
- Map URLs to views
- Allow clean, maintainable URL structure

### Admin
- Built-in administration interface
- Automatically generated from models
- Customizable for better UX

## Next Steps

1. **Add Authentication**: Implement user registration/login
2. **Add Pagination**: Handle large datasets efficiently
3. **Add Validation**: Implement proper data validation
4. **Add Tests**: Write unit and integration tests
5. **Deploy**: Deploy to platforms like Heroku, DigitalOcean, or AWS

## Common Commands Reference

```powershell
# Development
python manage.py runserver
python manage.py shell
python manage.py makemigrations
python manage.py migrate

# Database
python manage.py dbshell
python manage.py dumpdata
python manage.py loaddata

# User Management
python manage.py createsuperuser
python manage.py changepassword

# Static Files
python manage.py collectstatic
```

This tutorial provides a solid foundation for building Django backends. Start with this example and gradually add more features as you become comfortable with Django's concepts!
