```markdown
# Kenya Transmission Network - Setup Guide

## Install Required Packages
```sh
pip install django djangorestframework django-cors-headers psycopg2-binary django-filter django-leaflet django-mapstore django-geoserver
gis python-decouple
```

## Django Project Setup
```sh
django-admin startproject energy_network
cd energy_network
python manage.py startapp transmission
```

## settings.py Configuration

### Add required apps
```python
INSTALLED_APPS = [
    'django.contrib.gis',
    'rest_framework',
    'corsheaders',
    'transmission',
]
```

### Database Configuration (PostGIS)
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.contrib.gis.db.backends.postgis',
        'NAME': 'energy_db',
        'USER': 'postgres',
        'PASSWORD': 'yourpassword',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

### CORS Settings
```python
CORS_ALLOWED_ORIGINS = [
    'http://localhost:8000',
    'http://127.0.0.1:8000',
    'http://localhost:5173',
]
```

## Models (transmission/models.py)
```python
from django.contrib.gis.db import models

class TransmissionLine(models.Model):
    name = models.CharField(max_length=255)
    voltage = models.IntegerField(choices=[(132, '132kV'), (220, '220kV'), (400, '400kV'), (500, '500kV HVDC')])
    geom = models.LineStringField()

class CountyBoundary(models.Model):
    name = models.CharField(max_length=255)
    geom = models.MultiPolygonField()

class SubCountyBoundary(models.Model):
    name = models.CharField(max_length=255)
    geom = models.MultiPolygonField()
```

## Serializers (transmission/serializers.py)
```python
from rest_framework_gis.serializers import GeoFeatureModelSerializer
from .models import TransmissionLine, CountyBoundary, SubCountyBoundary

class TransmissionLineSerializer(GeoFeatureModelSerializer):
    class Meta:
        model = TransmissionLine
        geo_field = 'geom'
        fields = ('name', 'voltage')

class CountySerializer(GeoFeatureModelSerializer):
    class Meta:
        model = CountyBoundary
        geo_field = 'geom'
        fields = ('name',)

class SubCountySerializer(GeoFeatureModelSerializer):
    class Meta:
        model = SubCountyBoundary
        geo_field = 'geom'
        fields = ('name',)
```

## Views (transmission/views.py)
```python
from rest_framework import viewsets
from .models import TransmissionLine, CountyBoundary, SubCountyBoundary
from .serializers import TransmissionLineSerializer, CountySerializer, SubCountySerializer

class TransmissionLineViewSet(viewsets.ModelViewSet):
    queryset = TransmissionLine.objects.all()
    serializer_class = TransmissionLineSerializer

class CountyViewSet(viewsets.ModelViewSet):
    queryset = CountyBoundary.objects.all()
    serializer_class = CountySerializer

class SubCountyViewSet(viewsets.ModelViewSet):
    queryset = SubCountyBoundary.objects.all()
    serializer_class = SubCountySerializer
```

## URLs (transmission/urls.py)
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import TransmissionLineViewSet, CountyViewSet, SubCountyViewSet

router = DefaultRouter()
router.register(r'transmission-lines', TransmissionLineViewSet)
router.register(r'counties', CountyViewSet)
router.register(r'subcounties', SubCountyViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]
```

## Add URLs to Main URLs (energy_network/urls.py)
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('transmission.urls')),
]
```

## Migrate and Load Data
```sh
python manage.py makemigrations transmission
python manage.py migrate
```

### Load GIS Data
```python
from transmission.models import TransmissionLine
from django.contrib.gis.geos import GEOSGeometry

line = TransmissionLine.objects.create(name='Line A', voltage=132, geom=GEOSGeometry('LINESTRING (36.8219 -1.2921, 37.0 -1.3)'))
```

## Start the Server
```sh
python manage.py runserver
```

## Frontend (MapLibre Visualization)
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Kenya Transmission Network</title>
    <link rel="stylesheet" href="https://unpkg.com/maplibre-gl/dist/maplibre-gl.css">
    <style>
        body { margin: 0; padding: 0; }
        #map { width: 100vw; height: 100vh; }
        .legend, .filter {
            background: white;
            padding: 10px;
            position: absolute;
            bottom: 30px;
            left: 10px;
            border-radius: 5px;
            font-size: 12px;
        }
        .filter { top: 10px; left: 10px; }
    </style>
</head>
<body>
    <div id="map"></div>
    <div class="legend">
        <strong>Voltage Levels</strong><br>
        <span style="color: #ffcc00;">■</span> 132kV<br>
        <span style="color: #ff0000;">■</span> 220kV<br>
        <span style="color: #0000ff;">■</span> 400kV<br>
        <span style="color: #00ff00;">■</span> 500kV HVDC
    </div>
    <div class="filter">
        <label><input type="checkbox" value="132" checked> 132kV</label><br>
        <label><input type="checkbox" value="220" checked> 220kV</label><br>
        <label><input type="checkbox" value="400" checked> 400kV</label><br>
        <label><input type="checkbox" value="500" checked> 500kV HVDC</label>
    </div>
    <script src="https://unpkg.com/maplibre-gl/dist/maplibre-gl.js"></script>
    <script>
        const map = new maplibregl.Map({
            container: 'map',
            style: 'https://demotiles.maplibre.org/style.json',
            center: [37.0, -1.3],
            zoom: 6
        });

        function updateLayers() {
            let selectedVoltages = Array.from(document.querySelectorAll('.filter input:checked')).map(i => parseInt(i.value));
            map.setFilter('lines', ['in', ['get', 'voltage'], ...selectedVoltages]);
        }

        document.querySelectorAll('.filter input').forEach(input => input.addEventListener('change', updateLayers));
    </script>
</body>
</html>
```
```
