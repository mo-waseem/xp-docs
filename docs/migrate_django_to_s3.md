# 🛠️ Migrating Django Media & Static Files to Amazon S3 — The Complete Guide

Amazon S3 is a highly scalable and production-grade storage solution for Django projects. This guide will help you migrate from local file storage to Amazon S3 for both media and static files, handle path structure cleanly, ensure proper CORS access, and preserve file uniqueness with custom upload handling.

---

## ✅ Step 0: Install AWS CLI (Optional for Syncing Existing Files)

```bash
sudo apt update
sudo apt install awscli -y
```

Or via snap:

```bash
sudo snap install aws-cli --classic
```

---

## ✅ Step 1: Install Required Packages

Install AWS and Django storage libraries:

```bash
pip install boto3 django-storages
```

Then add `'storages'` to your `INSTALLED_APPS` in `settings.py`:

```python
INSTALLED_APPS += ["storages"]
```

---

## ✅ Step 2: Add AWS Configuration to `.env`

Add these values to your `.env` file:

```env
USE_S3=true
AWS_ACCESS_KEY_ID=your-access-key-id
AWS_SECRET_ACCESS_KEY=your-secret-access-key
AWS_STORAGE_BUCKET_NAME=your-bucket-name
AWS_S3_REGION_NAME=eu-west-2
```

---

## ✅ Step 3: Configure Django for S3 Storage (Django 4.2+)

In your `settings.py`:

```python
USE_S3 = env.bool("USE_S3", False)

if USE_S3:
    AWS_S3_CUSTOM_DOMAIN = f"{AWS_STORAGE_BUCKET_NAME}.s3.{AWS_S3_REGION_NAME}.amazonaws.com"

    STORAGES = {
        "default": {
            "BACKEND": "storages.backends.s3boto3.S3Boto3Storage",  # for media files
        },
        "staticfiles": {
            "BACKEND": "core.storage_backends.StaticRootS3Boto3Storage",  # for static files
        },
    }

    MEDIA_URL = f"https://{AWS_S3_CUSTOM_DOMAIN}/media/"
    STATIC_URL = f"https://{AWS_S3_CUSTOM_DOMAIN}/static/"

    AWS_S3_OBJECT_PARAMETERS = {
        "CacheControl": "max-age=86400",
    }
```

Create this file: `core/storage_backends.py`:

```python
from storages.backends.s3boto3 import S3StaticStorage

class StaticRootS3Boto3Storage(S3StaticStorage):
    location = "static"
```

---

## ✅ Step 4: Set a Public Bucket Policy (Optional)

To allow public access to media/static files, go to **S3 > Permissions > CORS configuration** and paste:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedOrigins": ["*"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }
]
```

> Replace `*` in `"AllowedOrigins"` with your actual domain in production.

---

## ✅ Step 5: Use a Custom `upload_to` for FileFields

Create a helper to ensure unique paths:

```python
import os
import uuid
from django.utils import timezone
from django.conf import settings

def unique_upload(instance, filename):
    # Django will handle the uniqueness of the name automatically by
    # the "get_available_name(name, max_length=None)" function that exists in
    # "django.core.files.storage.FileSystemStorage".

    # BUT..., in case of AWS S3 (especially with Bucket Owner Enforced and no ACLs),
    # files are often overwritten if the same name is used — so we enforce uniqueness manually.

    if getattr(settings, "USE_S3", False):
        model_name = instance.__class__.__name__.lower()
        filename = f"{uuid.uuid4()}_{str(timezone.now().timestamp())}_{filename}"
        filename = os.path.join(f"{model_name}/", filename)

    return filename
```

Use in your model:

```python
class Message(models.Model):
    media_file = models.FileField(upload_to=unique_upload)
```

---

## ✅ Step 6: Test File Uploads to S3

Open Django shell:

```bash
python manage.py shell
```

```python
from apps.rent_flow.models import Message
from django.core.files.uploadedfile import SimpleUploadedFile

msg = Message.objects.create(
    media_file=SimpleUploadedFile("test.png", b"hello", content_type="image/png")
)
print(msg.media_file.url)  # should be an S3 URL with /media/
```

---

## ✅ Step 7: Collect Static Files to S3

Make sure your project has a structure like:

```
your_project/
├── static/
│   └── css/
│       └── style.css
```

And in `settings.py`:

```python
STATICFILES_DIRS = [os.path.join(BASE_DIR, "static")]
STATIC_ROOT = os.path.join(BASE_DIR, "staticfiles")
```

Then run:

```bash
python manage.py collectstatic
```

You’ll see your static files uploaded to `static/` in the S3 bucket.

---

## ✅ Step 8: Fix `FileField` Paths Missing `/media/`

If older uploads are missing the `media/` prefix (e.g., `seth.jpg` instead of `media/seth.jpg`) but already exist in S3, just update the DB paths:

```python
from django.core.management.base import BaseCommand
from your_app.models import YourModel

class Command(BaseCommand):
    help = "Updates FileField names to include the media/ prefix"

    def handle(self, *args, **kwargs):
        updated = 0
        for obj in YourModel.objects.exclude(file='').filter(file__isnull=False):
            if not obj.file.name.startswith("media/"):
                obj.file.name = f"media/{obj.file.name}"
                obj.save(update_fields=["file"])
                updated += 1
                self.stdout.write(f"✔ Updated: {obj.file.name}")
        self.stdout.write(self.style.SUCCESS(f"Total updated: {updated}"))
```

---

## ✅ Step 9: Upload Existing Local Files to S3

If you have existing local media files:

```bash
aws s3 sync media/ s3://your-bucket-name/media/
```

> No need to set ACLs if your bucket has public access and object ownership is enforced.

---

## 🎉 Done!

You now have:

- Media uploads going directly to S3
- Static files being collected to S3 with a clean `static/` prefix
- Unique upload paths to avoid collisions
- Public access and CORS support configured
- Older files corrected to follow the new `media/` path structure
