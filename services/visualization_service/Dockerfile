FROM python:3.11-slim

WORKDIR /app

COPY services/visualization_service/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY services/ services/

EXPOSE 8011

CMD ["uvicorn", "services.visualization_service.app:app", "--host", "0.0.0.0", "--port", "8011"]
