# Django Authentication va Post CRUD

Django'ning o'rnatilgan autentifikatsiya tizimi (`django.contrib.auth`) yordamida **ro'yxatdan o'tish**, **kirish**, **chiqish** va faqat tizimga kirgan foydalanuvchilarga **post yaratishga ruxsat berish** — bularning barchasini bitta loyihada qanday qilish mumkinligi ko'rsatilgan.

---

## Loyiha tuzilmasi

```
project/
├── your_app/
│   ├── models.py        # Post modeli
│   ├── forms.py         # PostForm
│   ├── views.py         # Barcha view-lar
│   ├── urls.py          # App URL routing
│   └── templates/
│       ├── register.html
│       ├── login.html
│       ├── home.html
│       └── create_post.html
├── project/
│   ├── settings.py      # LOGIN_URL sozlamalari
│   └── urls.py          # Asosiy URL routing
└── manage.py
```

---

## 1. `forms.py` — PostForm

```python
from django import forms


class PostForm(forms.Form):
    title = forms.CharField(max_length=200)
    content = forms.CharField(widget=forms.Textarea)
```

| Nima | Tushuntirish |
|------|-------------|
| `forms.Form` | Oddiy forma (modelga bog'lanmagan). Ma'lumotni qo'lda `cleaned_data` orqali olamiz |
| `forms.CharField` | Matn kiritish maydoni |
| `widget=forms.Textarea` | `content` maydoni `<textarea>` sifatida ko'rsatiladi |

> [!NOTE]
> `ModelForm` ishlatilganda model bilan avtomatik bog'lanadi. Bu yerda oddiy `Form` ishlatilgan — ya'ni `Post.objects.create()` ni qo'lda chaqirish kerak.

---

## 2. `views.py` — Barcha view funksiyalar

### 2.1 `register_view` — Ro'yxatdan o'tish

```python
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth import login

def register_view(request):
    if request.method == 'POST':
        form = UserCreationForm(request.POST)
        if form.is_valid():
            user = form.save()
            login(request, user)          # Darhol tizimga kiritadi
            return redirect('home')
    else:
        form = UserCreationForm()
    return render(request, 'register.html', {'form': form})
```


| Qadamlar | Nima bo'ladi |
|----------|-------------|
| `UserCreationForm(request.POST)` | Django'ning tayyor formasi: `username`, `password1`, `password2` |
| `form.save()` | `User` obyektini bazaga yozadi, parolni **hash** qilib saqlaydi |
| `login(request, user)` | Session yaratadi — foydalanuvchi darhol tizimga kiradi |

> [!IMPORTANT]
> Templateda **`{{ form.as_p }}`** ishlatish shart. Aks holda parol validatsiyasi xatoliklari ko'rinmaydi.

---

### 2.2 `login_view` — Tizimga kirish

```python
from django.contrib.auth.forms import AuthenticationForm

def login_view(request):
    if request.method == 'POST':
        form = AuthenticationForm(request, data=request.POST)
        if form.is_valid():
            user = form.get_user()
            login(request, user)
            return redirect('home')
    else:
        form = AuthenticationForm()
    return render(request, 'login.html', {'form': form})
```

| Element | Tushuntirish |
|---------|-------------|
| `AuthenticationForm(request, data=request.POST)` | Birinchi argument — `request` (CSRF, session uchun), ikkinchisi — `data` |
| `form.get_user()` | Validatsiya muvaffaqiyatli bo'lsa, `User` obyektini qaytaradi |
| `login()` | Session cookie orqali foydalanuvchini tizimga kiritadi |

> [!CAUTION]
> `AuthenticationForm(request.POST)` **noto'g'ri**. `request` birinchi argument sifatida, `data=request.POST` ikkinchi argument sifatida uzatilishi kerak.

---

### 2.3 `logout_view` — Tizimdan chiqish

```python
from django.views.decorators.http import require_POST

@require_POST
def logout_view(request):
    logout(request)
    return redirect('login')
```

| Element | Tushuntirish |
|---------|-------------|
| `@require_POST` | Faqat POST so'rovni qabul qiladi. GET orqali logout — **CSRF zaiflik** |
| `logout(request)` | Sessionni o'chiradi, cookie'ni tozalaydi |

> [!WARNING]
> Logout GET orqali ishlasa, zararli sayt `<img src="/logout/">` qo'yib foydalanuvchini chiqarib yuborishi mumkin (**CSRF hujum**).

---

### 2.4 `home` — Bosh sahifa

```python
def home(request):
    posts = Post.objects.all().order_by('-created_at')
    return render(request, 'home.html', {'posts': posts})
```

- **Hamma ko'rishi mumkin** (login shart emas)
- Postlar **yangiidan eskiga** tartibda (`-created_at`)

---

### 2.5 `create_post` — Post yaratish (faqat tizimga kirganlar)

```python
from django.contrib.auth.decorators import login_required

@login_required
def create_post(request):
    if request.method == 'POST':
        form = PostForm(request.POST)
        if form.is_valid():
            Post.objects.create(
                title=form.cleaned_data['title'],
                content=form.cleaned_data['content'],
                author=request.user
            )
            return redirect('home')
    else:
        form = PostForm()
    return render(request, 'create_post.html', {'form': form})
```

| Element | Tushuntirish |
|---------|-------------|
| `@login_required` | Kirish talab etiladi. Kirmagan foydalanuvchi `LOGIN_URL`ga yo'naltiriladi |
| `PostForm(request.POST)` | Formadan ma'lumotni oladi |
| `form.cleaned_data['title']` | Validatsiyadan o'tgan, tozalangan qiymat |
| `author=request.user` | Hozirgi tizimga kirgan foydalanuvchi avtomatik `author` sifatida yoziladi |

> [!NOTE]
> `forms.Form` ishlatilgani uchun `Post.objects.create()` qo'lda chaqiriladi. `ModelForm` ishlatilsa, `form.save(commit=False)` + `post.author = request.user` + `post.save()` usuli ham bor.

---

## 3. `settings.py` — Sozlamalar

```python
LOGIN_URL = '/login/'
LOGIN_REDIRECT_URL = '/'
```

| Setting | Vazifasi |
|---------|---------|
| `LOGIN_URL` | `@login_required` dan qaytarilganda yo'naltiriladi (**majburiy**) |
| `LOGIN_REDIRECT_URL` | Login muvaffaqiyatli bo'lganda yo'naltirish (faqat Django'ning `LoginView` ishlatilsa) |

---

## 4. Templates

### 4.1 `register.html`

```html
<form method="post">
  {% csrf_token %}
  <h2>Register</h2>
  {{ form.as_p }}
  <button type="submit">Register</button>
  <p>Already have an account? <a href="{% url 'login' %}">Login</a></p>
</form>
```

| Element | Tushuntirish |
|---------|-------------|
| `{% csrf_token %}` | CSRF tokenni formaga qo'shadi — **har bir POST formada shart** |
| `{{ form.as_p }}` | Barcha form maydonlarini `<p>` ichida ko'rsatadi, **xatoliklarni avtomatik chiqaradi** |
| `{% url 'login' %}` | URL ni name bo'yicha dinamik hosil qiladi |

---

### 4.2 `login.html`

```html
<form method="post">
  {% csrf_token %}
  <h2>Login</h2>
  {{ form.as_p }}
  <button type="submit">Login</button>
  <p>No account? <a href="{% url 'register' %}">Register</a></p>
</form>
```

`register.html` bilan bir xil tuzilma. `AuthenticationForm` `username` va `password` maydonlarini avtomatik yaratadi.

---

### 4.3 `home.html`

```html
<div>
  <h2>Posts</h2>

  {% if user.is_authenticated %}
    <a href="{% url 'create_post' %}">Create Post</a>

    <!-- Logout POST orqali -->
    <form method="post" action="{% url 'logout' %}" style="display:inline;">
      {% csrf_token %}
      <button type="submit">Logout</button>
    </form>
    <span>Salom, {{ user.username }}!</span>
  {% else %}
    <a href="{% url 'login' %}">Login</a>
    <a href="{% url 'register' %}">Register</a>
  {% endif %}

  {% for post in posts %}
    <div>
      <h3>{{ post.title }}</h3>
      <p>{{ post.content }}</p>
      <small>{{ post.author }} | {{ post.created_at }}</small>
    </div>
  {% empty %}
    <p>No posts yet</p>
  {% endfor %}
</div>
```

| Element | Tushuntirish |
|---------|-------------|
| `{% if user.is_authenticated %}` | Tizimga kirgan/kirmaganligini tekshiradi |
| Logout `<form>` | POST orqali — CSRF himoyasi |
| `{% for post in posts %}` | Barcha postlarni aylantirib ko'rsatadi |
| `{% empty %}` | Post yo'q bo'lsa ko'rsatiladi |

---

### 4.4 `create_post.html`

```html
<form method="post">
  {% csrf_token %}
  <h2>Create Post</h2>

  <label>Title:</label>
  <input type="text" name="title" required>

  <label>Content:</label>
  <textarea name="content" required></textarea>

  {% if form.errors %}
    <ul style="color:red;">
      {% for field in form %}
        {% for error in field.errors %}
          <li>{{ field.label }}: {{ error }}</li>
        {% endfor %}
      {% endfor %}
    </ul>
  {% endif %}

  <button type="submit">Create</button>
  <p><a href="{% url 'home' %}">← Back to home</a></p>
</form>
```

| Element | Tushuntirish |
|---------|-------------|
| `name="title"` / `name="content"` | Form field nomlari `PostForm` dagi nomlar bilan mos kelishi shart |
| `{% if form.errors %}` | Validatsiya xatoliklarini ko'rsatadi |

---


## 5. Xavfsizlik mexanizmlari

| Mexanizm | Qayerda | Nimadan himoyalaydi |
|----------|---------|-------------------|
| `{% csrf_token %}` | Har bir POST formada | CSRF (Cross-Site Request Forgery) hujumlaridan |
| `@require_POST` | `logout_view` | GET orqali logout qilinishidan |
| `@login_required` | `create_post` | Ruxsatsiz post yaratishdan |
| `form.is_valid()` | Barcha view-larda | Noto'g'ri/zararlangan ma'lumotlardan |
| Parol hashing | `UserCreationForm.save()` | Parolning ochiq saqlanishidan |
| `cleaned_data` | `create_post` da | XSS va injection hujumlaridan |

---

## 6. Ishga tushirish

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```
