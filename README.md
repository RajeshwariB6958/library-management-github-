<img width="1533" height="787" alt="12" src="https://github.com/user-attachments/assets/29123086-2e5f-46e5-86d8-5da5094a58d1" />

 
 
 
 
 Step 1 — Setup
bash# Create project folder
mkdir library_api && cd library_api



<img width="1210" height="848" alt="rest-api" src="https://github.com/user-attachments/assets/4fcad5d4-4ede-4d3c-a390-b0746a8527d3" />


# Create virtual environment (isolated Python sandbox)
python -m venv venv
source venv/bin/activate        # Mac/Linux
venv\Scripts\activate           # Windows

# Install dependencies
pip install django djangorestframework
Start the Django project:
bashdjango-admin startproject library_project .
python manage.py startapp library
Your folder now looks like:
library_api/
├── library_project/   ← project settings
├── library/           ← our app (models, views, urls)
└── manage.py

Step 2 — Settings (library_project/settings.py)
Add rest_framework and our app to INSTALLED_APPS:
pythonINSTALLED_APPS = [
    ...
    'rest_framework',   # DRF - Django REST Framework
    'library',          # our app
]

Step 3 — Models (library/models.py)

Model = a Python class that maps to a database table. Think of it like a struct in C.

pythonfrom django.db import models

class Book(models.Model):
    title       = models.CharField(max_length=200)
    author      = models.CharField(max_length=100)
    isbn        = models.CharField(max_length=13, unique=True)  # unique = no duplicates
    total_copies = models.IntegerField(default=1)
    available_copies = models.IntegerField(default=1)

    def __str__(self):
        return self.title   # shows "Clean Code" instead of "Book object(1)"


class Member(models.Model):
    name  = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    joined_on = models.DateField(auto_now_add=True)  # auto-set on creation

    def __str__(self):
        return self.name


class BorrowRecord(models.Model):
    # ForeignKey = a relational link (like a pointer to another table)
    book      = models.ForeignKey(Book, on_delete=models.CASCADE)
    member    = models.ForeignKey(Member, on_delete=models.CASCADE)
    borrow_date = models.DateField(auto_now_add=True)
    return_date = models.DateField(null=True, blank=True)  # null = not returned yet
    is_returned = models.BooleanField(default=False)

    def __str__(self):
        return f"{self.member.name} → {self.book.title}"
Now run migrations (creates actual DB tables):
bashpython manage.py makemigrations
python manage.py migrate

Step 4 — Serializers (library/serializers.py)

Serializer = converts Python objects ↔ JSON. Like printf formatting but for APIs. This is the lingua franca (common language) between your DB and the outside world.

Create the file:
pythonfrom rest_framework import serializers
from .models import Book, Member, BorrowRecord

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model  = Book
        fields = '__all__'   # include all fields in JSON


class MemberSerializer(serializers.ModelSerializer):
    class Meta:
        model  = Member
        fields = '__all__'


class BorrowRecordSerializer(serializers.ModelSerializer):
    # These add readable names instead of just IDs in the response
    book_title  = serializers.CharField(source='book.title', read_only=True)
    member_name = serializers.CharField(source='member.name', read_only=True)

    class Meta:
        model  = BorrowRecord
        fields = ['id', 'book', 'book_title', 'member', 'member_name',
                  'borrow_date', 'return_date', 'is_returned']

Step 5 — Views (library/views.py)

ViewSet = handles all CRUD (Create, Read, Update, Delete) operations automatically. Very pragmatic (practical/efficient) — one class does the job of 5 functions.

pythonfrom rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import Book, Member, BorrowRecord
from .serializers import BookSerializer, MemberSerializer, BorrowRecordSerializer
from django.utils import timezone

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer


class MemberViewSet(viewsets.ModelViewSet):
    queryset = Member.objects.all()
    serializer_class = MemberSerializer


class BorrowViewSet(viewsets.ModelViewSet):
    queryset = BorrowRecord.objects.all()
    serializer_class = BorrowRecordSerializer

    # Custom action: POST /borrows/{id}/return_book/
    @action(detail=True, methods=['post'])
    def return_book(self, request, pk=None):
        record = self.get_object()  # get this specific borrow record

        if record.is_returned:
            return Response(
                {'error': 'This book was already returned'},
                status=status.HTTP_400_BAD_REQUEST
            )

        # Mark as returned and restore copy count
        record.is_returned = True
        record.return_date  = timezone.now().date()
        record.book.available_copies += 1
        record.book.save()
        record.save()

        return Response({'message': 'Book returned successfully!'})

    # Override create to handle borrow logic
    def create(self, request, *args, **kwargs):
        book_id   = request.data.get('book')
        member_id = request.data.get('member')

        book = Book.objects.get(id=book_id)

        if book.available_copies < 1:
            return Response(
                {'error': 'No copies available right now'},
                status=status.HTTP_400_BAD_REQUEST
            )

        # Deduct one available copy
        book.available_copies -= 1
        book.save()

        return super().create(request, *args, **kwargs)

Step 6 — URLs (library/urls.py)
Create this file:
pythonfrom django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import BookViewSet, MemberViewSet, BorrowViewSet

# Router auto-generates all URL patterns for us
router = DefaultRouter()
router.register(r'books',   BookViewSet)
router.register(r'members', MemberViewSet)
router.register(r'borrows', BorrowViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
Now hook it to the main project (library_project/urls.py):
pythonfrom django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('library.urls')),   # all our routes under /api/
]

Step 7 — Run and Test
bashpython manage.py runserver
API Endpoints Generated Automatically:
MethodURLActionGET/api/books/List all booksPOST/api/books/Add a bookGET/api/books/1/Get book with id=1PUT/api/books/1/Update bookDELETE/api/books/1/Delete bookPOST/api/borrows/Borrow a bookPOST/api/borrows/1/return_book/Return a book
Test with curl:
bash# Add a book
curl -X POST http://127.0.0.1:8000/api/books/ \
  -H "Content-Type: application/json" \
  -d '{"title":"Clean Code","author":"Robert Martin","isbn":"9780132350884","total_copies":3,"available_copies":3}'

# List all books
curl http://127.0.0.1:8000/api/books/

# Borrow book (book id=1, member id=1)
curl -X POST http://127.0.0.1:8000/api/borrows/ \
  -H "Content-Type: application/json" \
  -d '{"book":1,"member":1}'

# Return it
curl -X POST http://127.0.0.1:8000/api/borrows/1/return_book/

🧠 Quick Mental Model (for C programmers)
C conceptDjango equivalentstructModelprintf formattingSerializerfunction handling a routeViewSetmain() routerurls.py
