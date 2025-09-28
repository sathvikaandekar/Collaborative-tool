# Collaborative-tool
A collaborative tool is a software or platform that enables individuals or teams to work together, share information, and coordinate tasks in real-time, regardless of their location. It facilitates communication, organization, and productivity among team members, enhancing overall collaboration and project outcomes.
django-admin startproject pmtool
cd pmtool
python manage.py startapp projects
pip install django channels daphne  # for real-time WebSocket support (optional)
from django.db import models
from django.contrib.auth.models import User

class Project(models.Model):
    name = models.CharField(max_length=255)
    description = models.TextField(blank=True)
    members = models.ManyToManyField(User, related_name='projects')
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name

class Task(models.Model):
    STATUS_CHOICES = [
        ('todo', 'To Do'),
        ('in_progress', 'In Progress'),
        ('done', 'Done'),
    ]

    project = models.ForeignKey(Project, on_delete=models.CASCADE, related_name='tasks')
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True)
    assigned_to = models.ForeignKey(User, null=True, blank=True, on_delete=models.SET_NULL, related_name='tasks')
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='todo')
    due_date = models.DateField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title

class Comment(models.Model):
    task = models.ForeignKey(Task, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    from django.shortcuts import render, redirect, get_object_or_404
from django.contrib.auth.decorators import login_required
from django.contrib.auth import login, logout, authenticate
from django.contrib.auth.forms import UserCreationForm, AuthenticationForm
from .models import Project, Task, Comment
from django.contrib.auth.models import User

def register(request):
    if request.method == 'POST':
        form = UserCreationForm(request.POST)
        if form.is_valid():
            user = form.save()
            login(request, user)
            return redirect('dashboard')
    else:
        form = UserCreationForm()
    return render(request, 'register.html', {'form': form})

def user_login(request):
    if request.method == 'POST':
        form = AuthenticationForm(request, data=request.POST)
        if form.is_valid():
            user = form.get_user()
            login(request, user)
            return redirect('dashboard')
    else:
        form = AuthenticationForm()
    return render(request, 'login.html', {'form': form})

@login_required
def dashboard(request):
    projects = request.user.projects.all()
    return render(request, 'dashboard.html', {'projects': projects})

@login_required
def create_project(request):
    if request.method == 'POST':
        name = request.POST.get('name')
        description = request.POST.get('description')
        project = Project.objects.create(name=name, description=description)
        project.members.add(request.user)
        return redirect('dashboard')
    return render(request, 'create_project.html')

@login_required
def project_detail(request, pk):
    project = get_object_or_404(Project, pk=pk)
    tasks = project.tasks.all()
    return render(request, 'project_detail.html', {'project': project, 'tasks': tasks})

@login_required
def create_task(request, project_pk):
    project = get_object_or_404(Project, pk=project_pk)
    if request.method == 'POST':
        title = request.POST.get('title')
        description = request.POST.get('description')
        assigned_to_id = request.POST.get('assigned_to')
        assigned_user = User.objects.filter(id=assigned_to_id).first() if assigned_to_id else None
        Task.objects.create(project=project, title=title, description=description, assigned_to=assigned_user)
        return redirect('project_detail', pk=project.pk)
    users = project.members.all()
    return render(request, 'create_task.html', {'project': project, 'users': users})

@login_required
def add_comment(request, task_pk):
    task = get_object_or_404(Task, pk=task_pk)
    if request.method == 'POST':
        content = request.POST.get('content')
        Comment.objects.create(task=task, author=request.user, content=content)
        return redirect('project_detail', pk=task.project.pk)
        # pmtool/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('projects.urls')),
]

# projects/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('register/', views.register, name='register'),
    path('login/', views.user_login, name='login'),
    path('logout/', lambda request: (logout(request), redirect('login'))[1], name='logout'),
    path('dashboard/', views.dashboard, name='dashboard'),
    path('project/create/', views.create_project, name='create_project'),
    path('project/<int:pk>/', views.project_detail, name='project_detail'),
    path('project/<int:project_pk>/task/create/', views.create_task, name='create_task'),
    path('task/<int:task_pk>/comment/', views.add_comment, name='add_comment'),
]

        
    
