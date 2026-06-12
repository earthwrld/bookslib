FROM python:3.10-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

FROM base AS test
RUN python manage.py test

FROM base AS production
RUN addgroup --system appgroup && adduser --system --group appuser \
    && chown -R appuser:appgroup /app
USER appuser
EXPOSE 8000
CMD ["sh", "-c", "python manage.py makemigrations api && python manage.py migrate && python manage.py runserver 0.0.0.0:8000"]