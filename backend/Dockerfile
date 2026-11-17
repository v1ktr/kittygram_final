FROM python:3.12

WORKDIR /app

RUN pip install gunicorn==25.1.0

COPY requirements.txt .

RUN pip install -r requirements.txt --no-cache-dir

COPY . .

# Статика собирается в образ, а при старте копируется в volume,
# смонтированный в /backend_static. Каталог kittygram/ совпадает
# с префиксом, по которому gateway раздаёт файлы.
CMD ["sh", "-c", "python manage.py collectstatic --noinput && mkdir -p /backend_static/kittygram/static && cp -r /app/collected_static/. /backend_static/kittygram/static/ && exec gunicorn --bind 0.0.0.0:8000 --no-control-socket kittygram_backend.wsgi"]
