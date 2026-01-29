models.py
from django.db import models
import uuid
from django.utils import timezone

class Paste(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    ttl_seconds = models.IntegerField(null=True, blank=True)
    expires_at = models.DateTimeField(null=True, blank=True)

    max_views = models.IntegerField(null=True, blank=True)
    views = models.IntegerField(default=0)

    def is_expired(self, now):
        if self.expires_at and now >= self.expires_at:
            return True
        if self.max_views is not None and self.views >= self.max_views:
            return True
        return False
        
views.py

import json
import os
from django.http import JsonResponse, HttpResponse
from django.shortcuts import get_object_or_404
from django.utils import timezone
from django.views.decorators.csrf import csrf_exempt
from .models import Paste
from datetime import timedelta
from django.utils.html import escape

def get_now(request):
    if os.getenv("TEST_MODE") == "1":
        ms = request.headers.get("x-test-now-ms")
        if ms:
            return timezone.datetime.fromtimestamp(int(ms) / 1000, tz=timezone.utc)
    return timezone.now()


def healthz(request):
    return JsonResponse({"ok": True})


@csrf_exempt
def create_paste(request):
    if request.method != "POST":
        return JsonResponse({"error": "Method not allowed"}, status=405)

    try:
        data = json.loads(request.body)
    except:
        return JsonResponse({"error": "Invalid JSON"}, status=400)

    content = data.get("content")
    if not content or not isinstance(content, str):
        return JsonResponse({"error": "content required"}, status=400)

    ttl = data.get("ttl_seconds")
    max_views = data.get("max_views")

    paste = Paste(content=content)

    if ttl is not None:
        if not isinstance(ttl, int) or ttl < 1:
            return JsonResponse({"error": "invalid ttl"}, status=400)
        paste.ttl_seconds = ttl
        paste.expires_at = timezone.now() + timedelta(seconds=ttl)

    if max_views is not None:
        if not isinstance(max_views, int) or max_views < 1:
            return JsonResponse({"error": "invalid max_views"}, status=400)
        paste.max_views = max_views

    paste.save()

    return JsonResponse({
        "id": str(paste.id),
        "url": f"/p/{paste.id}"
    }, status=201)


def fetch_paste_api(request, id):
    paste = get_object_or_404(Paste, id=id)
    now = get_now(request)

    if paste.is_expired(now):
        return JsonResponse({"error": "Not found"}, status=404)

    paste.views += 1
    paste.save()

    remaining = None
    if paste.max_views is not None:
        remaining = max(paste.max_views - paste.views, 0)

    return JsonResponse({
        "content": paste.content,
        "remaining_views": remaining,
        "expires_at": paste.expires_at
    })


def view_paste_html(request, id):
    paste = get_object_or_404(Paste, id=id)
    now = get_now(request)

    if paste.is_expired(now):
        return HttpResponse(status=404)

    paste.views += 1
    paste.save()

    safe_content = escape(paste.content)
    return HttpResponse(f"<pre>{safe_content}</pre>")

project url
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('', include('pastes.urls')),
    path('admin/', admin.site.urls),
]
app url

from django.urls import path
from . import views

urlpatterns = [
    path('api/healthz/', views.healthz),
    path('api/pastes/', views.create_paste),
    path('api/pastes/<uuid:id>/', views.fetch_paste_api),
    path('p/<uuid:id>/', views.view_paste_html),
]


